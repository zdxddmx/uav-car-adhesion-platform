# 02 系统架构（System Architecture）

> 只描述「当前真实运行」的分层与流程，把【已接入运行】【源码存在但未接入】分开。所有框图按 `Src/main.c` / `bsp/*.c` / `app/*.c` 的真实调用关系生成。

## 1. 分层总览（真实调用链）

当前 `main()` 只初始化 GPIO / TIM2 / TIM5 / USART2 / UART5，然后进入裸机 10 ms 主循环；`flight_comm`、`rc_control`、FreeRTOS、编码器、ROS 都**不在运行路径**上（`Src/main.c` 第 364–399 行）。

```mermaid
flowchart TD
    subgraph INPUT["输入层（运行中）"]
        PS2["PS2 手柄 2.4G"] --> RECV["USB 无线接收器"]
        RECV --> CH559["CH559 USB Host"]
        CH559 --"ASCII 文本 57600 8N1"--> UART5["UART5 PC12/PD2"]
    end

    subgraph PARSE["解析层 bsp/"]
        UART5 --"逐字节中断"--> PS2P["ps2_usart5.c<br/>HAL_UART_RxCpltCallback<br/>ps2_parse_data"]
    end

    subgraph APP["应用层"]
        PS2P --"ps2.frame_id/buttons/right_key"--> MAIN["Src/main.c<br/>ps2_update → ps2_apply_command"]
    end

    subgraph VEH["车辆层 bsp/car.c"]
        MAIN --"car_forward/backward/turn"--> CAR["car.c"]
    end

    subgraph DRV["驱动层 bsp/"]
        CAR --"car_turn"--> STEER["steering.c"]
        CAR --"motor_set_all"--> MOTOR["motor.c"]
        STEER --"servo_set_angle"--> SERVO["servo.c"]
    end

    subgraph HW["硬件层"]
        SERVO --"CCR1/CCR2 + GPIO PC5/PB0"--> TIM5["TIM5 50Hz"]
        MOTOR --"CCR1~4 + IN1/IN2/STBY"--> TIM2["TIM2 20kHz"]
        TIM2 --> TB["TB6612FNG ×2"]
    end

    subgraph OFF["未接入运行路径（源码保留）"]
        FC["app/flight_comm.c (USART2 DMA+CRC8)"]
        RC["app/rc_control.c (弱符号后端)"]
        RTOS["Src/freertos.c (MX_FREERTOS_Init)"]
        ENC["bsp/encoder.c + com/pid.c"]
        ROS["app/ros_comm_task.c (UART4)"]
    end
    FC -. "未调用" .-> MAIN
    RC -. "未调用" .-> MAIN
    RTOS -. "未调用" .-> MAIN
    ENC -. "未启动" .-> MAIN
    ROS -. "未调度" .-> MAIN
```

> 分层规则证据：`TECHNICAL_DEVELOPMENT_DOCUMENT.md` 第 166–177 行 mermaid 分层图；「输入层只生成车辆级命令，只有底层驱动允许操作 TIM CCR 和 GPIO」。

## 2. 上电启动流程（真实顺序）

证据 `Src/main.c` 第 350–398 行 + `bsp/car.c` 第 93–105 行：

```mermaid
sequenceDiagram
    participant M as main()
    participant C as car.c
    participant S as servo.c
    participant MO as motor.c
    participant ST as steering.c

    M->>M: HAL_Init / SystemClock_Config
    M->>M: MX_GPIO_Init
    M->>M: MX_TIM2_Init (电机 PWM 20kHz)
    M->>M: MX_TIM5_Init (舵机 50Hz)
    M->>M: MX_USART2_UART_Init (调试 printf)
    M->>M: MX_UART5_Init (PS2)
    M->>C: car_init()
    C->>S: servo_init()  逐路错峰100ms启动
    C->>MO: motor_init() STBY先低、四路PWM启动、再拉高STBY
    C->>ST: steering_center()
    C->>MO: motor_stop_all()
    M->>C: car_stop()
    M->>ST: steering_center()
    M->>M: uasrt_rx_init() 开 UART5 单字节中断
    M->>M: while(1) 10ms 控制循环
```

## 3. PS2 主循环控制流（含失联保护）

证据 `Src/main.c` 第 388–399 行（主循环）、第 146–174 行（`ps2_update`）、第 176–291 行（`ps2_apply_command`）。

```mermaid
flowchart TD
    LOOP["while(1) 每10ms"] --> UPD["ps2_update()"]
    UPD --> COPY["ps2_copy_line(): 关中断拷贝 ISR 缓冲区"]
    COPY --> PARSE["ps2_parse_data(line)"]
    PARSE --> CHK{"frame_id == 0x01 ?"}
    CHK -- 是 --> REFRESH["buttons/right_key/connected=1<br/>ps2_last_valid_frame_ms=now"]
    CHK -- 否(0x02或坏行) --> NORE["不刷新时间戳"]
    REFRESH --> TMO{"now - last > 500ms ?"}
    NORE --> TMO
    TMO -- 超时 --> FSAFE["car_stop + steering_center + ps2_command_clear"]
    TMO -- 未超时 --> APPLY["ps2_apply_command()"]
    APPLY --> R1{"R1 ?"} -->|是| STOP1["car_stop"]
    APPLY --> START{"START ?"} -->|是| STOP2["car_stop + 按下沿回中一次"]
    APPLY --> L2{"L2 上升沿 ?"} -->|是| GEAR["挡位 30→60→80→30"]
    APPLY --> FACE{"right_key ?"} -->|1/2=Y/A| BACK["car_backward(pwm)"]
    FACE -->|3=X| LEFT["car_turn(+90°, pwm)"]
    FACE -->|4=B| RIGHT["car_turn(-90°, pwm)"]
    APPLY --> L1{"L1 ?"} -->|是| FWD["car_forward(pwm)"]
    APPLY --> LAT{"run_latched==0 ?"} -->|是| STOP3["car_stop"]
    LOOP --> DBG["ps2_debug_update() 每200ms printf"]
    LOOP --> DLY["HAL_Delay(10)"]
```

