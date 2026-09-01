# 12 代码导航（5 分钟 GitHub 屏幕共享演示路线）

> 目标：面试官让你「打开 GitHub 现场走一遍」，5 分钟内展示「我知道每个文件在哪、每段代码讲了什么」。时间轴 + 点哪个文件 + 滚到哪个函数 + 讲哪三句话。

## 时间轴总览

| 时间 | 动作 | 核心信息 |
|---|---|---|
| 0:00–0:40 | 根 README | 项目定位 + 证据边界 |
| 0:40–1:30 | `Src/main.c` | 上电顺序 + 主循环 + 失联保护 |
| 1:30–2:10 | `bsp/servo.c` | 软硬件 PWM 混合 |
| 2:10–2:45 | `bsp/motor.c` | J4 direction + 换向清 PWM |
| 2:45–3:15 | `app/flight_comm.c` | CRC8 协议 + 安全门 |
| 3:15–3:45 | PX4 `MylinkBridge.cpp` | 主动暴露风险 |
| 3:45–4:15 | `docs/PROJECT_STATUS.md` | 三分法收尾 |
| 4:15–5:00 | 留白应对追问 | 见 03 文档完整导览 |

---

## 0:00–0:40 根 `README.md`

- **点开**：仓库根目录 `README.md`。
- **滚到**：第 40–52 行「当前小车运行版本」表格。
- **讲三句话**：
  1. 「这个仓库是无人机贴面小车平台，车控 + PX4 覆盖层两块。」
  2. 「上电 `car_init→car_stop→steering_center`，然后进 PS2 控制主循环。」
  3. 「这里我特意写了：±90° 是软件命令上限，实际 PWM 被 1300~1700 μs 限幅，不等价于机械转角实测。」

## 0:40–1:30 `car/firmware/stm32f103-car-controller/Src/main.c`

- **点开**：`Src/main.c`。
- **滚到**：先 `main()`（340–401 行），再 `ps2_update()`（146–174 行）。
- **讲三句话**：
  1. 「`main` 只初始化 GPIO/TIM2/TIM5/USART2/UART5，不启动 FreeRTOS、编码器、飞控通信。」
  2. 「主循环 10 ms：`ps2_update → ps2_apply_command → ps2_debug_update`。」
  3. 「`ps2_update` 里超 500 ms 没合法 0x01 帧就 `car_stop + steering_center` 并清锁存。」

## 1:30–2:10 `bsp/servo.c`

- **点开**：`bsp/servo.c`。
- **滚到**：`servo_config` 表（22–28 行）+ `servo_tim5_irq_handler`（332–376 行）。
- **讲三句话**：
  1. 「四路舵机标定：center 1500、min 1300、max 1700 μs。」
  2. 「CH1/CH2 是硬件 PWM，CH3/CH4 引脚被 USART2 占了，只能做内部比较时刻。」
  3. 「所以 PC5/PB0 我在更新中断拉高、比较中断拉低，手搓两路软件 PWM，ISR 里只碰 CCR 和 GPIO。」

## 2:10–2:45 `bsp/motor.c`

- **点开**：`bsp/motor.c`。
- **滚到**：`motor_config` 表（30–36 行）+ `motor_apply_output`（63–91 行）。
- **讲三句话**：
  1. 「四个电机 TIM2 20 kHz，`PSC=35/ARR=99`，满量程 100 计数。」
  2. 「J4 那个轮子安装镜像，我在这里把 direction 设成 -1，不动上层。」
  3. 「换向前先清 PWM 再改方向脚，停止是 IN1=IN2=0 滑行，不是短路刹车。」

## 2:45–3:15 `app/flight_comm.c`

- **点开**：`app/flight_comm.c`。
- **滚到**：`flight_crc8()`（34–57 行）+ `rc_backend_read()`（309–340 行）。
- **讲三句话**：
  1. 「这是 11 字节飞控协议：AA 55 帧头 + 字段 + CRC8，CRC-8/ATM poly 0x07。」
  2. 「DMA 环形接收 + 失步重同步，丢一个字节不会永久错位。」
  3. 「但当前 `FLIGHT_COMM_ENABLE_CAR_OUTPUT=0`，且 main 没调 init——协议在、诊断做过，正式运行不接飞控。」

## 3:15–3:45 `adhesion/px4-overlay/src/modules/mylink_bridge/MylinkBridge.cpp`

- **点开**：`MylinkBridge.cpp`。
- **滚到**：`publishActuatorMotors()`（144–158 行）。
- **讲三句话**：
  1. 「mylink_bridge 读 TAKEOFF/T/LAND/STOP 文本命令。」
  2. 「这里直接给 M1~M4 写相同推力，绕过姿态闭环。」
  3. 「它没有 CRC、认证、超时，我在 README 里明确标了禁止实飞——这是实验代码不是可飞行代码。」

## 3:45–4:15 `docs/PROJECT_STATUS.md`

- **点开**：`docs/PROJECT_STATUS.md`。
- **滚到**：全文三栏。
- **讲三句话**：
  1. 「我把状态分成三类：已实测、已实现待回归、未完成。」
  2. 「已实测是四舵机四电机、PS2 链路、J4 方向修正；待回归是 30/60/80 三挡和 ±90 转向；未完成是飞控联调、编码器 PID、仲裁。」
  3. 「这套边界就是我面试里一直强调的东西——能编译 ≠ 已实测。」

---

## 演示技巧

1. **先给地图再进细节**：每点开一个文件，先说「它在哪一层、干什么」，再滚到函数。
2. **滚动用函数名导航**：提前记住 `servo_config`、`motor_apply_output`、`flight_crc8`、`publishActuatorMotors` 这几个锚点，用编辑器「跳到定义」比手滚快。
3. **主动停顿问「要我展开哪个」**：5 分钟不一口气灌完，留 1 分钟给面试官选方向。
4. **每个风险主动带一句**：飞控、蟹行、PID、90° 四个词一出现，就跟「还没实测/没启用」。
