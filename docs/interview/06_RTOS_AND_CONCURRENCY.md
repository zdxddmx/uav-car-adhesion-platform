# 06 RTOS 与并发（RTOS & Concurrency）

> 本项目当前是**裸机前后台（bare-metal foreground/background）**。文档先讲清楚「为什么是裸机、中断怎么组织、有哪些竞态」，再给【面试扩展思路】讲「若引入 FreeRTOS / 编码器 PID 会怎么改」。

---

## 1. 结论先行：当前无 RTOS

- FreeRTOS 源码（`Middlewares/Third_Party/FreeRTOS/`）与初始化函数 `MX_FREERTOS_Init()`（`Src/freertos.c` 91–129 行）**都存在于工程里**。
- 但 `Src/main.c` 第 364–368 行只调用 `MX_GPIO/TIM2/TIM5/USART2/UART5_Init`，**没有调用** `MX_FREERTOS_Init()`，`osKernelStart()` 也不存在。
- 因此运行时是「主循环 + 中断」的前后台模型；`freertos.c` 里定义的 4 个任务（defaultTask / chassic_task / remote_task / roscommTask）从未被创建调度。
- 证据：根 `README.md` 第 40 行「没有启动 FreeRTOS」；`CAR_CONTROL.md` 第 8 行；`CODE_MAP.md` 第 10 行。

## 2. 中断结构（前后台的真实「并发」来源）

### 2.1 中断一览（NVIC 优先级证据 `1_template_led.ioc` / `Src/dma.c`）

| 中断 | 优先级（抢占/子） | 作用 | 处理函数 |
|---|---|---|---|
| TIM5 | 5/0 | 舵机软件 PWM（UPDATE/CC3/CC4） | `TIM5_IRQHandler → servo_tim5_irq_handler` |
| UART5 | 5/0 | PS2 单字节接收 | `UART5_IRQHandler → HAL_UART_IRQHandler → RxCpltCallback` |
| USART2 | 5/0 | 调试 printf + 飞控 DMA | `USART2_IRQHandler` |
| DMA1_Ch6/7 | 5/0 | USART2 RX/TX DMA | `DMA1_Channel6/7_IRQHandler` |
| DMA2_Ch3 | 5/0 | UART4 RX DMA（未用） | `DMA2_Channel3_IRQHandler` |
| TIM6 | 5/0 | 编码器采样（未启动） | `TIM6_IRQHandler` |
| SysTick | 15/0 | HAL 时基 | `SysTick_Handler → HAL_IncTick` |

> 关键观察：TIM5（舵机）与 UART5（PS2）**同为抢占优先级 5**，且 TIM5 是 50 Hz 边沿中断、UART5 是逐字节中断。一旦 UART5 接收频率或主循环关中断时间过长，TIM5 软件 PWM 边沿会被推迟 → 舵机脉宽抖动（`SERVO_CALIBRATION.md` 明确「软件 PWM 尚未用示波器做定量抖动测量」）。

### 2.2 TIM5 软件 PWM 中断（`servo.c` 332–376 行）

- UPDATE 中断：把 CCR3/CCR4 设成下一周期脉宽，并把 PC5/PB0 拉高。
- CC3/CC4 比较中断：分别把 PC5/PB0 拉低。
- ISR 里只做「写 CCR + 写 GPIO」，刻意不跑 printf/浮点/长逻辑（`SERVO_CALIBRATION.md` 第 19 行）。

### 2.3 UART5 中断（`ps2_usart5.c` 39–60 行）

- `HAL_UART_RxCpltCallback` 把 `rx_data` 写入 `uart5_rx_buf`，遇 `'\n'` 或 64 字节置 `uart5_rx_finish=1`。
- 每字节重新 `HAL_UART_Receive_IT`，形成「中断驱动环形接收」。

## 3. 主循环调度（`Src/main.c` 388–399 行）

```c
while (1) {
    ps2_update();            // 解析 + 500ms 失联判断
    ps2_apply_command();     // 锁存/切挡/执行 car_* 
    ps2_debug_update();      // 每 200ms printf
    HAL_Delay(10);           // 10ms 周期
}
```

- 这是**协作式/阻塞式**调度：`HAL_Delay` 阻塞等待，周期 ≈ 10 ms + 每次 `car_*` 内部可能的 `HAL_Delay(100×4)`（舵机错峰）。
- 后果：转向更新最坏会阻塞主循环 ~400 ms（`steering_set_four_angles` 4 次 100 ms），期间 PS2 解析暂停——这正是 `car.c` 加「转向缓存」的原因（`car_set_steering` 只在模式/角度变化时更新）。

## 4. 临界区 / 竞态分析（真实存在的点）

| 共享资源 | 生产者 | 消费者 | 保护方式 | 评估 |
|---|---|---|---|---|
| `uart5_rx_buf` / `uart5_rx_finish` | UART5 ISR | 主循环 `ps2_copy_line` | `__disable_irq()/__enable_irq()` 原子拷贝 | 正确，但关中断期间 TIM5 边沿被推迟（抖动来源之一） |
| `ps2`（全局结构） | 主循环 `ps2_parse_data` | 主循环 `ps2_apply_command` | 无（同一上下文） | 无竞态 |
| `servo_pending_us`（volatile） | 主循环 `servo_set_us` | TIM5 ISR | `volatile` + 写 CCR 是硬件原子操作 | 基本安全，但「读-改-写」非原子，理论上有 16 位撕裂风险（μs 量级，本应用可接受） |
| `motor_config`（非 volatile） | `motor_init`（启动期） | `motor_set_pwm`（主循环） | 启动期单线程 | 安全 |