## 4. 状态机：PS2 运动命令锁存（隐含状态机）

代码用「锁存变量 + 命令枚举」实现，不是显式 switch 状态机。核心状态变量（`Src/main.c` 第 77–92 行）：

| 变量 | 作用 |
|---|---|
| `ps2_run_latched` | 运动锁存（0=停，1=有运动命令挂起） |
| `ps2_motion_command` | `PS2_CAR_STOP/FORWARD/BACKWARD/LEFT/RIGHT` 枚举 |
| `ps2_l2_was_pressed` | L2 边沿去抖（一次按只切一挡） |
| `ps2_start_was_pressed` | START 回中只执行一次 |
| `ps2_speed_gear_index` | 0/1/2 → 30/60/80% |

```mermaid
stateDiagram-v2
    [*] --> STOP: 上电 ps2_command_clear()
    STOP --> FORWARD: L1 按下(锁存)
    STOP --> BACKWARD: A/Y 按下(锁存)
    STOP --> LEFT: X 按下(锁存)
    STOP --> RIGHT: B 按下(锁存)
    FORWARD --> STOP: R1 或 START 或失联500ms
    BACKWARD --> STOP: R1 或 START 或失联500ms
    LEFT --> STOP: R1 或 START 或失联500ms
    RIGHT --> STOP: R1 或 START 或失联500ms
    STOP --> STOP: run_latched=0 每周期 car_stop
```

## 5. 数据流：PS2 报告 → 车辆命令

证据 `bsp/ps2_usart5.c` 第 82–147 行 + `Src/main.c` 第 156–163 行：

```mermaid
flowchart LR
    UART5["UART5 字节流"] --> CB["HAL_UART_RxCpltCallback<br/>攒 uart5_rx_buf[128]"]
    CB --> FIN{"'\\n' 或 cnt>=64"}
    FIN -->|是| FLAG["uart5_rx_finish=1"]
    FLAG --> COPY["main 主循环 ps2_copy_line<br/>__disable_irq 原子拷贝"]
    COPY --> SSCANF["sscanf 8×x%02X"]
    SSCANF --> FIELDS["frame_id / lx / ly / buttons / right_key"]
    FIELDS --> CMD["ps2_apply_command → car_* API"]
```

## 6. 飞控侧（PX4 覆盖层）架构

当前**未与车控联通**，但与车控是同一系统目标。证据 `adhesion/README.md`：

```mermaid
flowchart TD
    subgraph PX4["PX4 覆盖层（源码已收录）"]
        HC["height_commander<br/>状态机 IDLE→ASCENDING→HOLDING_HIGH→DESCENDING"]
        MLB["mylink_bridge<br/>ASCII TAKEOFF/T/LAND/STOP"]
        HC --"uORB publish"--> OFF["offboard_control_mode / trajectory_setpoint"]
        MLB --"uORB publish"--> AM["actuator_motors (direct_actuator)"]
    end
    subgraph BOARDS["板级配置 default.px4board"]
        B1["micoair h743-v2: 只编入 height_commander"]
        B2["px4 fmu-v6x: 只编入 mylink_bridge"]
        B3["px4 sitl: 两者"]
    end
    HC --> B1
    MLB --> B2
    HC --> B3
    MLB --> B3
    PX4 -. "11字节CRC8协议（STM32端已实现，未启用）" .-> CAR["STM32 车控"]
```

## 7. 资源占用总表（面试速查）

| 资源 | 用途 | 引脚 | 状态 |
|---|---|---|---|
| TIM2 CH1~4 | 电机 PWM | PA15/PB3/PB10/PB11 | 运行 |
| TIM5 CH1/CH2 | 舵机硬件 PWM | PA0/PA1 | 运行 |
| TIM5 CH3/CH4 | 舵机软件 PWM（比较中断） | PC5/PB0 | 运行 |
| TIM1/3/4/8 | 编码器 | PA8/9, PB4/5, PB6/7, PC6/7 | 预留 |
| TIM6 | 编码器采样 20ms | — | 未启动 |
| USART2 | printf 调试 + 飞控协议 | PA2/PA3 | 运行（仅 printf） |
| UART4 | ROS 通信 | PC10/PC11 | 未初始化 |
| UART5 | PS2 | PC12/PD2 | 运行 |
| DMA1_Ch6/7 | USART2 RX/TX | — | 配置，RX 未用 |
| DMA2_Ch3 | UART4 RX | — | 配置，未用 |
| I2C（软件模拟） | IMU（0x50） | PB8/PB9 | 未初始化 |
| GPIO PC15/PB14 | TB6612 STBY | — | 运行 |
