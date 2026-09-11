# 10 面试问答（Interview Q&A）

> 20~40 题，按 ★（基础）/★★（进阶）/★★★（硬核）/★★★★（系统设计）分级。所有答案**必须结合本仓库真实代码**，附文件/函数/宏证据。

---

## ★ 基础题

### Q1. 这个项目整体是做什么的？
无人机提供法向正压 + 顶部四轮转向小车贴面移动 + 检测载荷采集；我负责 STM32F103 车控固件和 PX4 飞控覆盖层。证据：`README.md` 第 1 行、`TECHNICAL_DEVELOPMENT_DOCUMENT.md` 2.1。

### Q2. 你用的主控是什么？资源和时钟？
STM32F103RCT6，Cortex-M3，256 KB Flash / 48 KB RAM，72 MHz。HSE 16 MHz ÷2 → PLL ×9 → 72 MHz（`.ioc` `HSE_VALUE=16000000`、`PLL_MUL9`；`uvprojx` `IROM/IRAM`）。

### Q3. 四个电机怎么驱动的？
TIM2 四路 PWM（20 kHz，`PSC=35/ARR=99`）→ TB6612 方向脚 + STBY。`motor_set_pwm` 先乘 `direction`、限幅 `pwm_max`，`motor_apply_output` 先清 PWM 再换向。

### Q4. 四个舵机怎么驱动的？为什么有「硬件 PWM」和「软件 PWM」之分？
TIM5 50 Hz（`PSC=71/ARR=19999`，1 μs/计数）。CH1/CH2（PA0/PA1）硬件 PWM；CH3/CH4 引脚被 USART2 占用，所以 CH3/CH4 只当比较时刻，用 GPIO（PC5/PB0）在更新中断拉高、比较中断拉低做软件 PWM（`servo.c` 332–376 行）。

### Q5. PS2 手柄是怎么接进来的？
PS2 → 2.4G 接收器 → CH559 USB Host → UART5（PC12/PD2，57600 8N1）→ STM32 逐字节中断 → `sscanf` 解析 `HUB0_Joystick data:` 8 字节。

### Q6. 失联了怎么办？
超 500 ms 没收到合法 `0x01` 帧 → `car_stop()+steering_center()+ps2_command_clear()`（`main.c` 166–173 行）。

### Q7. 编译通过了吗？固件多大？
Keil ARMCC 5.06，`0 Error, 0 Warning`，`Code=16684 RO-data=852 RW-data=184 ZI-data=2696`（`keil_rebuild.log` 58–60 行）。

---

## ★★ 进阶题

### Q8. 为什么舵机用软件 PWM，而不是再加一路定时器？
TIM1/3/4/8 被保留给编码器，且 PA2/PA3（TIM5 CH3/CH4 物理脚）被 USART2 占用。用 CH3/CH4 的 `TIM_OCMODE_TIMING` 只做内部比较，GPIO 手动翻转是最省资源的折中（`servo.c` 126–135 行）。

### Q9. 软件 PWM 有什么风险？你怎么缓解？
风险是脉宽抖动取决于中断延迟。缓解：ISR 只写 CCR + GPIO（不 printf/浮点），NVIC 优先级 5。但**尚未**用示波器量化（`SERVO_CALIBRATION.md` 第 19 行）——这是诚实边界。

### Q10. 为什么 J4 电机的 direction 是 -1？
实车确认 J4（PA0 车轮）驱动电机安装方向镜像，前进方向反了。只在 `motor_config` 表里改 `-1`，不改舵机（`J4_DIRECTION_FIX.md`）。

### Q11. 为什么换向要先把 PWM 清零？
避免带占空比直接翻转 H 桥方向造成冲击/直通风险（`motor.c` 第 68 行注释）。

### Q12. `motor_stop_all` 是刹车还是滑行？
滑行停止：PWM=0 且 IN1=IN2=0（`motor_apply_output` 83–88 行），不是 IN1=IN2=1 的短路刹车。

