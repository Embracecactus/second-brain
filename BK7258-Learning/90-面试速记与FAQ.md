# 90. 面试速记与 FAQ

> 目标：把整套 BK7258 知识浓缩成"面试能张口就来"的速记卡和问答集。适合面试前 30 分钟快速过一遍。

---

## 90.1 怎么用这篇文档

| 场景 | 用法 |
|---|---|
| 面试前 30 分钟 | 只看 90.2 速记卡，把数字和概念串一遍 |
| 被追问细节 | 从 90.3 的 FAQ 找对应答案，再回对应模块看展开 |
| 自我介绍 | 用 90.4 的"电梯陈述"框架组织 |

速记原则：**先讲一句话结论，再讲一个图，再讲一个数字**。面试官要的是"你懂"，不是"你背得熟"。

---

## 90.2 面试速记卡

### 芯片与架构

| 速记点 | 一句话答案 |
|---|---|
| BK7258 是什么 | 博通集成（Beken）Wi-Fi 6 + BLE 5.4 Combo SoC，模组为涂鸦 T5-AI |
| 核心架构 | 3× ARM Cortex-M33：CPU0=CP（控制面，单核）+ CPU1/2=AP（应用面，SMP 双核） |
| 主频 | DPLL 最高 ~480MHz，产品路径用 320MHz DVFS |
| 存储 | 片上 Flash ~8MiB（逻辑基址 0x02000000）+ 外接 PSRAM 16MiB |
| 软件栈 | BootROM → BL1 → MCUboot(BL2) → NuttX(CP 单核 + AP SMP) |

### 启动链（一句话图）

```
BootROM → BL1(XIP@0x02000000) → [BL2(SRAM@0x28020000)] → CP(XIP@0x02010000) → AP(XIP@0x02150000)
```

- BL1：自研 Tier-1，上电最先跑，负责时钟标定、FAL 分区解析、引导选择。
- BL2：MCUboot 外壳，负责验签、选槽、跳转（可选环节）。
- CP：NuttX 单核，跑控制面。
- AP：由 CP 释放，NuttX SMP 双核，跑应用面。

### 关键地址速记

| 项 | 地址 |
|---|---|
| BL1 XIP / MSP | 0x02000000 / 0x2809F700 |
| CP XIP | 0x02010000 |
| AP XIP | 0x02150000 |
| BL2 primary / secondary XIP | 0x024d0000 / 0x024f0000 |
| BL2 SRAM 运行区 | 0x28020000..0x2802ffff |
| CP RAM 下界（MSPLIM） | 0x28010000 |
| BL1→BL2 策略记录 | 0x2801ffd0 |

### 分区（数字版）

| 分区 | 物理偏移 / 大小 |
|---|---|
| BL1 (primary_bootloader) | 0x000000 / 0x11000 |
| CP (primary_cp_app) | 0x011000 / 0x154000 |
| AP (primary_ap_app) | 0x165000 / 0x121000 |
| s_app (slot B 暂存) | 0x286000 / 0x275000 |
| BL2 | 0x51d000 / 0x22000 |
| littlefs | 0x600000 / 0x100000 |

### 安全启动（一句话）

MCUboot 用 **ECDSA-P256 验签 + 硬件回滚保护**；BL1 侧有板级 Manifest（开发根签名），当前 `BL1_MANIFEST_ENFORCE=0`，OTP/eFuse 真安全启动未启用。

### 驱动范式（一句话）

**零寄存器访问**：NuttX lower-half 封装 armino SDK 的 `bk_*` HAL，不直接读写寄存器。

### 跨核通信（一句话）

CP 与 AP 通过 **RPTUN/RPMsg**（基于 mailbox 硬件）通信。

---

## 90.3 高频面试题 FAQ

### Q1：你做过什么项目？怎么介绍？

**参考回答**：把 openvela/NuttX 移植到 BK7258（3 核 M33 的 Wi-Fi/BLE SoC）。我负责 board overlay：自研 BL1 二级引导、MCUboot BL2 板级适配、CP/AP 双域 NuttX 构建与 11 个外设驱动移植，并用 RPTUN/RPMsg 打通核间通信。过程中沉淀了一套"配置断言 + 幂等核实 + 状态门闩"的驱动质量方法，修复了多个真实并发/生命周期 bug。

**要点**：数字（11 个驱动、3 核、320MHz）+ 方法（不是只会调库）。

### Q2：为什么需要 BL1 和 BL2 两级引导？

**答案**：芯片上电只保证 BootROM 在固定地址跑。BootROM 能力有限，BL1 做最必要的"脏活"——时钟标定、Flash 初始化、分区解析，然后跳 BL2 做"安全活"——验签、选槽、防回滚。**分级的本质是"职责分离 + 每级只做最少事"**：一级坏了另一级还能兜底，也符合安全启动的"信任链从根逐步建立"思想。

### Q3：什么是 XIP？为什么不用 RAM？

