# 00 项目总览（Project Overview）

> 面试开场用的「一句话讲清项目 + 边界诚实表述」。所有结论都有仓库证据；无证据的一律标注【待本人确认】/【根据代码推断】。

## 1. 一句话定义

一个「多旋翼提供法向正压 + 顶部四轮转向小车贴面移动 + 检测载荷采集」的组合式攀附平台；本仓库交付的是其中 **STM32F103 四舵机四电机小车固件** 与 **PX4 攀附飞控自定义模块覆盖层** 两大部分的可公开、可复现源码。

> 证据：根 `README.md` 第 1–5 行；`docs/TECHNICAL_DEVELOPMENT_DOCUMENT.md` 第 27–29 行（2.1 总体目标）。

## 2. 背景与动机

- 场景：无人机需要把检测载荷送到高空壁面/顶面并「贴住」工作，单靠多旋翼悬停不够稳、不够省电；方案是让无人机负责「送达 + 施加并维持法向正压」，小车负责「贴面后的四轮移动 + 转向 + 停车」。
- 因此系统天然分成两个控制器：**飞控（PX4）** 和 **车控（STM32F103）**，中间有一条 UART 命令链路 + 一套控制权仲裁需求。
- 证据：`docs/TECHNICAL_DEVELOPMENT_DOCUMENT.md` 2.1 节（29 行）与第 47–54 行 mermaid 框图。

## 3. 应用场景（面试可讲，但要说清当前边界）

| 场景 | 当前状态 |
|---|---|
| 高空壁面/顶面检测（雷达载荷） | 方案级；雷达同步、载荷、结构强度【未实现】 |
| 四轮小车本地遥控移动 | **已实测**（PS2 → CH559 → UART5 → STM32 原始链路） |
| 飞控相对高度实验 | 源码已收录，**待编译/待台架/待实飞** |
| 飞控—车控 UART 联调 | 协议模块已存在，**未接入运行固件** |

> 证据：`docs/PROJECT_STATUS.md` 全文；根 `README.md` 第 5 行、第 40–62 行。

## 4. 系统需求（从代码反推）

1. **四路转向舵机独立控制**：50 Hz PWM，中位 1500 μs，范围 1300~1700 μs —— `bsp/servo.c` 标定表。
2. **四路电机独立驱动**：20 kHz PWM + TB6612 方向脚，车辆坐标正负速度语义 —— `bsp/motor.c`。
3. **普通四轮转向（前后轴反向）**：`steering_turn()` 前轴 `+angle`、后轴 `-angle` —— `bsp/steering.c` 第 78–87 行。
4. **失联安全**：PS2 超 500 ms 无有效 `0x01` 报告即停车回中 —— `Src/main.c` 第 166–173 行。
5. **分层可替换控制源**：`rc_control` 抽象层 + 弱符号 `rc_backend_read()`，未来换 PS2/SBUS/飞控不改 `car.c` —— `app/rc_control.c` 第 38–42 行。

## 5. 难点（真实难点 + 面试可展开）

| 难点 | 代码体现 | 说明 |
|---|---|---|
| 只有 TIM5 一路定时器资源，却要 4 路舵机 | `bsp/servo.c` 硬件 CH1/CH2 + 软件 CH3/CH4 | CH3/CH4 用「更新中断拉高 + 比较中断拉低」在 GPIO 上做软件 PWM |
| 软件 PWM 的抖动受中断延迟影响 | `servo_tim5_irq_handler()` 只做 GPIO 与 CCR 操作 | `SERVO_CALIBRATION.md` 明确「尚未用示波器做定量抖动测量」 |
| 大扭矩舵机同时启动电流冲击 | `servo_init()` 每路错开 100 ms | `SERVO_START_STAGGER_MS=100` |
| 方向镜像安装（J4 反装） | `motor_config[3].direction = -1` | `J4_DIRECTION_FIX.md` |
| 失联保护不能误判 | `ps2_update()` 用「最近一次合法 `0x01` 帧」计时，而非「非零摇杆活动」 | `CAR_CONTROL.md` 第 127–130 行点明旧逻辑缺陷 |