### Q13. 500 ms 判失联，为什么是 500 ms 不是 100 ms？
500 ms 是「链路心跳」判据——以「最近合法帧到达」为基准；旧代码用 100 ms「非零摇杆活动」判据，会把「松手保持中位」误判为失联（`CAR_CONTROL.md` 7.2）。500 ms 对遥控车既保证及时停车，又不会因偶发丢帧误触发。

### Q14. 主循环周期是多少？怎么实现的？
10 ms，`while(1){ ps2_update; ps2_apply_command; ps2_debug_update; HAL_Delay(10); }`（`main.c` 388–399 行）。

### Q15. `ps2_copy_line` 为什么要关中断？
`uart5_rx_buf` 由 UART5 ISR 写、主循环读，拷贝时若被中断打断会出现半帧。用 `__disable_irq/__enable_irq` 做原子拷贝（`main.c` 127–141 行）。

---

## ★★★ 硬核题

### Q16. 你的 CRC8 是什么参数？覆盖哪些字节？
CRC-8/ATM：poly `0x07`、init `0x00`、RefIn/RefOut=False、XorOut `0x00`，覆盖 Byte0~Byte9（10 字节），标准向量 `123456789→0xF4`。证据 `flight_comm.c` 34–57 行、`FLIGHT_UART.md` 81–92 行。

### Q17. CRC8 模块为什么当前没启用？
`flight_comm.h` `FLIGHT_COMM_ENABLE_CAR_OUTPUT=0`，且 `main()` 不调 `flight_comm_init()`；USART2 被 `printf` 占用做调试。所以协议代码在、诊断做过（回 OK/CRC_ERR），但正式运行路径不接飞控（`FLIGHT_UART.md` 第 15–19 行）。

### Q18. 如果飞控要接管小车，你会怎么改？
① USART2 日志迁移到独立口；② 实现唯一控制源仲裁（PS2 与飞控互斥）；③ `FLIGHT_COMM_ENABLE_CAR_OUTPUT` 单独评审后置 1；④ 飞控周期发 11 字节协议并映射遥控失效为 `enable=0`；⑤ 切换前先停车回中、新源重新使能（`TECHNICAL_DEVELOPMENT_DOCUMENT.md` 7.4、`FLIGHT_UART.md` 第 10 节）。

### Q19. 丢一个字节会不会让后续所有帧错位？为什么？
不会。`flight_parser_push` 攒满 11 字节校验失败后，`flight_parser_resync` 在缓冲内重新搜 `AA 55`（`flight_comm.c` 146–177 行），单字节丢包不永久错位。

### Q20. TIM2 的 20 kHz 和 TIM5 的 50 Hz 分别怎么算？
`72e6/(35+1)/(99+1)=20 kHz`；`72e6/(71+1)/(19999+1)=50 Hz`（`MOTOR_TEST.md` 第 25–31 行）。TIM5 计数单位恰好 1 μs，直接拿 μs 写 CCR。

### Q21. PA15/PB3 是 JTAG 脚，你的 PWM 为什么能用？
`HAL_TIM_MspPostInit` 里 `__HAL_AFIO_REMAP_TIM2_ENABLE()` 把 TIM2 CH1/CH2/CH3/CH4 remap 到 PA15/PB3/PB10/PB11（`tim.c` 532 行）。

### Q22. `servo_set_angle` 的 ±90° 是怎么换算成 μs 的？
钳到 ±90° → 乘 `direction` → 正角度映射到 `center~max`、负角度映射到 `min~center`，`(output_us+0.5)` 四舍五入后 `servo_set_us` 限幅（`servo.c` 262–299 行）。

### Q23. 编码器/PID 代码在哪？为什么没跑？
`bsp/encoder.c`（TIM6 20 ms 采样 + `pid_calc_inc`）、`com/pid.c`、`Src/tim.c`（TIM1/3/4/8 编码器初始化）。没跑是因为 `main()` 不调用这些 Init、不启动 TIM6（`main.c` 364–368 行只 init TIM2/TIM5）。

### Q24. FreeRTOS 到底启没启动？
没启动。`MX_FREERTOS_Init()` 定义了 4 个任务但 `main()` 从不调用，`osKernelStart()` 也不存在（`freertos.c` 91–129 行 vs `main.c` 340–401 行）。

