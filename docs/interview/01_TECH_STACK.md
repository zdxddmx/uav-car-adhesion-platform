# 01 技术栈（Tech Stack）

> 每个技术栈按「是什么 → 为什么用 → 怎么用 → 代码在哪里 → 面试官可能怎么追问」展开。术语保留英文，证据精确到文件/函数/宏/寄存器参数。

---

## 1. STM32F103RCT6（MCU）

- **是什么**：ST 的 Cortex-M3 主流单片机，F103 高性能线（XL-density）成员，RCT6 = 256 KB Flash / 48 KB RAM / 64 脚 LQFP。
- **为什么用**：实习项目已有 `WLKJ2025011 CAR-MOTOR-V1.2S` 控制板（`car/README.md` 第 5 行），板载两颗 TB6612、CH559、7.4 V 舵机电源域，MCU 是既有 BOM 选择；4 个 TIM 编码器 + 2 路高级/通用定时器足够四电机四舵机。
- **怎么用 / 代码在哪里**：
  - 器件确认：`1_template_led.ioc` 第 42 行 `Mcu.CPN=STM32F103RCT6`、第 107 行 `Mcu.UserName=STM32F103RCTx`；`MDK-ARM/1_template_led.uvprojx` `<Device>STM32F103RC</Device>`，`IROM(0x8000000-0x803FFFF)`=256 KB、`IRAM(0x20000000-0x2000BFFF)`=48 KB。
  - 启动文件：`MDK-ARM/startup_stm32f103xe.s`（对应 XL-density 向量表）。
- **追问点**：RCT6 的 Flash/RAM 多大？→ 256 KB / 48 KB；为什么中断向量表用的 `xe` 启动文件？→ `RCT` 属于 XL-density（256–512 KB），`xe` 是匹配启动文件命名。

## 2. 时钟树 / RCC / HSE

- **是什么**：系统时钟与总线时钟分频，决定所有定时器/PWM 的计数频率。
- **为什么用 HSE+PLL**：72 MHz 是 F103 最高 SYSCLK，TIM 需要整数 μs 级计数。
- **怎么用 / 代码在哪里**：
  - `Src/main.c` `SystemClock_Config()`（第 407–440 行）：`HSE ON`、`PLL source=HSE`、`PLLMUL=RCC_PLL_MUL9`、`APB1=DIV2`、`APB2=DIV1`、`FLASH_LATENCY_2`。
  - `.ioc` 关键值：`RCC.HSE_VALUE=16000000`、`RCC.HSEDivPLL=RCC_HSE_PREDIV_DIV2`、`RCC.PLLMUL=RCC_PLL_MUL9`、`RCC.AHBFreq_Value=72000000`、`RCC.APB1Freq_Value=36000000`、`RCC.APB1TimFreq_Value=72000000`、`RCC.APB2TimFreq_Value=72000000`。
  - 推导链：**HSE 16 MHz ÷2（PLLXTPRE）→ 8 MHz → PLL ×9 → 72 MHz SYSCLK**；APB1=36 MHz（APB1 预分频≠1 时定时器时钟 ×2=72 MHz），APB2=72 MHz。
- **追问点**：为什么 APB1 是 36 MHz 但 TIM2/TIM5 计数按 72 MHz？→ F103 规定 APB1 预分频系数≠1 时，挂在 APB1 的 TIM 时钟自动 ×2（`.ioc` 里 `APB1TimFreq_Value=72000000` 就是证据）。HSE 到底多少？→ 16 MHz（`.ioc` `HSE_VALUE`），不是 Keil 工程模板里的默认 8 MHz。

## 3. TIM 定时器（PWM / Encoder）

- **是什么**：通用/高级定时器，本仓库承担电机 PWM、舵机 PWM、编码器计数、周期中断四种角色。
- **代码在哪里 / 参数**：

| 定时器 | 角色 | PSC / ARR | 结果 | 证据 |
|---|---|---|---|---|
| TIM2 | 4 路电机 PWM | 35 / 99 | 20 kHz，100 计数满量程 | `Src/tim.c` 第 95–100 行 |
| TIM5 | 4 路舵机 PWM | 71 / 19999 | 50 Hz，1 μs/计数 | `Src/tim.c` 第 249–253 行 |
| TIM1/3/4/8 | 编码器（预留） | 0 / 65535，`TIM_ENCODERMODE_TI12` | 四倍频计数 | `Src/tim.c` 各 `MX_TIM*_Init` |
| TIM6 | 编码器采样周期中断 | 7199 / 199 | 20 ms（50 Hz） | `Src/tim.c` 第 302–305 行 |

