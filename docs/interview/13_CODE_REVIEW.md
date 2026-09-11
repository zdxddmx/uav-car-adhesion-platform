# 13 代码质量审计（Code Review）

> 只记录问题、不改代码。按「严重度」分级：🔴 潜在真 Bug / 🟠 边界与健壮性 / 🟡 ISR·竞态·定时器 / ⚪ 可维护性。每一条标注「是否在当前运行路径上」。审计对象是自研代码（`bsp/` `app/` `com/` `Src/` + PX4 两个模块），第三方 HAL/FreeRTOS/CMSIS 不深挖。

---

## 1. 🔴 潜在 Bug（多数在「未启用」路径，但不代表不用修）

### 1.1 编码器计数 16 位符号截断 —— `bsp/encoder.c` 20–25 行
- **问题**：`int temp = (short)__HAL_TIM_GetCounter(htim)` 把 16 位计数值强制按有符号解释。TIM ARR=65535，计数值在 32768~65535 区间会被读成负数，计数跨越 32767 时符号跳变。
- **影响**：编码器计数方向/速度在高速或跨过半程时会突变。【当前未启用，一旦启动编码器 PID 必踩】。
- **建议**：用 16 位无符号差值的标准环形计数法（`int32_t delta = (int16_t)((uint16_t)cur - (uint16_t)prev)`）。

### 1.2 `xyz_speed_transition` 负数取余错误 —— `app/ros_comm_task.c` 81–85 行
- **问题**：`transition%1000` 在 C 里对负数结果也为负，`transition/1000+(transition%1000)*0.001` 对负速度的换算会拼出错误值。
- **影响**：ROS 速度回传在负方向错位。【未启用】。
- **建议**：统一用 `transition / 1000.0f`。

### 1.3 `getAuxChannelValue` 通道 6/7 都返回 aux2 —— `HeightCommander.cpp` 65–78 行
- **问题**：`case 6: return man_ctrl.aux2; case 7: return man_ctrl.aux2;`，两个物理通道映射同一 aux。
- **影响**：`HC_CHANNEL` 默认 7 时与实际 RC_MAP 可能不一致，实机触发通道错乱。【PX4 未实飞，仓库 README 已自报】。
- **建议**：实机前结合 `manual_control_setpoint` 定义与 RC_MAP 参数修正。

### 1.4 `vofa.c` 引用跨文件裸全局变量 —— `com/vofa.c` 35–41 行
- **问题**：`UserData[0]=&a ... UserData[6]=&g` 依赖 `chassis_task.c` 里定义的裸全局 `float a,b,c,d,e,f,g`，无头文件声明保护。
- **影响**：只要 `chassis_task.c` 被移出编译，`vofa.c` 链接即失败；变量语义无文档。【当前因 chassis_task.c 在编而侥幸通过】。

---

## 2. 🟠 边界与健壮性

### 2.1 `mylink_bridge` 无 CRC/认证/超时/限幅 —— `MylinkBridge.cpp`
- **问题**：`strtof(space+1, nullptr)` 不检查解析失败；`T` 值无范围钳位；`publishActuatorMotors` 直接给 M1~M4 写相同推力绕过姿态闭环；无命令心跳超时。
- **影响**：误码/注入/失控直接作用于执行器。【已用 `MLB_ENABLE=0` 禁用，但代码仍在】。
- **建议**：永久禁用或删除（`PROJECT_STATUS.md` 第 52 行已列）。

### 2.2 `printf`/阻塞发送占用 USART2 —— `com/com_debug.c` 3–7 行
- **问题**：`fputc` 用阻塞 `HAL_UART_Transmit(&huart2,...)`，与飞控协议共用 USART2；阻塞发送会停住调用者。
- **影响**：接入真飞控后 ASCII 日志会混进二进制协议链路。【当前只做调试，风险已知】。

### 2.3 `Delay_us` 忙等与主频耦合 —— `bsp/bsp_iic.c` 6–10 行
- **问题**：`Delay = us * 72 / 8` 硬编码 72 MHz；改主频或编译器优化会失准。
- **建议**：用 DWT 周期计数器或 HAL 定时器实现。

### 2.4 `motor_ctrol` 与 `car.c` 的电机映射语义不一致 —— `bsp/motor.c` 309–313 行
- **问题**：`motor_ctrol` 把 `chassis->motor[0..3]` 映射到 `MOTOR_3/1/4/2`（历史差速底盘顺序），而 `car.c` 是 `motor_set_all(s,s,s,s)` 对称直驱。两套语义并存易混淆。
- **影响**：未来有人把 `car.c` 换成 `motor_ctrol` 会得到错误轮序。【未启用路径】。

---

## 3. 🟡 ISR·竞态·定时器

