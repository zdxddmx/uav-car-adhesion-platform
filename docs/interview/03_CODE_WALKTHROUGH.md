# 03 代码走读（Code Walkthrough）

> 选取 15 个最值得现场讲的文件，按「作用 / 核心函数 / 为什么这样设计 / 改进空间」展开；末尾给出「面试官让现场打开 GitHub 时的点击顺序」。

---

## 1. `Src/main.c` —— 上电顺序 + PS2 主循环（必讲第一文件）

- **作用**：整个裸机程序的入口与 10 ms 调度核心。
- **核心函数/变量**：
  - `main()`（340–401 行）：`HAL_Init → SystemClock_Config → MX_GPIO/TIM2/TIM5/USART2/UART5_Init → car_init → car_stop → steering_center → uasrt_rx_init → while(1){ ps2_update; ps2_apply_command; ps2_debug_update; HAL_Delay(10) }`。
  - `ps2_copy_line()`（117–144 行）：`__disable_irq()` 关中断，把 ISR 写满的 `uart5_rx_buf` 原子拷到主循环 `ps2_line`，再清 `uart5_rx_finish`、开中断。
  - `ps2_update()`（146–174 行）：解析合法 `0x01` 帧刷新 `ps2_last_valid_frame_ms`；超 500 ms 调 `car_stop()+steering_center()+ps2_command_clear()`。
  - `ps2_apply_command()`（176–291 行）：R1 最高优先停；START 停+按下沿回中一次；L2 上升沿切挡；face 键 Y/A=后退、X=左转+90、B=右转-90；L1=直行。
  - 宏（59–64 行）：`PS2_CONTROL_PERIOD_MS=10`、`PS2_DEBUG_PERIOD_MS=200`、`PS2_CONNECTION_TIMEOUT_MS=500`、`PS2_MAX_STEERING_ANGLE_DEG=90`；挡位表 `ps2_speed_percent[]={30,60,80}`（87–92 行）。
- **为什么这样设计**：把「解析」和「执行」分开——ISR 只攒字节，主循环做 `sscanf` 解析和状态决策，避免在中断里跑 printf/浮点/长逻辑。
- **改进空间**：`HAL_Delay(10)` 是阻塞延迟，会引入 ±10 ms 抖动；锁存式 L1 不如 deadman 安全（`TECHNICAL_DEVELOPMENT_DOCUMENT.md` 290–292 行已自评）。

## 2. `bsp/servo.c` —— 四舵机标定 + 硬/软 PWM（技术亮点）

- **作用**：4 路舵机的标定、限幅、PWM 生成。
- **核心函数**：
  - 标定表 `servo_config[4]`（22–28 行）：`{1500,1300,1700,1}` 即 `center_us/min_us/max_us/direction`。
  - `servo_clamp_us()`（47–61 行）：硬限幅到 min/max。
  - `servo_init()`（157–195 行）：载入中位 → GPIO → TIM5 通道配置 → NVIC 优先级 5 → 使能 UPDATE/CC3/CC4 中断 → 逐路 `HAL_TIM_PWM_Start` 错开 100 ms。
  - `servo_set_angle()`（262–299 行）：先钳到 ±90°，乘 `direction`，线性映射到 `center±(min/max-center)`，`+0.5F` 四舍五入后 `servo_set_us()`。
  - `servo_tim5_irq_handler()`（332–376 行）：UPDATE 时置 PC5/PB0 高并把 `servo_pending_us` 写入 CCR3/CCR4；CC3/CC4 比较中断分别置 PC5/PB0 低。
- **为什么这样设计**：TIM5 的 CH3/CH4 引脚 PA2/PA3 被 USART2 占用，所以 CH3/CH4 只做「比较时刻」，用 GPIO 翻转实现软件 PWM。
- **改进空间**：软件 PWM 抖动未量化（`SERVO_CALIBRATION.md` 自评）；`servo_set_all()`/`steering_set_four_angles()` 内部 `HAL_Delay(100)` 会阻塞主循环。

## 3. `bsp/motor.c` —— 四电机 PWM + TB6612 方向

- **作用**：电机 PWM、方向脚、STBY、限幅、车辆坐标语义。
- **核心函数**：
  - `motor_config[4]`（30–36 行）：`{1,80},{1,80},{1,80},{-1,80}`，`-1` 是 J4 反装修正。
  - `motor_hardware[4]`（38–44 行）：TIM 通道 + IN1/IN2 端口脚映射。
  - `motor_apply_output()`（63–91 行）：先清 CCR → 按符号设 IN1/IN2 → 写占空比；停止态 IN1=IN2=0。
  - `motor_set_pwm()`（182–214 行）：`signed_pwm = pwm * direction`，取幅值，钳到 `pwm_max`，再应用。
  - `motor_init()`（146–180 行）：STBY 先低，四路 PWM 启动，把 `pwm_max` 强制钳到 80%，再拉高 STBY。
  - `motor_percent_to_pwm()`（133–144 行）：`full_scale = ARR+1 = 100`，`percent/100*100`。