- **追问点**：TIM2 20 kHz 怎么算？→ `72e6/(35+1)/(99+1)=20 kHz`（`MOTOR_TEST.md` 第 25–31 行）。TIM5 为什么 `PSC=71, ARR=19999`？→ `72e6/72/20000=50 Hz`，计数单位恰好 1 μs，直接拿 μs 写 CCR。

## 4. 硬件 PWM vs 软件 PWM（舵机）

- **是什么**：舵机要 4 路 50 Hz 脉宽信号，但 TIM5 只有 4 个比较通道、其中 CH1/CH2 有物理输出脚可用（PA0/PA1），CH3/CH4 引脚被占（PA2 是 USART2_TX、PA3 是 USART2_RX），所以 CH3/CH4 只当「内部比较时刻」，用 GPIO 手动翻转。
- **为什么这样**：资源受限下的折中；PA2/PA3 已用于飞控 USART2，PB0/PC5 是剩余可用脚。
- **代码在哪里**：`bsp/servo.c`：
  - `servo_gpio_init()` 把 PC5/PB0 配成输出（第 100–110 行）；
  - `servo_tim5_channels_init()` 把 CH3/CH4 配成 `TIM_OCMODE_TIMING`（只比较不输出，第 126–135 行）；
  - `servo_tim5_irq_handler()`：UPDATE 中断把 PC5/PB0 拉高并更新 CCR3/CCR4，CC3/CC4 比较中断分别把 PC5/PB0 拉低（第 332–375 行）。
- **追问点**：软件 PWM 有什么风险？→ 脉宽抖动取决于中断响应延迟，`SERVO_CALIBRATION.md` 明确「软件 PWM 尚未用示波器做定量抖动测量」。

## 5. TB6612FNG（电机驱动）

- **是什么**：双路 H 桥直流电机驱动，每片两通道，支持 PWM 调速 + IN1/IN2 方向 + STBY 使能。
- **为什么用**：板载两颗 = 4 通道，正好带 4 个 JGA25 减速电机；STBY 脚可做硬件急停/上电锁定。
- **代码在哪里**：`bsp/motor.c`：
  - 硬件映射表 `motor_hardware[4]`（第 38–44 行）：每路记录 TIM 通道 + IN1/IN2 端口脚；
  - `motor_set_standby()` 同时控制 `PC15`（TB6612_1_STBY）与 `PB14`（TB6612_2_STBY）（第 56–61 行）；
  - `motor_apply_output()` 先清 CCR 再改方向脚、最后写占空比，停止态 `IN1=IN2=0`（滑行停止）（第 63–91 行）。
- **追问点**：为什么换向前先清 PWM？→ 避免带占空比直接翻转 H 桥方向造成冲击/直通风险（`motor.c` 第 68 行注释 + `MOTOR_TEST.md` 第 42 行）。停止是刹车还是滑行？→ 滑行（IN1=IN2=0，不是短路刹车 11）。

## 6. PS2 手柄 + CH559 + UART5

- **是什么**：Twin USB Joystick 2.4G 无线手柄 → USB 无线接收器 → 板载 USB-A → CH559（USB Host 单片机）→ UART5（TTL 文本流）→ STM32。
- **为什么用**：本地遥控调试需求；CH559 把 USB HID 转成一行 ASCII，STM32 不用自己实现 USB Host。
- **代码在哪里**：`bsp/ps2_usart5.c` / `bsp/ps2_usart.h`：
  - `uasrt_rx_init()` → `HAL_UART_Receive_IT(&huart5,&rx_data,1)` 逐字节中断；
  - `HAL_UART_RxCpltCallback()` 攒到 `uart5_rx_buf`，遇 `'\n'` 或 64 字节置 `uart5_rx_finish`；
  - `ps2_parse_data()` 用 `sscanf("HUB0_Joystick data: x%02X ...", 8 个)` 解析 8 字节报告。
  - 引脚：UART5 TX=PC12、RX=PD2，57600 8N1（`Src/usart.c` 第 73–79 行、185–193 行）。
- **追问点**：这是 STM32 直读 PS2 的 SPI 驱动吗？→ 不是，是外部 CH559 桥接器的**文本行解析器**（`CAR_CONTROL.md` 第 127–129 行原话强调）。

## 7. PX4 / uORB / MAVLink

