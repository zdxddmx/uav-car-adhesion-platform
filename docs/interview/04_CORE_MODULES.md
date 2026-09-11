# 04 核心模块（Core Modules）

> 每个模块按「功能 / 接口 / 实现 / 依赖 / 边界」逐个讲。证据精确到函数与关键变量。

---

## 1. servo（舵机模块）— `bsp/servo.c` / `servo.h`

- **功能**：四路位置舵机的标定、角度→PWM 换算、硬/软 PWM 生成、安全限幅。
- **接口**（`servo.h`）：`servo_init / servo_set_us / servo_center / servo_center_all / servo_set_center / servo_set_direction / servo_set_angle / servo_set_all / servo_calibration_test / servo_tim5_irq_handler`。
- **实现要点**：
  - `ServoConfig{center_us,min_us,max_us,direction}`，全局表 `servo_config[4]`（`servo.c` 22–28 行）。
  - `servo_set_us()` 只做 `clamp` + 写 CCR（LF/RF）；LR/RR 由中断消费 `servo_pending_us`。
  - `servo_set_angle()` 线性映射：正角度到 `center~max`，负角度到 `min~center`，`direction` 反号。
- **依赖**：`htim5`（`Src/tim.c`）、GPIOA/B/C、`TIM5_IRQHandler`（`stm32f1xx_it.c`）。
- **边界**：`servo_set_angle` 钳 ±90°（`SERVO_ANGLE_LIMIT_DEG`）；软件 PWM 抖动未量化；标定程序 `servo_calibration_test()` 死循环，当前 `main()` 不调用。

## 2. motor（电机模块）— `bsp/motor.c` / `motor.h`

- **功能**：四路电机 PWM、TB6612 方向/STBY、车辆坐标正负速度、限幅、百分比换算。
- **接口**：`motor_init / motor_set_pwm / motor_forward / motor_reverse / motor_stop / motor_stop_all / motor_set_all / motor_get_pwm_full_scale / motor_percent_to_pwm / motor_single_test / motor_four_test / motor_ctrol`。
- **实现要点**：
  - `MotorConfig{direction,pwm_max}`，`motor_config[4]={{1,80},{1,80},{1,80},{-1,80}}`。
  - `motor_apply_output()`：先清 CCR → 设方向 → 写幅值；停止态 `IN1=IN2=0`。
  - `motor_init()`：STBY 低 → 四路 PWM Start → `pwm_max` 钳到 80% → STBY 高。
- **依赖**：`htim2`、GPIO（PC13/14/15、PC0/1、PC8/9、PB12/13/14）、`chassis_mode.h`（`motor_ctrol` 用）。
- **边界**：80% 上限是「代码强制」非「实测最优」；`motor_ctrol` 是遗留差速底盘兼容入口，与 `car.c` 语义不同。

## 3. steering（转向模块）— `bsp/steering.c` / `steering.h`

- **功能**：车辆转向角 → 四舵机角（普通/蟹行），回中。
- **接口**：`steering_center / steering_crab / steering_turn / steering_test_angle` + 4 个测试函数。
- **实现要点**：`steering_turn` 前轴 `+angle` 后轴 `-angle`；`steering_crab` 四轮同角；`steering_set_four_angles` 每路 100 ms 错峰。
- **依赖**：`servo.h`。
- **边界**：角度钳 ±90°；蟹行能力存在但无机械验证。

## 4. car（车辆模块）— `bsp/car.c` / `car.h`

- **功能**：屏蔽转向+电机组合，暴露车辆级 API，缓存转向状态。
- **接口**：`car_init / car_stop / car_forward / car_backward / car_turn / car_crab`。
- **实现要点**：`car_set_steering()` 用 `car_steering_mode` + `car_steering_angle_deg` 缓存，`CAR_STEERING_CHANGE_EPSILON=0.01` 判断变化。
- **依赖**：`motor.h`、`servo.h`、`steering.h`。
- **边界**：`car_stop()` 只停电机不回中（`car.c` 107–112 行注释明确）；角度钳 ±90°。