- **为什么这样设计**：`direction` 统一在底层处理镜像安装，上层（car/PS2）不针对单轮打正负号补丁（`J4_DIRECTION_FIX.md` 第 17 行）。
- **改进空间**：`motor_ctrol()` 里的 `chassis->motor[0..3] → MOTOR_3/1/4/2` 映射是历史差速底盘的遗留，与 `car.c` 的对称直驱语义不一致，属易混淆点。

## 4. `bsp/steering.c` —— 转向几何（普通 vs 蟹行）

- **作用**：把「车辆转向角」翻译成四个舵机角。
- **核心函数**：
  - `steering_turn(angle)`（78–87 行）：前轴 `+angle`、后轴 `-angle`（普通四轮转向）。
  - `steering_crab(angle)`（72–76 行）：四轮同角（蟹行，当前 PS2 版本不启用）。
  - `steering_set_four_angles()`（25–40 行）：逐舵机 100 ms 错峰。
  - `steering_center()` → `servo_center_all()`。
- **为什么这样设计**：`steering_clamp_angle()` 与 `servo_set_angle()` 双重 ±90° 钳位，保证命令角与 PWM 范围都安全。
- **改进空间**：`steering_crab` 保留但无机械 ±90° 验证（`README.md` 第 52 行）；真蟹行需要独立校验四轮同角下无连杆干涉。

## 5. `bsp/car.c` —— 车辆级 API（分层核心）

- **作用**：屏蔽「转向 + 电机」两层的组合细节，暴露 `car_forward/backward/turn/crab/stop/init`。
- **核心函数**：
  - `car_init()`（93–105 行）：`servo_init→motor_init→steering_center→motor_stop_all`。
  - `car_set_steering()`（69–91 行）：**缓存**最近转向模式+角度，`CAR_STEERING_CHANGE_EPSILON=0.01` 判断是否变化，避免每 10 ms 重复执行带 100 ms 错峰的舵机更新。
  - `car_forward/backward/turn/crab`（114–143 行）。
- **为什么这样设计**：转向动作贵（每次 4×100 ms 错峰阻塞），所以只在「模式或角度变化」时才更新；电机速度每次刷新。
- **改进空间**：`car_angle_changed` 用 `fabs` 手写，可换 `fabsf`；缓存是「避免重算」的典型做法，但需要面试时能讲清「为何缓存、何时失效」。

## 6. `bsp/ps2_usart5.c` + `bsp/ps2_usart.h` —— PS2 文本解析

- **作用**：UART5 单字节中断接收 + `HUB0_Joystick data:` 行解析。
- **核心函数**：
  - `uasrt_rx_init()`（24–27 行）：`HAL_UART_Receive_IT(&huart5,&rx_data,1)`。
  - `HAL_UART_RxCpltCallback()`（39–60 行）：攒 `uart5_rx_buf`，遇 `'\n'` 或 64 字节封帧，`uart5_rx_finish=1`。
  - `ps2_parse_data()`（83–196 行）：`sscanf` 8 个 `x%02X`；`frame_id=x1`、`lx=map(x4)`、`ly=map(x5)`、`buttons=x7`、face 键从 `x6`。
  - `ps2_usart.h`：`PS2_BUTTON_L1..R3` 位掩码（20–27 行）、`KEY_TRIANGLE=0x1F / CROSS=0x4F / SQUARE=0x8F / CIRCLE=0x2F`（14–17 行）。
- **为什么这样设计**：CH559 输出的是固定 ASCII 行，用 `sscanf` 一次解析 8 字节最直观；只接受 `res==8` 才更新结构体，防半帧污染。
- **改进空间**：`ps2_parse_data` 里仍有旧 `remote_stopped_flag`（按「非零活动」100 ms 计时，176–185 行）——`CAR_CONTROL.md` 已警告该逻辑不能用于 500 ms 心跳，属「遗留未删」的死逻辑，面试要能区分「当前用的 500 ms」与「旧 100 ms」。

## 7. `app/flight_comm.c` + `.h` —— 11 字节 CRC8 飞控协议（未启用但完整）