### 3.1 `ps2_copy_line` 关中断期间推后 TIM5 软件 PWM —— `Src/main.c` 127–141 行
- **问题**：`__disable_irq` 保护下做 128 字节逐字节拷贝，期间 TIM5 更新/比较中断被屏蔽，PC5/PB0 边沿推迟。
- **影响**：舵机软件 PWM 脉宽抖动（μs 级）。【当前运行路径，这是软件 PWM 抖动的直接来源之一】。
- **建议**：只拷贝到 `'\0'/'\r'/'\n'` 就 break（代码已做），或缩短临界区；最终需示波器量化。

### 3.2 `servo_pending_us` 的 volatile 16 位共享 —— `bsp/servo.c` 30–33 行
- **问题**：主循环 `servo_set_us` 写、TIM5 ISR 读。Cortex-M3 对 16 位对齐访问是原子 LDRH/STRH，基本无撕裂；但「先改 pending 再写 CCR」不是原子序列，极端时序下 ISR 可能读到「新 pending + 旧 CCR」。
- **影响**：单周期脉宽瞬时不一致。【低风险，本应用可接受】。

### 3.3 UART5 与 TIM5 同为 NVIC 优先级 5 —— `Src/usart.c` 196 行 / `bsp/servo.c` 177 行
- **问题**：两个实时性敏感中断同优先级，按到达顺序串行，PS2 高频率字节会挤占舵机边沿。
- **建议**：若实测抖动超差，把 TIM5 提到更高抢占优先级（数值更小）。

### 3.4 TIM6 中断优先级 5，但 `SysTick` 是 15 —— `Src/tim.c` 486 行 / `stm32f1xx_it.c`
- **问题**：若未来启用 FreeRTOS，`configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY=5` 意味着优先级 5 的 ISR 不能调用 `xSemaphoreGiveFromISR` 等 API（`ros_comm_task.c` 恰好在 TIM6/DMA 场景用了这类调用，见 55–58 行）。
- **影响**：RTOS 集成时中断优先级边界需重审。【当前裸机不涉及】。

---

## 4. ⚪ Watchdog·异常恢复·魔法数字

### 4.1 无看门狗、HardFault 死循环 —— `Src/stm32f1xx_it.c` 93–103 行
- **问题**：`HardFault_Handler` 只有 `while(1)`，`Error_Handler` 是 `__disable_irq(); while(1)`；工程未启用 IWDG/WWDG。
- **影响**：电机运动时程序跑飞/硬件异常 → MCU 卡死 → 电机可能维持最后 PWM 输出（无硬件复位兜底）。
- **建议**：加 IWDG + 上电原因记录 + HardFault 里主动 `motor_stop_all`。

### 4.2 阻塞式 `HAL_Delay` 贯穿执行路径 —— `bsp/car.c` 经 `steering.c` → `servo.c`
- **问题**：`steering_set_four_angles` 每次转向 4×100 ms `HAL_Delay`，`car_turn` 执行期间主循环被阻塞 ~400 ms，PS2 解析停摆。
- **影响**：转向时 500 ms 失联保护的响应被延长；`car.c` 的转向缓存只是「减少频率」不是「消除阻塞」。
- **建议**：非阻塞调度（时基差值判时）或把错峰下放到定时器状态机。

### 4.3 魔法数字散落
- **已收敛**：`servo_config`（1500/1300/1700）、`motor_config`（80）、`ps2_speed_percent`（30/60/80）、`PS2_CONNECTION_TIMEOUT_MS`（500）、`PS2_MAX_STEERING_ANGLE_DEG`（90）。
- **未收敛**：`motor.c` 测试延时 2000/1000/3000 ms、`servo.c` 标定步进 50 μs/1000 ms、`steering.c` 2000/3000 ms，仍散落为裸字面量。
- **建议**：统一到 `*_config.h`，测试参数与运行参数分文件。

### 4.4 GBK/UTF-8 编码混乱 —— `app/ros_comm_task.c` 等
- **问题**：部分文件 GBK 编码（`chassis_mode.h`、`chassis_task.h`、`app_key.c`、`pid.h`、`calc_fun.h`、`com_debug.h`），`ros_comm_task.c` 内出现「锟斤拷」乱码，说明曾被错误转码。
- **影响**：跨 IDE/跨平台 diff 与维护易出错；不影响 ARMCC 编译（注释被忽略），但字符串常量若被转坏会出问题。
- **建议**：统一 UTF-8 并保留原注释。

---

## 5. 审计结论（一句话）

> 当前运行路径（`main.c` + `car/steering/servo/motor` + `ps2_usart5`）整体是**防御式写法的裸机代码**：有空指针检查、数组越界保护、配置表集中、失联兜底；真正的风险集中在「未启用的编码器/PID/ROS/PX4」路径（符号截断、负数取余、直接执行器输出），以及「软件 PWM 抖动 + 无看门狗 + 阻塞延迟」这三点系统级欠账。这些都应作为面试里「我知道还欠什么」的主动表达。