### Q25. 你的 PID 是位置式还是增量式？有没有抗积分饱和？
两套都有：`pid_calc`（位置式，积分只在输出未饱和时累加 = 抗饱和）、`pid_calc_inc`（增量式，死区 0.04 m/s）（`pid.c` 22–89 行）。

---

## ★★★★ 系统设计题

### Q26. 这个系统里「谁是唯一控制源」现在是怎么解决的？
**还没解决**。当前只有 PS2 直写 `car`；`rc_control`/`flight_comm` 有抽象层但未接入运行路径，也没有仲裁层（`CODE_MAP.md` 第 10 行）。这是已知未完成项。

### Q27. 如果 PS2 和飞控同时发命令怎么办（现在的行为 vs 应该的行为）？
现在：只有 PS2 路径生效，飞控协议根本没接，所以不会同时。应该：加 Control Manager，切换前停车回中、要求新源重新使能，任一源失联不得继承另一源旧命令（`TECHNICAL_DEVELOPMENT_DOCUMENT.md` 7.4）。

### Q28. mylink_bridge 为什么禁止实飞？
它读 ASCII `TAKEOFF/T/LAND/STOP`，`publishActuatorMotors` 给 M1~M4 写相同推力，绕过姿态闭环；且无认证/CRC/序列号/命令超时/可靠限幅（`MylinkBridge.cpp` 144–158 行、`README.md` 第 62 行）。

### Q29. height_commander 的状态机是什么？有什么已知问题？
`IDLE→ASCENDING→HOLDING_HIGH→DESCENDING→IDLE`，RC 上升沿升 `HC_DELTA`、下降沿回原高。已知问题：`getAuxChannelValue` 把通道 6 和 7 都映射 `aux2`，实机前要修正；无 ToF/触点/压紧/低电量退出（`HeightCommander.cpp` 69–77、177–207 行）。

### Q30. NED 坐标里「上升」为什么是 z 更负？
PX4 用 NED（North-East-Down），z 轴向下为正，所以高度增加 = z 数值减小。代码里 `_target_altitude = current_z - delta`（`HeightCommander.cpp` 181 行）。

### Q31. 如果要加 MAVLink，你会怎么接？
新增独立 MAVLink backend，输出统一 `RcCommand`，MAVLink heartbeat/控制消息纳入 500 ms 超时；不把 MAVLink 解析写进 `car.c`，不同时启用两个有控制权的强定义 backend（`FLIGHT_UART.md` 第 182–184 行）。

### Q32. 这个项目的证据边界怎么界定？面试官为什么在意这个？
仓库用「已实测 / 已实现待实测 / 未完成」三分法（`PROJECT_STATUS.md`），并把「构建成功 ≠ 烧录 ≠ 实车验证」写死（`PS2_NORMAL_TURN90.md` 第 37 行）。面试官在意：嵌入式候选人最容易把「能编译」说成「已验证」，能主动区分才是靠谱工程师。

---

## 追问陷阱题（提前准备）

### Q33. 你的舵机真的能转到 90° 吗？
不能保证。软件允许 ±90° 命令，但 PWM 被 1300~1700 μs 限幅，机械转角取决于舵机/连杆，未做 ±90° 无干涉实测（`README.md` 第 52 行）。

### Q34. 你的小车能「蟹行」吗？
代码有 `steering_crab`/`car_crab`，但当前 PS2 版本**取消了蟹行**（`PS2_NORMAL_TURN90.md` 第 5 行「已彻底取消 CRAB」），且蟹行只有斜向能力、不代表 90° 横移验证（`CAR_CONTROL.md` 第 53 行）。

### Q35. 你的编码器 PID 调好了吗？
没有。编码器资源保留、代码存在，但未启动、未标定方向、未整定（`MOTOR_TEST.md` 第 8 节、`PROJECT_STATUS.md`）。

### Q36. 飞控和车控联调到什么程度？
飞控侧模块待编译/待台架/待实飞；STM32 侧 11 字节协议做过 USB-TTL 诊断，但两套协议不同、未联通、无仲裁（`PROJECT_STATUS.md` 第 28–45 行）。