- **作用**：USART2 DMA 环形接收 + 11 字节帧解析 + CRC8 + 500 ms 超时。
- **核心函数**：
  - `flight_crc8()`（34–57 行）：CRC-8/ATM，poly 0x07。
  - `flight_validate_and_apply_frame()`（105–144 行）：校验帧头/CRC/version/mode 范围，合法回 `OK\r\n`，CRC 错回 `CRC_ERR\r\n`。
  - `flight_parser_push()` + `flight_parser_resync()`（179–221、146–177 行）：失步后重新搜 `AA 55`，单字节丢包不会永久错位。
  - `flight_process_dma_bytes()`（223–249 行）：读 DMA1_Channel6 剩余计数，消费新字节。
  - `rc_backend_read()`（309–340 行）：`FLIGHT_COMM_ENABLE_CAR_OUTPUT=0` 时只输出全零命令。
- **为什么这样设计**：`FLIGHT_COMM_ENABLE_CAR_OUTPUT` 作为「阶段一安全门」，协议可调试但电机不动作（`FLIGHT_UART.md` 第 14–16 行）。
- **改进空间**：USART2 同时被 `printf` 占用，接入真飞控前必须迁移日志口（`FLIGHT_UART.md` 第 40 行）。

## 8. `app/rc_control.c` + `.h` —— 遥控抽象层（弱符号后端）

- **作用**：协议无关的统一 `RcCommand` + 失联保护。
- **核心函数**：
  - `RcCommand` 结构（`rc_control.h` 6–13 行）：`throttle/steering/mode/enable/connected`。
  - `__weak rc_backend_read()`（38–42 行）：默认返回 0（无后端）。
  - `rc_update()`（58–103 行）：读一帧 → 限幅 ±1000 → 归一化 mode/enable → 500 ms 超时进 `rc_enter_failsafe()`。
- **为什么这样设计**：弱符号让「换后端不改 car」成为可能；只有 `flight_comm.c` 提供一个强定义。
- **改进空间**：当前 `main()` **没接入** `rc_update()`（`CODE_MAP.md` 第 10 行），所以这层是「已实现待接线」。

## 9. `Src/tim.c` —— 定时器配置（寄存器级参数集中地）

- **作用**：TIM1~8 的 HAL 配置，含编码器与 PWM。
- **关键点**：
  - TIM2 `PSC=35, ARR=99`（95–100 行）→ 20 kHz；TIM5 `PSC=71, ARR=19999`（249–253 行）→ 50 Hz。
  - TIM1/3/4/8 全部 `TIM_ENCODERMODE_TI12`、`Period=65535`（如 43–64 行）。
  - TIM6 `PSC=7199, ARR=199`（302–305 行）→ 20 ms。
  - `HAL_TIM_MspPostInit()`（504–558 行）：TIM2 用 `__HAL_AFIO_REMAP_TIM2_ENABLE()` 把 CH1/CH2/CH3/CH4 映射到 PA15/PB3/PB10/PB11；TIM3 用 `__HAL_AFIO_REMAP_TIM3_PARTIAL()` 映射到 PB4/PB5。
- **为什么这样设计**：PA15/PB3 是 JTAG 脚，必须 remap 才能当 PWM 用——这是 F103 的一个经典坑，面试好讲。
- **改进空间**：TIM1/3/4/8 的编码器 MspInit 在 `MX_TIM*_Init` 里会被调用，但 `main()` 不调这些 Init 函数，所以编码器 GPIO 实际**未**被配置为输入——可讲「配置函数在、调用缺失」。

## 10. `Src/usart.c` —— 串口/DMA 引脚与参数

- **作用**：USART2/UART4/UART5 的初始化与 DMA 挂接。
- **关键点**：
  - USART2：PA2/PA3、115200、DMA1_Ch6(RX,Circular,High)/Ch7(TX,Normal,Medium)（227–256 行）。
  - UART5：PC12/PD2、57600、无 DMA（73–89 行）。
  - UART4：PC10/PC11、115200、DMA2_Ch3 RX Circular（147–162 行）。
- **追问点**：为什么 UART5 没用 DMA？→ 文本行短、逐字节中断 + 主循环 `sscanf` 已够，`ps2_usart5.c` 第 1 行注释「后续可扩展 dma+空闲中断」。

## 11. `Src/stm32f1xx_it.c` —— 中断入口

- **作用**：把外设中断接到 HAL 处理函数；`TIM5_IRQHandler()` 直接调 `servo_tim5_irq_handler()`（295–298 行）。
- **关键点**：`SysTick_Handler()` 同时调 `HAL_IncTick()` 和 `xPortSysTickHandler()`（166–183 行）——这是 FreeRTOS 兼容写法，但 FreeRTOS 未启动，`xPortSysTickHandler` 在调度器未启动时是安全的空操作。

## 12. `Src/gpio.c` —— GPIO 初始电平

