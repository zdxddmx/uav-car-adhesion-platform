# 09 优化（Optimization）

> 分两部分：**已经实现的优化**（代码里真实存在，附证据）与 **如果继续迭代我会做的优化**（标注是「设计建议」而非已实现）。

---

## 1. 已经实现的优化

### 1.1 舵机启动错峰，降低瞬时电流（`bsp/servo.c`）

- 做法：`servo_init()` 里四路舵机逐路 `HAL_TIM_PWM_Start` / 使能，每路之间 `HAL_Delay(SERVO_START_STAGGER_MS=100)`（184–194 行）。
- 收益：四个大扭矩舵机同时上电的瞬时电流会被错开，减少电源压降与堵转冲击。
- 代价：上电到就绪多了 ~300 ms。

### 1.2 转向动作缓存，避免每周期重复错峰（`bsp/car.c`）

- 做法：`car_set_steering()` 用 `car_steering_mode` + `car_steering_angle_deg` 缓存，`CAR_STEERING_CHANGE_EPSILON=0.01` 判断是否变化，只有变化才调 `steering_turn/crab`（69–91 行）。
- 收益：避免 10 ms 主循环反复执行「4×100 ms 错峰」的舵机更新，把转向动作降为「仅变化时一次」。
- 代价：多两个静态状态变量；需保证「模式或角度变化」判定正确。

### 1.3 先清 PWM 再换向（`bsp/motor.c`）

- 做法：`motor_apply_output()` 先 `__HAL_TIM_SET_COMPARE(..., 0)`，再改 IN1/IN2，最后写幅值（68–90 行）。
- 收益：避免带占空比直接翻转 H 桥方向造成冲击/直通风险。

### 1.4 方向修正集中化（`bsp/motor.c` + `J4_DIRECTION_FIX.md`）

- 做法：`motor_config[].direction` 统一处理镜像安装，`motor_set_pwm` 里 `signed_pwm = pwm * direction`（196–197 行）。
- 收益：上层（car/PS2）不针对单轮打正负号补丁，方向语义可维护。

### 1.5 500 ms 失联保护（`Src/main.c` + `app/flight_comm.c` + `app/rc_control.c`）

- 做法：三层（PS2 主循环 / flight_comm / rc_control）都用「最近合法帧时间戳 + 500 ms」判失联，进 failsafe 停车。
- 收益：手柄关机、拔接收器、CH559 停发、飞控停发都不会维持旧油门。

### 1.6 协议健壮性：CRC8 + 失步重同步（`app/flight_comm.c`）

- 做法：`flight_validate_and_apply_frame` 校验帧头/CRC/字段范围；失败后 `flight_parser_resync` 重新搜 `AA 55`。
- 收益：一个丢字节不会永久错位后续所有帧（`FLIGHT_UART.md` 第 79 行）。

### 1.7 按键边沿去抖（`Src/main.c`）

- 做法：`ps2_l2_was_pressed` / `ps2_start_was_pressed` 用「上一周期状态」做上升沿检测。
- 收益：L2 长按只切一挡，START 长按只回中一次（`main.c` 224–234、208–221 行）。

### 1.8 PID 抗积分饱和（`com/pid.c`）

- 做法：`pid_calc` 里 `integral += err` 只在输出**未**饱和的 else 分支执行（42–44 行）。
- 收益：避免积分项在输出饱和后继续累积导致超调/震荡。

### 1.9 软件 PWM 的 ISR 精简（`bsp/servo.c`）

- 做法：`servo_tim5_irq_handler` 只写 CCR + GPIO，不 printf/不浮点（`SERVO_CALIBRATION.md` 第 19 行）。
- 收益：降低软件 PWM 脉宽抖动。

---

## 2. 如果继续迭代，我会做的优化

> 以下均为【面试扩展思路 / 设计建议】，当前**未实现**。

### 2.1 安全：L1 改 deadman + 物理急停

- 把锁存式 L1 改成「持续按住才允许运动」（`TECHNICAL_DEVELOPMENT_DOCUMENT.md` 290–292 行建议）。
- 加独立硬件急停，切断 TB6612 STBY 或电机电源（当前只有软件停车）。

### 2.2 安全：命令斜坡与看门狗

- `car_forward/car_backward` 当前是 30%↔80% 瞬时跃变，应加 `motor_set_pwm` 输出斜坡。
- 加 IWDG 独立看门狗 + 上电原因记录（当前 `stm32f1xx_it.c` 的 HardFault 只是死循环，无复位/日志）。

### 2.3 通信：USART2 日志分离 + 控制源仲裁

- `printf`（`com_debug.c` `fputc` 重定向到 `huart2`）与飞控协议同口，接入真飞控前必须迁移调试口到独立 UART（`FLIGHT_UART.md` 第 40 行）。
- 实现唯一控制源 Control Manager：PS2 与飞控不能同时直写 `car`，切换前先停车回中并要求新源重新使能（`TECHNICAL_DEVELOPMENT_DOCUMENT.md` 7.4）。

### 2.4 运动控制：编码器单轮速度环

- 先按 `MOTOR_TEST.md` 第 8 节顺序验证 A/B 相、计数符号、每转计数、丢脉冲，再启用单轮 PID。
- 复用 `encoder.c` 现成的 TIM6 20 ms 采样 + `pid_calc_inc`，但 Kp/Ki（旧值 282/85）需重新整定。

### 2.5 舵机：量化软件 PWM 抖动

- 示波器实测 PC5/PB0 的周期与高电平宽度（`SERVO_CALIBRATION.md` 第 19 行待办）。
- 若抖动超差，改用「硬件 PWM 引脚重新分配」或外扩 PWM 芯片（如 PCA9685），不再依赖中断。

### 2.6 代码质量：魔法数字收敛 + 消除阻塞

- `1300/1700/80/30/60/80/90/500/100` 等已部分进宏/数组，但 `motor.c` 的测试延时、`servo.c` 的标定步进仍散落，应收敛到配置头。
- `HAL_Delay(10)` 与 `steering` 内部 `HAL_Delay(100)` 是阻塞延迟，会引入抖动；改用定时器时基的非阻塞调度（如 `HAL_GetTick` 差值判时）。
- `printf` 阻塞发送改 DMA TX 环形缓冲（工程已配置 `hdma_usart2_tx`，但 `printf` 用的是阻塞 `HAL_UART_Transmit`）。

### 2.7 飞控侧：淘汰 mylink_bridge 危险路径

- 永久禁用或删除 `publishActuatorMotors` 直接执行器输出（`PROJECT_STATUS.md` 第 52 行「删除或永久禁用 mylink_bridge 的危险直接执行器路径」）。
- `height_commander` 修正 `getAuxChannelValue` 通道 6/7 都映射 aux2 的问题，补充 ToF/触点/压紧/低电量退出状态机。

---

## 3. 优化的「优先级排序」（面试口述）

> 「如果继续做，我按这个顺序：**先安全（deadman+急停+看门狗）→ 再通信（日志分离+仲裁）→ 再闭环（编码器单轮 PID）→ 最后性能（软件 PWM 量化+非阻塞调度）**。因为这是要上无人机倒挂贴面的东西，安全边界 > 功能闭环 > 性能优化。」