- **是什么**：PX4 是开源无人机飞控固件（NuttX RTOS + uORB 发布/订阅总线 + 模块化飞行控制）。
- **为什么用**：无人机侧需要飞控能力，选择 PX4 覆盖层而不是自研整机飞控。
- **代码在哪里**：
  - `height_commander`：订阅 `manual_control_setpoint`、`vehicle_status`、`vehicle_local_position`，发布 `offboard_control_mode` / `trajectory_setpoint` / `vehicle_command`（`HeightCommander.cpp` 第 89–96 行 hpp 定义）。
  - `mylink_bridge`：发布 `offboard_control_mode(direct_actuator)` / `actuator_motors` / `vehicle_command`（`MylinkBridge.cpp` 第 136–157 行）。
  - 覆盖策略：只存项目自定义文件，覆盖到上游 PX4 commit `6388739efb068d72677f6a5777742e30aefa21a6`（`SOURCE_MANIFEST.md` 第 49–53 行）。
- **追问点**：MAVLink 用了没？→ **没实现**。板级配置里有 `CONFIG_MODULES_MAVLINK=y`，但业务代码用的是自定义 uORB 发布/订阅 + 自定义文本协议，没有 MAVLink 消息解析；`FLIGHT_UART.md` 第 182–184 行把 MAVLink 列为「后续升级」。

## 8. CRC8 协议（自定义 11 字节车辆协议）

- **是什么**：`0xAA 0x55` 帧头 + version + enable + mode + throttle(i16 大端) + steering(i16 大端) + sequence + CRC8，共 11 字节。
- **代码在哪里**：`app/flight_comm.c`：
  - `flight_crc8()`：CRC-8/ATM，poly `0x07`，init `0x00`，RefIn/RefOut=False，XorOut=`0x00`（第 34–57 行）；
  - `flight_validate_and_apply_frame()`：校验帧头/CRC/version/mode 范围（第 105–144 行）；
  - `flight_parser_resync()`：丢字节后重新搜 `AA 55`（第 146–177 行）。
- **为什么没启用**：`flight_comm.h` 第 15 行 `FLIGHT_COMM_ENABLE_CAR_OUTPUT 0`，且 `main()` 不调 `flight_comm_init()`；USART2 被 `printf` 占用做调试。
- **追问点**：为什么 CRC 覆盖 Byte0~Byte9？→ 因为 Byte10 本身就是 CRC，只对前 10 字节计算（`FLIGHT_CRC_DATA_SIZE=10`）。

## 9. Keil MDK / CubeMX / ST-Link

- **是什么**：Keil MDK-ARM（ARMCC 5.06）编译 + ST-Link/SWD 烧录；CubeMX `.ioc` 生成 HAL 骨架。
- **代码在哪里**：`MDK-ARM/1_template_led.uvprojx`（工程入口）、`1_template_led.ioc`（CubeMX 快照，`CODE_MAP.md` 提醒「不要无审查地整工程重新生成」）。
- **构建证据**：`keil_rebuild.log` 第 1、58–60 行：`V5.06 update 5 (build 528)`，`Code=16684 ... 0 Error(s), 0 Warning(s)`。
- **追问点**：为什么 `.ioc` 不能随便重新生成？→ 因为 `servo.c`/`motor.c`/`ps2_usart5.c` 有大量 `USER CODE` 区之外的改动（尤其 TIM5 CH3/CH4 软件 PWM、TIM2 引脚 remap），无审查重新生成会覆盖这些手写逻辑。

## 10. FreeRTOS / ROS / 编码器 PID（「有源码但未启用」三件套）

- **FreeRTOS**：`Middlewares/Third_Party/FreeRTOS` + `Src/freertos.c`（`MX_FREERTOS_Init()` 定义了 defaultTask/chassic_task/remote_task/roscommTask），但 `Src/main.c` **没有调用** `MX_FREERTOS_Init()`（`main.c` 第 364–368 行只 init GPIO/TIM2/TIM5/USART2/UART5）。
- **ROS**：`app/ros_comm_task.c` 用 UART4 DMA+IDLE 做 24 字节帧、XOR 校验、300 ms 超时，但它是 FreeRTOS 任务体，同样未被调度。
- **编码器 PID**：`bsp/encoder.c` + `com/pid.c`（`pid_calc_inc` 增量式）存在，`Src/tim.c` 有 TIM1/3/4/8 编码器初始化函数，但 `main()` 不调用它们，也不启动 TIM6。
- **面试话术**：这三样「工程里带着源码但当前运行固件不启动」，是我主动遵守的边界（`README.md` 第 40 行、`CAR_CONTROL.md` 第 8 行）。