## 6. 「我的工作内容」分三档（面试必讲，按证据强度）

### 6.1 明确实现（代码 + 构建日志 + 分阶段测试记录）

- `car / steering / servo / motor` 四层车辆 API 的搭建与整合（`CAR_CONTROL.md` 说明这是「新增基础运动控制层」）。
- TIM2 四路电机 PWM + TB6612 方向/STBY 控制，含 80% 上限与 J4 `direction=-1` 修正（`bsp/motor.c`）。
- TIM5 两路硬件 PWM + 两路软件 PWM 的四舵机驱动与标定框架（`bsp/servo.c`、`SERVO_CALIBRATION.md`）。
- PS2 最小「按键直行/停止」实车版本，随后扩展到锁存动作 + 三挡调速 + ±90° 普通转向（`PS2_BASIC_CONTROL.md` → `PS2_BUTTON_TEST.md` → `PS2_NORMAL_TURN90.md`）。
- USART2 11 字节 CRC8 飞控协议 + DMA 环形接收 + 失步重同步 + 500 ms 超时（`app/flight_comm.c`），并做了 USB-TTL 往返诊断（`FLIGHT_UART.md` 第 13–19 行，返回 `OK/CRC_ERR`）。

### 6.2 根据代码推断（实现了，但需谨慎表述）

- `rc_control` 弱符号后端抽象层已就位，但**当前 PS2 主循环并未经过它**（`CODE_MAP.md` 第 10 行「rc_control 源码保留，但未进入当前运行路径」）。
- 编码器资源（TIM1/TIM3/TIM4/TIM8 + TIM6）与增量式 PID、里程计换算代码已存在，但**未在 `main()` 启动**（`Src/tim.c` 仅定义 `MX_TIM*_Init()`，`Src/main.c` 只调用 TIM2/TIM5）。

### 6.3 建议作为「设计思考」讲（不是已完成功能）

- 飞控—车控**唯一控制源仲裁层**（`TECHNICAL_DEVELOPMENT_DOCUMENT.md` 7.4 节的 Control Manager 构想）。
- 锁存式 L1 改为 **deadman（持续按住使能）** 的安全改进方向（`TECHNICAL_DEVELOPMENT_DOCUMENT.md` 第 290–292 行明确建议）。
- 编码器单轮速度 PID 的「逐轮验证 A/B 相、计数符号、每转计数、丢脉冲」引入顺序（`MOTOR_TEST.md` 第 8 节）。

## 7. 最终成果（不编数据，只讲可证明事实）

- **构建**：Keil MDK ARMCC 5.06，`0 Error(s), 0 Warning(s)`，`Code=16684 RO-data=852 RW-data=184 ZI-data=2696`（`car/releases/PS2_NORMAL_TURN90_SPEED30_60_80/keil_rebuild.log` 第 58–60 行）。
- **可烧录 HEX**：`1_template_led.hex`，SHA256 `ABE8271A…62962D17`（`car/README.md` 第 12–16 行）。
- **从代码结构看已实现**：上电 `car_init→car_stop→steering_center` → PS2 10 ms 控制循环；L1 直行、A/Y 后退、X/B ±90° 普通转向、R1 停、START 回中、L2 三挡切挡、500 ms 失联保护（`Src/main.c` 第 371–399 行）。
- **尚未量化验证**：舵机软件 PWM 抖动（μs 级）、±90° 机械转角无干涉、30/60/80% 实车回归——均需「后续通过示波器 + 架空测试量化验证」，不得在面试中报出具体抖动/速度数据。

## 8. 面试开场建议（一句话版本）

> 「这是我实习期间做的无人机贴面小车平台，我负责车控这一侧的 STM32F103 固件和 PX4 覆盖层。车控是裸机前后台：TIM2 四路电机 PWM + TIM5 四路舵机（两硬两软），PS2 经 CH559 走 UART5 进来，500 ms 失联自动停车；飞控侧写了 height_commander 和 mylink_bridge 两个 PX4 模块。目前小车执行层和 PS2 链路有实物验证，飞控联调还在台架前阶段——这个边界我会一直讲清楚。」