- **作用**：上电把 TB6612 方向脚/STBY、PC13/14/15、PB12/13/14 等全部拉低，确保 STBY 默认禁止（`MX_GPIO_Init` 第 53–59 行）。
- **追问点**：为什么上电 STBY 要低？→ 防止上电瞬间电机误动，`motor_init()` 也再次确认 STBY 低（`MOTOR_TEST.md` 第 4 节）。

## 13. `com/pid.c` + `com/pid.h` —— PID（未启用）

- **作用**：位置式 `pid_calc` + 增量式 `pid_calc_inc`。
- **关键点**：`pid_calc` 的积分只在「输出未饱和」时累加（`pid.c` 第 42–44 行 else 分支），即抗积分饱和；`pid_calc_inc` 有 `PID_INC_VEL_DEADBAND_MS=0.04` 死区（`pid.h`）。
- **改进空间**：`pid_calc` 的积分项没有独立限幅，仅靠输出饱和间接抑制；增量式步长硬编码 ±3.0。

## 14. `app/chassis_task.c` + `bsp/encoder.c` —— 差速/麦克纳姆底盘遗留（未启用）

- **作用**：旧版 FreeRTOS 底盘任务、麦克纳姆分解、编码器速度环。
- **关键点**：`decompose_and_normalize()` 是麦克纳姆轮速度分解（`chassis_task.c` 第 183–214 行）；`motor_pid_init` Kp=282/Ki=85/Kd=0/max_out=100（第 71–77 行）；`encoder.c` 的 `HAL_TIM_PeriodElapsedCallback`（TIM6 20 ms）计算四轮速度并调 `pid_calc_inc`。
- **为什么讲**：展示「我读过并理解旧代码，且清楚它没被当前运行路径使用」——这是审计能力的体现。

## 15. `adhesion/px4-overlay/.../HeightCommander.cpp` + `MylinkBridge.cpp` —— 飞控自定义模块

- **HeightCommander**：状态机 `IDLE→ASCENDING→HOLDING_HIGH→DESCENDING→IDLE`（`updateStateMachine()` 177–207 行）；`getAuxChannelValue()` 把通道 6 和 7 都映射为 `aux2`（69–77 行，实机前需修正）；NED 坐标「上升 = z 更负」。
- **MylinkBridge**：ASCII `TAKEOFF/T/LAND/STOP`；`publishActuatorMotors()` 给 M1~M4 写相同推力（144–158 行）——**直接执行器输出，绕过姿态闭环，禁止实飞**（`MylinkBridge.cpp` 无 CRC/认证/超时）。
- **为什么讲**：诚实暴露高风险缺口是加分项，说明候选人知道「实验代码 ≠ 可飞行代码」。

---

## 16. GitHub 现场演示点击顺序（完整导览路线）

> 面试官让「打开 GitHub 现场走一遍」时，按这个顺序，30 秒内点到关键行：

1. **根 `README.md`** → 滚到「当前小车运行版本」表格（40–52 行）：讲「上电 car_init→car_stop→steering_center，PS2 锁存控制，500 ms 失联，±90° 只是软件命令上限」。
2. **`car/firmware/stm32f103-car-controller/Src/main.c`** → 滚到 `main()`（340–401 行）：讲上电顺序和 `while(1)` 三件套 `ps2_update/ps2_apply_command/ps2_debug_update`。
3. **同文件** 滚到 `ps2_apply_command()`（176–291 行）：讲 R1 最高优先、L2 边沿切挡、face 键映射。
4. **`bsp/servo.c`** → 滚到 `servo_config` 表（22–28 行）与 `servo_tim5_irq_handler()`（332–376 行）：讲「两硬两软」PWM。
5. **`bsp/motor.c`** → 滚到 `motor_config` 表（30–36 行）与 `motor_apply_output()`（63–91 行）：讲 J4 `direction=-1` 和「先清 PWM 再换向」。
6. **`bsp/car.c`** → 滚到 `car_set_steering()`（69–91 行）：讲「转向缓存避免每 10 ms 重复错峰」。
7. **`app/flight_comm.c`** → 滚到 `flight_crc8()`（34–57 行）和 `rc_backend_read()`（309–340 行）：讲「协议已实现但 `FLIGHT_COMM_ENABLE_CAR_OUTPUT=0` 未启用」。
8. **`adhesion/px4-overlay/src/modules/height_commander/HeightCommander.cpp`** → 滚到 `updateStateMachine()`（131–215 行）：讲 RC 边沿触发相对高度状态机。
9. **`adhesion/px4-overlay/src/modules/mylink_bridge/MylinkBridge.cpp`** → 滚到 `publishActuatorMotors()`（144–158 行）：主动指出「直接执行器输出，禁止实飞」。
10. **`docs/PROJECT_STATUS.md`** → 全文：用「已实测 / 已实现待实测 / 未完成」三栏收尾，展示证据边界意识。