## 5. PS2 / CH559 接收模块 — `bsp/ps2_usart5.c` / `ps2_usart.h`

- **功能**：UART5 单字节中断接收 + `HUB0_Joystick data:` 文本行解析。
- **接口**：`uasrt_rx_init / ps2_parse_data`；全局 `ps2`（`PS2_HandleTypeDef`）、`uart5_rx_buf`、`uart5_rx_finish`。
- **实现要点**：`HAL_UART_RxCpltCallback` 攒字节、`'\n'` 封帧；`ps2_parse_data` 用 `sscanf` 解析 8 个 `x%02X`；`buttons`（Byte6）按位掩码。
- **依赖**：`huart5`、`calc_fun.h`（`abs_int`）、`com_debug.h`、`chassis_task.h`（`extern Chassis_TypeDef chassis`，仅编译期引用）。
- **边界**：只认 `frame_id==0x01`；`0x02` 忽略；旧 `remote_stopped_flag` 100 ms 逻辑是死代码，不承担 500 ms 心跳。

## 6. car control（当前 PS2 控制逻辑）— `Src/main.c`

- **功能**：把 `Ps2Command` 映射为 `car_*` 调用，实现锁存、切挡、失联保护。
- **实现要点**：见 03 文档第 1 条；`Ps2Command{buttons,right_key,connected}`（`main.c` 38–43 行）。
- **边界**：这是**当前唯一**控制路径；`rc_control` 未接入；SELECT/蟹行/摇杆在此版本不用。

## 7. flight_comm（飞控协议模块）— `app/flight_comm.c` / `.h`

- **功能**：11 字节 CRC8 协议 + USART2 DMA 环形接收 + 失步重同步 + 500 ms 超时。
- **接口**：`flight_comm_init / flight_comm_update / flight_comm_get_command / flight_comm_is_connected`；条件编译的 `rc_backend_read`。
- **实现要点**：帧格式 `AA 55 | ver | enable | mode | throttle_hi lo | steer_hi lo | seq | crc8`；`flight_parser_resync` 丢字节重搜帧头。
- **依赖**：`huart2` + `hdma_usart2_rx`（DMA1_Ch6 Circular）。
- **边界**：`FLIGHT_COMM_ENABLE_CAR_OUTPUT=0`、`main()` 不调用 `flight_comm_init()`；USART2 被 `printf` 占用。

## 8. rc_control（遥控抽象层）— `app/rc_control.c` / `.h`

- **功能**：协议无关 `RcCommand` + 失联保护 + 弱符号后端钩子。
- **接口**：`rc_init / rc_update / rc_get_command / rc_backend_read(weak)`。
- **实现要点**：`RcCommand{throttle,steering,mode,enable,connected}`；`rc_update` 限幅 ±1000、500 ms 超时。
- **边界**：未接入运行路径；与 `flight_comm` 的强定义后端是「未来接线点」。

## 9. mylink_bridge（PX4 实验文本桥）— `MylinkBridge.cpp` / `.hpp`

- **功能**：串口读 `TAKEOFF / T <val> / LAND / STOP`，请求解锁/切 Offboard/直发执行器。
- **接口**：`init / Run / openPort / closePort / readAndParse / executeCommand / publishActuatorMotors`。
- **实现要点**：`O_NONBLOCK` 打开 `MLB_PORT`（默认 `/dev/ttyS3`）；`publishActuatorMotors` 给 M1~M4 写相同归一化推力；`MLB_ENABLE=0` 时 `Run()` 直接 return。
- **边界**：无认证/CRC/序列号/命令超时/可靠限幅；直接执行器输出；禁止实飞（`adhesion/README.md` 第 24–26 行）。

## 10. height_commander（PX4 相对高度模块）— `HeightCommander.cpp` / `.hpp`

