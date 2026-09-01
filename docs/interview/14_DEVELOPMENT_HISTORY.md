# 14 开发历程（Development History）

> 从 `git log` 重建项目成熟过程。**诚实说明**：本仓库 git 历史是「整理导入式」的（大量 `Add files via upload`），没有 `fix:/bug:/refactor:` 粒度提交；真实的迭代轨迹藏在工程内 Markdown 开发笔记 + 备份目录名里。两部分都列出来，供面试准确表述。

---

## 1. Git 提交全览（10 个 commit）

`git log --format="%H|%ad|%s"` 结果（时间 +0800）：

| 顺序 | commit hash | 日期 | message | 说明 |
|---|---|---|---|---|
| 1 | `406121e` | 2026-08-11 15:35 | chore(repo): add repository hygiene rules | 仓库卫生规则 |
| 2 | `787b931` | 2026-08-11 15:35 | feat(car): import STM32 four-wheel controller | 导入小车固件 |
| 3 | `5a30999` | 2026-08-11 15:36 | feat(px4): add adhesion control overlays | 导入 PX4 覆盖层 |
| 4 | `80273b9` | 2026-08-11 15:36 | docs: summarize project status and source boundaries | 状态与边界文档 |
| 5 | `0580413` | 2026-08-11 15:37 | style(docs): normalize markdown and build log whitespace | 文档/日志格式化 |
| 6 | `f0e0945` | 2026-08-11 21:44 | Add files via upload | 上传（无细分说明） |
| 7 | `ae0e05d` | 2026-08-14 19:31 | Add files via upload | 上传（无细分说明） |
| 8 | `e3ec6a0` | 2026-08-17 10:34 | feat(car): update controller to current PS2 build | 更新到当前 PS2 版本（30/60/80 三挡 + ±90） |
| 9 | `ca50098` | 2026-08-17 10:36 | docs: add integrated flight-car development guide | 集成开发指南 |
| 10 | `6c3fc35` | 2026-08-17 10:36 | Merge pull request #1 | 合并 PR |

- **HEAD** = `6c3fc35`（merge commit）。
- **首个提交** `406121e`（2026-08-11 15:35），**末次提交** `6c3fc35`（2026-08-17 10:36），跨度约 6 天（公开仓库时间）。

## 2. 从工程内 Markdown 笔记重建的真实迭代（比 git 更细）

> 备份目录名暴露了真实的开发阶段（每条备份名都是「改动前快照」）：

| 阶段 | 备份/基线名（证据） | 内容 | 对应文档 |
|---|---|---|---|
| 原始底盘 | `差速底盘_5_11` | 差速底盘旧代码 | `PS2_BASIC_CONTROL.md` 第 5–8 行 |
| ① 车辆/遥控抽象层 | `差速底盘_5_11_before_car_rc_20260811_185303` | 新增 `car.c` + `rc_control.c`，主循环切安全轮询 | `CAR_CONTROL.md` |
| ② 飞控 UART 迁移 | `差速底盘_5_11_before_usart2_migration_20260812_151532` | USART2 迁到 `flight_comm`（11 字节 CRC8 + DMA） | `FLIGHT_UART.md` |
| ③ PS2 基础控制 | `差速底盘_5_11_before_ps2_basic_20260814_100956` | 最小「按键直行/停止」实车版 | `PS2_BASIC_CONTROL.md` |
| ④ 90° 普通转向 | `差速底盘_5_11_PS2_NORMAL_TURN90_20260814_143958` | 当前收录基线 | `SOURCE_MANIFEST.md` 第 16–20 行 |

## 3. 功能演进的「版本谱系」（从多份 PS2 笔记反推）

这些 `.md` 是不同测试阶段的快照，展示了「锁存 + 挡位 + 转向角」参数如何逐步收敛到当前版本：

| 版本快照 | 挡位 | 转向角 | 蟹行 | 证据 |
|---|---|---|---|---|
| PS2_BASIC_CONTROL | 固定 10% PWM | ±10° | 无 | `PS2_BASIC_CONTROL.md` 第 119–123 行 |
| PS2_BUTTON_TEST | 10/20/30%（20 起） | 无转向 | 无 | `PS2_BUTTON_TEST.md` 第 15–16 行 |
| PS2_BASIC_DELIVERY | 20/40/70% | ±45° | 无 | `PS2_BASIC_DELIVERY.md` 第 7–14 行 |
| PS2_CRAB_MODE | 20/40/70%（蟹行限 20%） | ±45° | 有 | `PS2_CRAB_MODE.md` |
| **PS2_NORMAL_TURN90（当前）** | **30/60/80%** | **±90°** | **已取消** | `PS2_NORMAL_TURN90.md` 第 5 行 |

> 当前发布版本对应 `car/releases/PS2_NORMAL_TURN90_SPEED30_60_80/`，即「±90° 普通转向 + 30/60/80 三挡、取消蟹行」。

## 4. 成熟过程小结（面试可讲）

1. **阶段一（8-11）**：把历史差速底盘改造成「分层车辆 API + 遥控抽象」，主循环从测试循环切到安全轮询（`CAR_CONTROL.md`）。
2. **阶段二（8-12）**：把飞控通信迁到 USART2 的 `flight_comm`，引入 CRC8 + DMA + 500 ms 超时，并加 `FLIGHT_COMM_ENABLE_CAR_OUTPUT` 安全门（`FLIGHT_UART.md`）。
3. **阶段三（8-14 起）**：PS2 从「最小直行/停止」逐步加到「锁存动作 + 三挡调速 + 普通转向」，中途试过蟹行模式后又在最终版取消（`PS2_CRAB_MODE.md` → `PS2_NORMAL_TURN90.md`）。
4. **阶段四（8-17）**：把当前 PS2 版本、PX4 覆盖层、技术文档整合进仓库，形成可复现基线（`e3ec6a0` + `ca50098`）。

## 5. 面试表述建议（诚实口径）

> 「仓库的 git 历史是整理导入式的，公开提交里没有逐条 fix/refactor 的粒度——因为原始开发在本地目录里用『改动前备份』管理。所以我能给到的成熟过程，是从仓库里那几份带日期的开发笔记和备份名反推的：先分层抽象、再飞控 UART、再 PS2 逐步加功能、最后取消蟹行收敛到当前 30/60/80 + ±90 版本。这个过程的『证据』是 Markdown 笔记 + 构建日志 + HEX SHA256，不是一条条 commit message。」