**答案**：XIP = Execute In Place，代码直接在 Flash 上执行，不拷贝到 RAM。优点：省 RAM、启动快。代价：Flash 读取比 RAM 慢，所以**初始化代码和热路径才放 RAM**（如 BL2 拷到 SRAM 跑），大镜像放 Flash XIP。

### Q4：CP 和 AP 怎么通信？

**答案**：硬件上用 mailbox 邮箱，软件上跑 RPTUN/RPMsg 协议（见 [25-跨核通信-RPTUN](./25-跨核通信-RPTUN.md)）。RPMsg 是"结构化消息"，两边各自定义通道和回调，一方发、一方收，屏蔽了底层 mailbox 细节。

### Q5：怎么排查"上电没反应"？

**答案**（按顺序）：
1. 查 console 日志（UART1，460800）——看到哪一步停的。
2. 查电源/复位是否正常。
3. J-Link SWD 连上，看 PC 停在哪（是卡 BL1、BL2 验签失败、还是 CP panic）。
4. 验签失败就看镜像签名/密钥是否匹配。
5. 用稀疏烧录对比，确认烧进去的是不是预期的固件。

### Q6：你写一个驱动，步骤是什么？

**答案**：
1. 确认 SDK 的 `bk_*` HAL 有哪些函数（先核实幂等性）。
2. 定义 NuttX 的 ops 结构体（如 `i2c_ops_s`）。
3. 实现 setup/transfer/shutdown 等函数，内部只调 `bk_*`，不碰寄存器。
4. 定义私有数据结构，绑定 ops 和硬件单元。
5. 调 NuttX 注册函数（如 `i2c_register`），在 `board_initialize` 里启用。
6. 加 CONFIG 断言和 running 门闩，防配置缺失和并发重入。

### Q7：安全启动怎么做防回滚？

**答案**：镜像带安全计数（security counter），MCUboot 启用 `MCUBOOT_HW_ROLLBACK_PROT`，每次升级计数单调递增，固件带旧计数就被拒。作用：防止攻击者把系统"降级"回有漏洞的旧版本。

### Q8：你踩过最深的坑是什么？

**参考回答**（选一个真实修复过的）：
- **坑一**：线程退出后资源被释放，导致野指针——修法是 close 路径做 pthread_join 集合。
- **坑二**：ISR 里反复读状态寄存器读到撕裂值——修法是入口一次性快照。
- **坑三**：驱动被重复 start 导致竞态——修法加原子 running 门闩。
- **坑四**：静态库缺符号，误判 SDK 没实现——实际是 CONFIG 父开关被裁剪，先查配置链。

**讲法**：现象 → 根因 → 修法 → 如何防止再犯（沉淀成 checklist）。

### Q9：NuttX 和 Linux 驱动模型有什么不同？

**答案**：都有"upper-half/lower-half"分层的思想。区别：NuttX 更轻（无 mmap 那些重型抽象）、单内核地址空间、很多驱动是"轮询/简单中断"风格；Linux 有设备树、复杂电源管理、模块化加载。但 **ops 结构体 + 注册函数**这套心智模型是相通的。

---

## 90.4 一分钟电梯陈述

> "我负责把 NuttX 跑上 BK7258——一颗 3 核 Cortex-M33 的 Wi-Fi/BLE 芯片。核心工作三条线：**引导链**（自研 BL1 + MCUboot BL2，支持验签和双槽升级）、**系统移植**（CP 单核 + AP SMP 双域，核间 RPTUN/RPMsg）、**驱动**（11 个外设驱动，'零寄存器访问'范式封装 Beken SDK）。过程中我把'驱动质量'做成方法论——配置断言、幂等核实、状态门闩——并实际修复了多个并发和生命周期 bug。"

---

## 90.5 考前 3 分钟

1. 默写启动链：`BootROM → BL1(0x02000000) → BL2(0x28020000) → CP(0x02010000) → AP(0x02150000)`。
2. 默写驱动三步：`ops 结构体 → 私有数据 → 注册函数`。
3. 默写通信一句话：`RPTUN/RPMsg 基于 mailbox`。
4. 默写安全一句话：`ECDSA-P256 + 回滚保护`。
5. 记住你修复过的 2 个坑，能讲出"现象→根因→修法"。

---

## 底部导航

←上一篇：[30 构建-烧录-调试实战](./30-构建-烧录-调试实战.md) · ↑返回导航：[00 开始这里](./00-开始这里-导航与学习路径.md)

---

📂 **本文涉及源码路径**

- `/home/lijian/project/open-vela/contest2026_135_yongwangzhiqian/board/bk7258/bootloader/`（BL1）
- `/home/lijian/project/open-vela/contest2026_135_yongwangzhiqian/board/bk7258/bootloader/bl2/`（BL2/MCUboot）
- `/home/lijian/project/open-vela/contest2026_135_yongwangzhiqian/board/bk7258/chip/ap/`（AP 驱动）
- `/home/lijian/project/open-vela/contest2026_135_yongwangzhiqian/board/bk7258/chip/cp/`（CP 驱动）
- 各概念展开请回链：[00 导航](./00-开始这里-导航与学习路径.md)