> 关键点：`ps2_copy_line()` 用关中断做原子拷贝（`main.c` 127–141 行），这是裸机下最典型的「ISR↔主循环共享缓冲」处理手法，面试能讲出「为什么关中断、代价是什么」即可加分。

## 5. 为什么当前不用 RTOS（结合仓库真实理由）

仓库没有直接说「为什么不用」，但可从边界推断并诚实标注【根据代码推断】：

1. **当前任务结构极简**：只有「解析 PS2 + 下发车辆命令」一条链路，10 ms 轮询足够，没有需要抢占的多任务（`main.c` 主循环只有 3 个函数）。
2. **软件 PWM 依赖低抖动中断**：舵机软件 PWM 直接依赖 TIM5 中断及时响应，引入 RTOS 的 `portYIELD_FROM_ISR`/临界区会**增加**脉宽抖动风险（`SERVO_CALIBRATION.md` 第 19 行反向印证「ISR 只做 GPIO/CCR 以降低抖动」）。
3. **安全可控优先**：锁存控制 + 500 ms 失联 + 上电停车，需要的是「确定的执行顺序」而不是「并发吞吐」。
4. **FreeRTOS 是 CubeMX 骨架自带**：`freertos.c` 是 CubeMX 生成的模板，里面的任务体（`chassic_task` 等）是历史差速底盘遗留，不是当前交付物。

> 面试话术：不是「不会用 RTOS」，而是「这个阶段用裸机前后台更合适——控制链短、要保证舵机中断低抖动、且要明确谁是唯一控制源」。

## 6. 【面试扩展思路】若引入 FreeRTOS 会怎么改

1. **任务划分**：
   - `PS2Task`（高优先级，`osPriorityHigh`）：`HAL_UART_Receive_IT` 收字节 → 队列 `xQueueSend` 给解析任务；或直接 `ps2_parse_data`。
   - `ControlTask`（中优先级）：`xQueueReceive` 拿 `Ps2Command` → `car_*` 下发。
   - `DebugTask`（低优先级）：200 ms `printf`，从全局状态读取。
2. **同步原语**：`uart5_rx_finish` 换成二值信号量/队列，`ps2_line` 用队列传递，**不再需要 `__disable_irq` 手动拷贝**。
3. **保留裸机中断**：TIM5 软件 PWM 仍走 ISR（RTOS 下的 ISR 也能用，只要优先级 `≥ configMAX_SYSCALL_INTERRUPT_PRIORITY=5`）；若 ISR 要通知任务，用 `xSemaphoreGiveFromISR` + `portYIELD_FROM_ISR`（`ros_comm_task.c` 第 55–58 行已有现成写法）。
4. **风险提醒**：`HAL_Delay` 必须换成 `osDelay`；`printf` 是阻塞的，多任务下要迁移到独立口或加互斥。

## 7. 【面试扩展思路】若引入编码器 PID 会怎么改

1. **先修方向/计数**：按 `MOTOR_TEST.md` 第 8 节顺序，逐轮验证 A/B 相、计数符号、每转计数、丢脉冲，建立独立 `encoder direction`（代码里 `encoder.c` 第 52/73 行已有 `-read_encoder_count` 的符号试探痕迹）。
2. **启动 TIM6 采样**：`MX_TIM6_Init()` + `HAL_TIM_Base_Start_IT(&htim6)`，每 20 ms 在 `HAL_TIM_PeriodElapsedCallback` 里算四轮速度（`encoder.c` 48–93 行是现成实现）。
3. **单轮速度环先行**：`pid_init(&pid, 282, 85, 0, 100)` 的 Kp/Ki 是旧底盘的初值（`chassis_task.c` 71–77 行），不能直接照搬到新的车体——需要重新整定，且保留 PWM 限幅 + 失控停车。
4. **与当前 `car.c` 的冲突**：当前 `car_forward/backward/turn` 是**开环 PWM 直驱**；引入 PID 后应变成「上层给目标速度 `RcCommand.throttle/steering` → 中层换算目标轮速 → PID 输出 PWM」，即把 `motor_set_pwm` 从「命令入口」降级为「PID 输出执行器」，`rc_control.c` 的 `RcCommand` 正好是这层接口。

## 8. 一句话总结（面试口述）

> 「这个项目我刻意保持裸机前后台：主循环 10 ms 轮询 PS2，TIM5 和 UART5 两个中断做实时边沿和收字节。为什么不上 RTOS？因为这阶段任务就一条控制链，而且舵机有两路软件 PWM，靠 TIM5 中断的低延迟来保证脉宽，上 RTOS 反而可能因为临界区加大抖动。工程里其实带 FreeRTOS 源码和一个没调用的 `MX_FREERTOS_Init`，那是 CubeMX 骨架和历史底盘任务遗留，我把它当边界明确没启动。」