- **功能**：RC 辅助开关边沿触发的相对高度状态机，发布 Offboard 位置目标。
- **接口**：`init / Run / updateStateMachine / publishOffboardSetpoint / sendOffboardModeCommand / resetToIdle / getAuxChannelValue`。
- **实现要点**：`IDLE→ASCENDING→HOLDING_HIGH→DESCENDING→IDLE`；NED 上升=z 更负；`HC_DELTA=5.0m`、`HC_CHANNEL=7`、`HC_ENABLE=0`。
- **边界**：通道 6/7 都映射 `aux2` 需实机前修正；无 ToF/触点/压紧/低电量退出逻辑（`TECHNICAL_DEVELOPMENT_DOCUMENT.md` 8.2）。

## 11. CRC8 模块 — `flight_crc8()`（`app/flight_comm.c`）+ `tools/send_car_command.py::crc8_atm()`

- **功能**：CRC-8/ATM 校验，poly `0x07`，init `0x00`，标准向量 `123456789→0xF4`。
- **实现要点**：C 与 Python 双实现（`send_car_command.py` 27–34 行），可互相验证。
- **边界**：当前运行固件不调用；`mylink_bridge` 没有 CRC，两者**不是同一协议**（`README.md` 第 62 行）。

## 12. PID 模块 — `com/pid.c` / `pid.h`

- **功能**：位置式 + 增量式 PID。
- **实现要点**：`pid_calc`（位置式，抗饱和）、`pid_calc_inc`（增量式，死区 0.04 m/s，步长钳 ±3.0）。
- **边界**：未启用；仅被 `encoder.c`/`chassis_task.c`（未调度）引用。

## 13. encoder 模块 — `bsp/encoder.c` / `encoder.h`

- **功能**：TIM1/3/4/8 编码器计数、轮速换算、低通滤波、TIM6 周期采样。
- **实现要点**：`Encoder_Calc_Mps` 用 `ENC_LINE=11, ENC_MULTIPLE=4, GEAR_RATIO=30, WHEEL_DIAMETER=0.065, CAPTURE_PERIOD=0.02`；`HAL_TIM_PeriodElapsedCallback` 每 20 ms 计算四轮速度并 `pid_calc_inc`。
- **边界**：`main()` 不启动编码器定时器、不启动 TIM6、不使能其 NVIC 入口调用链。

## 14. ROS 通信模块 — `app/ros_comm_task.c` / `.h`

- **功能**：UART4 DMA+IDLE 接收 24 字节帧、XOR 校验、控制/IP 指令、300 ms 超时。
- **实现要点**：`check_sum`（XOR）、`ros_comm_recv_proc`（帧头 `FRAME_HEADER`、帧尾校验）、`xyz_speed_transition`。
- **边界**：FreeRTOS 任务体，`MX_FREERTOS_Init` 未调用 → 未运行。

## 15. IMU 模块 — `bsp/bsp_imu.c` / `bsp_imu.h` + 软件 I2C `bsp_iic.c`

- **功能**：读 0x50 地址 IMU 寄存器（0x30 起 32 字节），换算加速度/角速度/角度。
- **实现要点**：软件 I2C（PB8=SCL、PB9=SDA），`IIC_ReadBytes(0x50,0x30,32,...)`；`accel/32768*16`、`gyro/32768*2000`。
- **边界**：未初始化；`IIC_Init()` 空实现（依赖 CubeMX GPIO 已配置）。

## 16. 板级配置模块 — `adhesion/px4-overlay/boards/*/default.px4board`

- **功能**：把自定义模块编进不同 PX4 目标。
- **实现要点**：`micoair/h743-v2` 只加 `CONFIG_MODULES_HEIGHT_COMMANDER=y`；`px4/fmu-v6x` 只加 `CONFIG_MODULES_MYLINK_BRIDGE=y`；`px4/sitl` 两者都加。
- **边界**：「出现在板级配置里」≠「编译/启动/实飞成功」（`TECHNICAL_DEVELOPMENT_DOCUMENT.md` 第 408 行）。
