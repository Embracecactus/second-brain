---
tags:
  - embedded-linux
  - bsp
  - bootloader
  - u-boot
  - board-porting
  - driver-model
  - rockchip
  - rv1126b
category: embedded-linux
created: 2026-06-23
updated: 2026-06-23
status: active
soc: Rockchip RV1126B
kernel: Linux 6.1.141
parent: bsp-boot-flow
---

# U-Boot 板级适配与启动流程

> **前置笔记**：[[bsp-boot-flow]] — 阶段一主笔记 (Boot Chain, FIT, 分区表)
> **深度分析**：[[bsp-spl-fit]] — SPL FIT 镜像解析与验证源码追踪
>
> 本文聚焦 U-Boot 源码层的板级适配架构，涵盖 SoC 级代码、板级代码、Driver Model 初始化、DTS `u-boot,dm-spl` 标记、Kconfig 选板链路、以及适配新板子的完整 checklist。

---

## 一、U-Boot 启动流程全景

### 1.1 RV1126B 的三阶段启动

```
┌──────────────────────────────────────────────────────────────────┐
│  阶段                 固件                       开发者可控?      │
├──────────────────────────────────────────────────────────────────┤
│  TPL (DDR Init)  rv1126b_spl_loader_v1.09.105.bin    ✗ 闭源    │
│  SPL             u-boot-spl.bin (从源码编译)         ✓ 开源    │
│  U-Boot proper   u-boot-nodtb.bin (从源码编译)       ✓ 开源    │
│  ATF BL31        rv1126b_bl31_v1.13.elf              ✗ 闭源    │
│  OP-TEE BL32     rv1126b_bl32_v1.04.bin              ✗ 闭源    │
└──────────────────────────────────────────────────────────────────┘
```

> **关键差异**：RV1126 (Cortex-A7, ARM32) 的 TPL 包含 DDR 训练代码，从源码编译。RV1126B (Cortex-A53, ARM64) 的 TPL 改用预编译二进制，DDR 训练代码不开放。

### 1.2 SPL 完整初始化时序 (board_init_f)

> **源码**：`arch/arm/mach-rockchip/spl.c:204-266`
>
> 这是 SPL 启动的关键路径，每个函数调用都不可省略。

```c
void board_init_f(ulong dummy)               // spl.c:204
{
    gd->flags = dummy;                       // L212: 保存启动标志
    
    rockchip_stimer_init();                  // L213: 安全定时器
    //   读取 CONFIG_ROCKCHIP_STIMER_BASE (0x20820000)
    //   如果定时器未使能 → 初始化并使能
    
    debug_uart_init();                       // L226: 早期 UART
    //   基于 DTS stdout-path 配置 UART0
    //   输出 "U-Boot SPL board init" 到串口
    //   此时还没有控制台框架，只能用 printascii()
    
    gd->sys_start_tick = get_ticks();        // L229: 记录启动时间戳
    
    spl_early_init();                        // L234: DM 早期初始化
    //   → dm_init_and_scan(true)
    //   → 遍历 DTB 中所有 u-boot,dm-pre-reloc 标记的设备
    //   → 绑定(Bind) 但不探活(Probe)
    //   失败 → hang()
    
    uclass_get_device(UCLASS_RAM, 0, &dev); // L241: 确认 DDR 就绪
    //   → 查找 UCLASS_RAM 类设备 (sdram_rv1126b)
    //   → 对于 RV1126B，DDR 已由 TPL 初始化
    //   → 此调用仅确认 DDR 控制器状态正常
    //   失败 → printf + return
    
    preloader_console_init();                // L247: 控制台就绪
    //   → 基于 CONFIG_BAUDRATE (1500000) 重配 UART
    //   → 此后可以使用 printf()
    
    spl_hotkey_init();                       // L253: 检测热键
    //   → 读取 ADC 按键值 (GPIO/ADC)
    //   → ctrl+c: 标记进入 USB 下载模式
    //   → ctrl+z: 标记进入 maskrom 模式
    
    arch_cpu_init();                         // L257: ★ SoC 级初始化
    //   → rv1126b.c:417 (773行的核心函数)
    //   ├─ 安全防火墙配置 (SGRF)
    //   ├─ USB PHY 初始化
    //   ├─ TSADC 温度传感器使能
    //   ├─ PVTPLL (ISP/ENC/AIISP) 使能
    //   └─ board_set_iomux() 启动设备引脚复用
    
    rk_board_init_f();                       // L258: 板级回调 (weak)
    //   通常为空，板级特定初始化可在此实现
    
    // L262-264: 如果配置了 back_to_brom，返回 BootROM
}
```

**函数调用** → **源码位置** → **作用速查表**：

| 调用顺序 | 函数 | 源码位置 | 核心作用 | 失败后果 |
|---------|------|---------|---------|---------|
| 1 | `rockchip_stimer_init()` | spl.c:213 | 安全定时器使能 | 计时不可用 |
| 2 | `debug_uart_init()` | spl.c:226 | 早期串口输出 | 看不到日志 |
| 3 | `spl_early_init()` | spl.c:234 | DM 预绑定 | **hang()** |
| 4 | `uclass_get_device(RAM)` | spl.c:241 | 确认 DDR | return |
| 5 | `preloader_console_init()` | spl.c:247 | printf 可用 | 无标准输出 |
| 6 | `spl_hotkey_init()` | spl.c:253 | USB 下载快捷键 | 只能正常启动 |
| 7 | `arch_cpu_init()` | rv1126b.c:417 | **安全+外设初始化** | 外设异常 |
| 8 | `rk_board_init_f()` | spl.c:258 | 板级回调 (weak) | 可忽略 |

### 1.3 SPL board_init_r — FIT 加载与跳转

> **源码**：`common/spl/spl.c:530-620`

`board_init_f()` 执行完毕后，SPL 框架调用 `board_init_r()`：

```
[SPL board_init_r()]         common/spl/spl.c:530
  │
  ├─ spl_initr_dm()                       L535: DM 完整扫描
  │   → dm_init_and_scan(false)
  │   → 探活所有 u-boot,dm-spl 标记的设备
  │   → CRU (时钟) / GRF (寄存器) / eMMC / UART ...
  │
  ├─ timer_init()                         L562: 定时器初始化
  │
  ├─ spl_board_init()                     L566: 板级 SPL 初始化
  │   → rv1126b(spl.c:357):
  │     ├─ setup_led()                    LED 指示
  │     └─ rk_spl_board_init()            板级回调
  │
  ├─ spl_image 初始化                      L569-593:
  │   ├─ entry_point_bl32 = -1            OP-TEE 入口 (无效)
  │   └─ entry_point_bl33 = SYS_TEXT_BASE U-Boot 入口 (默认值)
  │
  ├─ board_boot_order(spl_boot_list)       L594: 获取启动设备列表
  │   → 从 DTS chosen/u-boot,spl-boot-order 读取
  │
  ├─ boot_from_devices()                   L595: ★ 核心启动循环
  │   │
  │   ├─ 遍历启动设备 (eMMC → SD → SPI → USB)
  │   ├─ 每个设备:
  │   │   ├─ spl_load_image()             设备特定的 loader
  │   │   │   └─ spl_load_simple_fit()    ★ FIT 镜像处理
  │   │   │       ├─ 配置层 RSA 验签
  │   │   │       ├─ 加载 firmware (atf-1 → 0x40000000)
  │   │   │       ├─ 追加 FDT
  │   │   │       └─ 遍历 loadables
  │   │   │           ├─ uboot  → 0x40200000 (BL33 入口)
  │   │   │           ├─ atf-2  → System SRAM
  │   │   │           ├─ atf-3  → PMU SRAM
  │   │   │           └─ atf-4  → System SRAM
  │   │   └─ 成功 → break
  │   └─ 全部失败:
  │       └─ spl_ab_decrease_reset()       A/B 降级
  │           └─ tries_remaining-- → do_reset()
  │
  ├─ spl_perform_fixups(&spl_image)        L601: FDT 修正
  │
  └─ jump_to_image_no_args(&spl_image)     L615: ★ 跳转到下一阶段
      └─ 跳转到 spl_image.entry_point (= 0x40000000, ATF BL31)
```

> **详细 FIT 处理流程**：参见 [[bsp-spl-fit]] (515行深度分析)

### 1.4 ATF BL31 → U-Boot proper 过渡

```
[ATF BL31] 0x40000000
  │
  ├─ EL3 初始化
  │   ├─ GIC-400 中断控制器配置
  │   ├─ MMU/页表设置
  │   ├─ SMC 调用处理注册
  │   └─ PSCI (电源管理接口) 初始化
  │
  ├─ 返回 EL2 (Hypervisor 模式)
  │   └─ 跳转到 spl_image.entry_point_bl33
  │       → 0x40200000 (U-Boot proper)
  │
  ▼
[U-Boot proper] 0x40200000
  │
  ├─ _start (arch/arm/lib/vectors.S)
  │   → 设置异常向量表
  │   → lowlevel_init (SMP 初始化)
  │   → _main (arch/arm/lib/crt0_64.S)
  │
  ├─ board_init_f()                    通用板级初始化框架
  │   ├─ init_sequence_f[]             遍历初始化函数列表
  │   │   ├─ fdtdec_setup()            设置 FDT
  │   │   ├─ initf_malloc()            早期 malloc
  │   │   ├─ arch_cpu_init()           架构初始化
  │   │   ├─ initf_dm()                DM 预重定位绑定
  │   │   ├─ serial_init()             串口 (正式)
  │   │   ├─ dram_init()               DDR 容量检测
  │   │   └─ ...
  │   └─ relocate_code()               ★ U-Boot 自身重定位
  │       → 将 U-Boot 从 0x40200000 搬到 DDR 高端
  │
  ├─ board_init_r()                    重定位后的初始化
  │   ├─ init_sequence_r[]             遍历初始化函数列表
  │   │   ├─ initr_dm()                DM 完整扫描
  │   │   ├─ initr_mmc()               eMMC/SD 驱动
  │   │   ├─ board_init()              ★ 板级初始化
  │   │   │   → rv1126b(board.c:560):
  │   │   │     ├─ smp_event1(SEVT_0)  SMP 同步
  │   │   │     ├─ board_debug_init()  UART 调试
  │   │   │     ├─ early_download()    按键进入下载
  │   │   │     └─ rk_board_init()    Rockchip 板级
  │   │   ├─ initr_net()               网卡初始化
  │   │   ├─ board_late_init()         晚期板级初始化
  │   │   │   → board.c:483:
  │   │   │     ├─ rockchip_set_ethaddr()      设置 MAC 地址
  │   │   │     ├─ rockchip_set_serialno()     设置序列号
  │   │   │     ├─ setup_download_mode()       下载模式标志
  │   │   │     ├─ rockchip_show_logo()        显示启动 Logo
  │   │   │     ├─ setup_boot_mode()           设置启动模式
  │   │   │     ├─ soc_clk_dump()              打印时钟树
  │   │   │     ├─ amp_cpus_on()               AMP 核心上电
  │   │   │     └─ rk_board_late_init()        板级回调
  │   │   ├─ run_main_loop()           ★ 进入主循环
  │   │   │   ├─ cli_init()            命令行接口
  │   │   │   ├─ autoboot_command()    自动启动
  │   │   │   │   └─ bootm (加载 Kernel FIT)
  │   │   │   └─ cli_loop()            U-Boot shell
  │   │   └─ ...
  │   │
  │   └─ bootm / booti → Linux Kernel
  │
  ▼
[Linux Kernel] boot.img (FIT: Image + DTB + resource)
```

### 1.5 完整启动时间线 (自上电到 Kernel)

```
时间轴 →

0ms     [上电]
         │
~50ms   [BootROM] 芯片内置 ROM 代码
         │  → 从 eMMC 读取 MiniLoaderAll.bin
         │
~150ms  [TPL/DDR Init] 闭源二进制
         │  → DDR 初始化 (1332MHz)
         │  → SPL 加载
         │  串口输出: (无)
         │
~160ms  [SPL board_init_f]  arch/arm/mach-rockchip/spl.c
         │  → 安全定时器 → 早期 UART → DM 绑定 → 确认 DDR
         │  串口输出: "U-Boot SPL board init"
         │
~170ms  [SPL board_init_r]  common/spl/spl.c
         │  → DM 完整扫描 (CRU/eMMC/UART/PMIC)
         │  → board_boot_order()
         │  串口输出: "Trying fit image at 0x4000 sector"
         │
~180ms  [SPL FIT 加载]
         │  → 配置层 RSA 验签
         │  → 加载 atf-1 (LZMA 解压)
         │  → 加载 uboot + atf-2/3/4 + fdt
         │  串口输出: "## Checking atf-1 ... sha256+ OK"
         │            "## Checking uboot ... sha256+ OK"
         │            ...
         │
~230ms  [跳转到 ATF BL31]  0x40000000
         │  → EL3 初始化 (GIC/MMU/PSCI)
         │  串口输出: (无)
         │
~250ms  [ATF → U-Boot]  0x40200000
         │  → U-Boot SPL 串口: "U-Boot SPL 2017.09-..."
         │
~300ms  [U-Boot board_init_f]
         │  → DM 预绑定 → 重定位
         │  串口输出: "U-Boot 2017.09-..."
         │
~400ms  [U-Boot board_init_r]
         │  → DM 完整扫描 → eMMC → 网络 → 显示 Logo
         │  串口输出: "Model: ALIENTEK RV1126B board"
         │
~500ms  [autoboot 倒计时]
         │  串口输出: "Hit any key to stop autoboot: 0"
         │
~600ms  [bootm → Kernel boot.img]
         │  → 从 boot 分区 (mmcblk0p3) 加载 boot.img
         │  → 解析 FIT → 验签 → 解压内核
         │  串口输出: "## Flattened Device Tree blob at ..."
         │            "Starting kernel ..."
         │
~700ms  [Kernel 解压]  自解压 → 内核入口
         │  串口输出: "Booting Linux on physical CPU 0x0"
         │            "Linux version 6.1.141 ..."
         │
~3s     [Kernel 启动完成]
         │  → 挂载 rootfs → systemd/systemV init
         │  串口输出: "Freeing unused kernel memory"
         │
~5s     [用户空间就绪]
         │  串口输出: "Welcome to Ubuntu 22.04"
         │            "sportcam login:"
```

> **计时说明**：以上时间为基于 RV1126B 典型配置的估计值，实际时间取决于 eMMC 速度、内核配置等。可通过 `CONFIG_BOOTSTAGE` + `bootstage report` 在板端精确测量。

### 1.6 各阶段串口日志特征 — 完整定位对照表

> 以下日志来自实际源码中的 `printf()`/`printascii()` 调用。
> 每条日志后标注 **源码文件:行号**，可直接跳转验证。

#### 阶段 0: BootROM + TPL/DDR Init (无输出)

```
[无任何串口输出]
```

| 现象 | 说明 |
|------|------|
| 上电后约 150ms 无输出 | BootROM 和 TPL DDR 初始化均为闭源固件，不输出日志 |
| 如果此阶段超过 500ms 仍无输出 | 可能 DDR 颗粒不匹配或 eMMC 未正确焊接 |

**grep 定位**：无 — 此阶段无可用于 grep 的日志特征。

---

#### 阶段 1: SPL board_init_f — 第一条日志

```
U-Boot SPL board init                           ← ★ 上电后第一条输出
```

| 日志 | 源码位置 | 输出方式 | 含义 |
|------|---------|---------|------|
| `U-Boot SPL board init` | `spl.c:227` | `printascii()` | **第一条输出**，标志 SPL 代码已开始执行 |
| (无输出, 正常时) | `spl.c:234-246` | — | `spl_early_init()` + DDR 确认 |
| `spl_early_init() failed: N` | `spl.c:236` | `printf()` | ⚠️ DM 初始化失败 → `hang()` |
| `DRAM init failed: N` | `spl.c:243` | `printf()` | ⚠️ DDR 未就绪 → `return` (继续尝试) |
| `ctrl+b: Bootrom download!` | `spl.c:173` | `printf()` | 🔧 用户触发了 USB 下载模式 |
| `SPL Hotkey: ctrl+X` | `spl.c:197` | `printf()` | 🔧 检测到热键组合 |

**grep/sed 分界点**：

```bash
# 检测 SPL 是否启动
grep -m1 "U-Boot SPL board init" boot.log

# 检测 SPL 初始化是否失败
grep -E "spl_early_init.*failed|DRAM init failed" boot.log
```

**正常特征**：`U-Boot SPL board init` 出现后 10ms 内进入下一阶段（FIT 加载），中间无其他日志输出。

---

#### 阶段 2: SPL board_init_r — FIT 加载与验签

```
Trying fit image at 0x4000 sector                 ← FIT 加载开始
## Checking atf-1 0x40000000 (lzma @0x40200000) ... sha256+ OK
## Checking uboot 0x40200000 (lzma @0x42200000) ... sha256+ OK
## Checking atf-2 0x3ffbb000 ... sha256+ OK
## Checking atf-3 0x3ff1e000 ... sha256+ OK
## Checking atf-4 0x3ffbd000 ... sha256+ OK
## Checking fdt 0x41000000 ... sha256+ OK
```

| 日志 | 源码位置 | 含义 |
|------|---------|------|
| `Trying fit image at 0x4000 sector` | `spl_fit.c:952` | **FIT 加载入口**，扇区号 = FIT 在 eMMC 上的位置 |
| `Trying fit image at 0x8000 sector` | `spl_fit.c:957` | A/B 容错：尝试第二份 FIT 镜像 |
| `IO error` | `spl_fit.c:959` | ⚠️ 读取扇区失败（eMMC 硬件故障） |
| `Not fit magic` | `spl_fit.c:965` | ⚠️ FIT Header 校验失败（文件损坏/不是 FIT） |
| `## Checking {name} 0x{addr} ({comp} @0x{src}) ...` | `spl_fit.c:354` | 压缩段验证（atf-1, uboot） |
| `## Checking {name} 0x{addr} ...` | `spl_fit.c:359` | 无压缩段验证（atf-2/3/4, fdt） |
| `sha256` / `sha256+` | `image-fit.c:1364` / 代码无显式 | SHA256 hash 校验通过 |
| `sha256 error!` | `image-fit.c:1466` | ⚠️ **hash 校验失败** (对应段的数据损坏) |
| `+ ` | 验签代码 | RSA 签名校验通过 |
| `- ` | 验签代码 | RSA 签名校验失败 (non-required, 不阻断) |
| `OK` | `spl_fit.c:385` | **段加载完成** (验证+解压+搬运成功) |
| `rollback index: N >= N(min), OK` | `spl_fit.c:780` | 防降级校验通过 |
| `fit reject rollback: N < N(min)` | `spl_fit.c:775` | ⚠️ 固件版本过旧，拒绝启动 |
| `No default config node` | `spl_fit.c:755` | ⚠️ FIT 中无 configurations 节点 |
| `fit verify configure failed, ret=N` | `spl_fit.c:761` | ⚠️ **配置层 RSA 验签失败** |
| `Decrypting Data ... OK` | `spl_fit.c:372,376` | AES 加密段解密（仅 CONFIG_SPL_FIT_CIPHER 时） |

**每条 `## Checking` 日志的格式解读**：

```
## Checking atf-1 0x40000000 (lzma @0x40200000) ... sha256+ OK
  │         │     │           │       │              │      │
  │         │     │           │       │              │      └─ 加载成功
  │         │     │           │       │              └─ hash 通过
  │         │     │           │       └─ 压缩数据临时地址
  │         │     │           └─ 压缩算法 (lzma/none)
  │         │     └─ 目标加载地址
  │         └─ FIT 节点名
  └─ 固定前缀
```

**grep/sed 定位**：

```bash
# 统计加载段数
grep -c "## Checking" boot.log

# 检查哪个段验证失败
grep -B1 "error!" boot.log

# 检查 FIT magic 校验
grep "Not fit magic" boot.log

# 检查 A/B 容错（多份 FIT 尝试）
grep "Trying fit image at" boot.log | wc -l

# 检查配置层签名状态
grep -E "fit verify configure|rollback index" boot.log
```

**正常特征**：`Trying fit image` → 6 条 `## Checking` 行 → 每条以 `OK` 结尾。

---

#### 阶段 3: ATF BL31 (无输出)

```
[无任何串口输出，约 20ms]
```

| 现象 | 说明 |
|------|------|
| FIT 加载的最后一条 `OK` 后无输出 | ATF BL31 接管 EL3，UART 未初始化 |
| 如果此阶段超过 100ms 仍无输出 | 可能 ATF 崩溃或 entry point 错误 |

**grep 定位**：无直接日志。用最后一条 `OK` 到下一条 `U-Boot` 之间的间隔来判断。

---

#### 阶段 4: U-Boot proper 初始化

```
U-Boot 2017.09-gxxxxxxxx-dirty (Jun 22 2026 - 17:00:00 +0800)   ← 版本号

Model: ALIENTEK RV1126B board                                     ← 板级识别
DRAM:  512 MiB (512 MiB max)                                       ← DDR 容量
MMC:   dwmmc@20300000: 0, dwmmc@20310000: 1                        ← eMMC/SD
Using default environment                                         ← 环境变量
In:    serial                                                      ← stdin
Out:   serial                                                      ← stdout
Err:   serial                                                      ← stderr
Net:   eth0: ethernet@215c0000                                     ← 网络

Hit any key to stop autoboot:  0                                   ← autoboot 倒计时
```

| 日志 | 源码/来源 | 含义 |
|------|---------|------|
| `U-Boot 2017.09-g...` | `cmd/version.c` | **U-Boot 版本标识**，标志 U-Boot proper 已启动 |
| `Model: ALIENTEK RV1126B board` | DTS `model` 属性 | 板级识别，来自 `rv1126b-alientek.dts` |
| `DRAM:  XXX MiB` | `dram_init()` | 检测到的 DDR 总容量 |
| `MMC:   dwmmc@...` | `initr_mmc()` | eMMC/SD 控制器初始化 |
| `Hit any key to stop autoboot:` | `autoboot_command()` | 🔧 按任意键进入 U-Boot shell |

**grep/sed 定位**：

```bash
# 定位 U-Boot 版本行（阶段分界标记）
grep "^U-Boot 20" boot.log

# 确认板子型号
grep "^Model:" boot.log

# 确认 DDR 容量
grep "^DRAM:" boot.log

# 检查是否进入 shell
grep "Hit any key\|=>" boot.log
```

**正常特征**：版本号 → 板级信息 → autoboot 倒计时 → 3秒后自动启动 Kernel。

---

#### 阶段 5: U-Boot → Kernel 过渡

```
## Loading kernel from FIT Image at 0x... ...
## Flattened Device Tree blob at 0x...
   Booting using the fdt blob at 0x...
   Loading Device Tree to 0x..., end 0x...
## Flattened Device Tree(FDT) from separate resource ...
## Starting kernel ...                          ← ★ 跳转点

[ATF BL31 安全监控，无输出]

Booting Linux on physical CPU 0x0000000000 [0x410fd034]   ← ★ Kernel 第一条输出
Linux version 6.1.141 (user@host) (gcc version 10.3.1 ...) ← 内核版本信息
```

| 日志 | 源码/来源 | 含义 |
|------|---------|------|
| `## Loading kernel from FIT Image ...` | `common/image-fit.c` | U-Boot 解析 Kernel FIT |
| `## Flattened Device Tree blob at ...` | `common/image-fdt.c` | FDT 加载地址 |
| `## Starting kernel ...` | `arch/arm/lib/bootm.c` | **跳转到内核入口**（最后一条 U-Boot 日志） |
| `Booting Linux on physical CPU 0x0` | `arch/arm64/kernel/head.S` | **Kernel 第一条输出**（CPU0 启动） |
| `Linux version 6.1.141` | `init/version.c` | 内核版本标识 |

**grep/sed 定位**：

```bash
# U-Boot → Kernel 切换点
grep "Starting kernel" boot.log

# Kernel 第一条输出 (确认内核已启动)
grep -m1 "Booting Linux" boot.log

# 内核版本
grep -m1 "Linux version" boot.log

# 测量 U-Boot 启动耗时 (版本号 → 启动内核)
grep -E "^U-Boot|Starting kernel" boot.log
```

---

#### 阶段 6: Kernel 启动

```
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd034]
[    0.000000] Linux version 6.1.141 ...
[    0.000000] Machine model: ALIENTEK RV1126B (SportCam)
[    0.000000] earlycon: uart8250 at MMIO32 0x20810000 (options '')
[    0.000000] printk: bootconsole [uart8250] enabled
[    0.000000] Memory policy: Data cache writealloc
[    0.000000] Reserved memory: created CMA memory pool at 0x..., size 128 MiB
[    0.000000] OF: reserved mem: initialized node ... 
...
[    1.234567] Freeing unused kernel memory: 2048K           ← Kernel 启动完成
[    1.345678] Run /sbin/init as init process                 ← 启动 init
```

| 日志 | 源码位置 | 含义 |
|------|---------|------|
| `Booting Linux on physical CPU 0x%x` | `arch/arm64/kernel/head.S` | **CPU0 入口** |
| `Linux version %s` | `init/version.c` | 内核版本信息 |
| `Machine model: %s` | `arch/arm64/kernel/setup.c` | DTS model 属性 |
| `earlycon: uart8250 at MMIO32 0x%x` | `drivers/tty/serial/earlycon.c` | 早期串口使能 |
| `Freeing unused kernel memory: %dK` | `init/main.c` | **内核启动完成的标志** |
| `Run /sbin/init as init process` | `init/main.c` | 用户空间 init 启动 |

**grep/sed 定位**：

```bash
# 内核早期启动 (第一条有意义输出)
grep -m1 "Booting Linux\|Machine model" boot.log

# 内核启动完成
grep -m1 "Freeing unused kernel memory" boot.log

# 全部内核日志 (dmesg 格式)
grep "^\[" boot.log | head -20
```

---

#### 阶段 7: 用户空间 Init 与服务启动

> **Init 系统**：BusyBox init (非 systemd)
> **主控台**：`ttyFIQ0` (Rockchip FIQ Debugger, 波特率 1.5Mbps)
> **服务启动**：`/etc/init.d/rcS` → 按 S?? 前缀数字顺序遍历 `/etc/init.d/S??*`

##### 7.1 Init 启动与服务加载序列

```
[    1.234567] Freeing unused kernel memory: 2048K           ← Kernel 完成
[    1.345678] Run /sbin/init as init process                 ← BusyBox init 启动
              │
              ├─ /etc/inittab:
              │   ::sysinit:/bin/mount -t proc proc /proc
              │   ::sysinit:/bin/mount -a                      ← 挂载 fstab
              │   ::sysinit:/etc/init.d/rcS                    ← ★ 服务启动入口
              │
              ├─ rcS 脚本遍历 /etc/init.d/S??*:
              │
Starting syslogd: OK                                          ← S01syslogd
Starting klogd: OK                                            ← S02klogd
Running sysctl: OK                                            ← S02sysctl
Populating /dev using udev: done                              ← S10udev
lv_demo is disabled by default                                ← S10lv_demo
Starting system message bus: done                             ← S30dbus
iptables rules applied                                        ← S35iptables
Starting network: OK                                          ← S40network
Starting bluetoothd: OK                                       ← S40bluetoothd
Starting connmand: OK                                         ← S45connman
Starting chronyd: OK                                          ← S49chronyd
Starting crond: OK                                            ← S50crond
Starting pulseaudio: OK                                       ← S50pulseaudio
Starting sshd: OK                                             ← S50sshd
Starting systemui: done                                       ← S50systemui
Starting dnsmasq: OK                                          ← S80dnsmasq
              │
              ├─ getty 在 ttyFIQ0 上启动 (BoardRoot Ubuntu rootfs)
              │   或直接 /bin/sh (Buildroot rootfs)
              │
Welcome to Ubuntu 22.04.5 LTS!                                ← /etc/issue
sportcam login:                                               ← ★ 登录提示符
```

##### 7.2 完整服务对照表

| 优先级 | 脚本 | 日志特征 | 作用 |
|--------|------|---------|------|
| S01 | `syslogd` | `Starting syslogd: OK` | 系统日志守护进程 |
| S02 | `klogd` | `Starting klogd: OK` | 内核日志守护进程 |
| S02 | `sysctl` | `Running sysctl: OK` | 应用 `/etc/sysctl.d/` 内核参数 |
| S10 | `udev` | `Populating /dev using udev: done` | 设备节点创建 (eudev) |
| S10 | `lv_demo` | `lv_demo is disabled by default` | LVGL 演示 (默认禁用) |
| S30 | `dbus` | `Starting system message bus: done` | D-Bus 消息总线 |
| S35 | `iptables` | (无标准输出, 静默执行) | 防火墙规则加载 |
| S40 | `network` | `Starting network: OK` | `ifup -a` 启动网卡 |
| S40 | `bluetoothd` | `Starting bluetoothd: OK` | 蓝牙守护进程 |
| S45 | `connman` | `Starting connmand: OK` | 连接管理器 (WiFi/以太网) |
| S49 | `chronyd` | `Starting chronyd: OK` | NTP 时间同步 |
| S50 | `crond` | `Starting crond: OK` | Cron 定时任务 |
| S50 | `pulseaudio` | `Starting pulseaudio: OK` | PulseAudio 声音服务 |
| S50 | `sshd` | `Starting sshd: OK` | SSH 服务器 (openssh) |
| S50 | `systemui` | `Starting systemui: done` | Qt System UI (等待 Wayland) |
| S80 | `dnsmasq` | `Starting dnsmasq: OK` | DNS/DHCP 服务器 |
| — | `getty` | (通过 /etc/inittab 或 systemd) | `ttyFIQ0` 登录终端 |

> **源码位置**：Init 脚本分布在
> - Buildroot 标准脚本：`buildroot/package/*/S??*` (编译后安装到 target)
> - Alientek 板级覆盖：`buildroot/board/alientek/atk-dlrv1126b/fs-overlay/etc/init.d/S10lv_demo`, `S50systemui`
> - BoardRoot 框架 (实际部署时)：通过 `boardroot/` 目录的 profile 脚本覆盖

##### 7.3 rootfs 类型识别

本 SDK 存在**两套 rootfs 构建体系**：

| 体系 | 构建方式 | 用途 | 识别标志 |
|------|---------|------|---------|
| **Buildroot** | `buildroot/configs/rv1126b_sportcam_defconfig` | SDK 默认编译目标 | `Welcome to RV1126B Buildroot` |
| **BoardRoot (Ubuntu)** | `device/rockchip/common/boardroot/` | 实际板端部署 | `Welcome to Ubuntu 22.04.5 LTS!` |

BoardRoot 是叠加在 debootstrap 之上的厂商适配框架，负责将 SDK 编译的二进制 (MPP/RKAIQ/RGA) 集成到 Ubuntu rootfs 中。实际板端运行的是 BoardRoot 构建的 Ubuntu rootfs，因此日志中出现的是 Ubuntu 风格的 getty 和 `sportcam` 主机名，而非 Buildroot 默认值。

##### 7.4 Camera 服务的特殊说明

| 服务 | 启动方式 | 当前状态 |
|------|---------|---------|
| `rkaiq_3A_server` | `/etc/init.d/S40rkaiq_3A` (external 目录中存在) | **未安装到 rootfs**，需手动添加 |
| ISP 驱动 (rkisp) | 内核模块，Kernel 启动时 probe | **自动加载** (取决于 DTS 配置) |
| MPP 服务 (mpp_service) | 内核驱动，/dev/mpp_service | **自动创建** (设备树已有) |
| rkipc IPC 应用 | `app/rkipc/` 编译产物 | 需手动添加或通过 BoardRoot 集成 |

> **提示**：如果板上看不到 `rkaiq_3A_server` 启动，需要将 `external/camera_engine_rkaiq/rkaiq_3A_server/S40rkaiq_3A` 安装到 rootfs 的 `/etc/init.d/` 目录。

##### 7.5 grep/sed 定位增强

```bash
# 检查 init 启动
grep -m1 "Run /sbin/init" boot.log

# 统计启动了多少个服务
grep -c "Starting.*OK\|Starting.*done" boot.log

# 列出所有启动的服务
grep "Starting.*OK\|Starting.*done\|disabled by default" boot.log

# 检查哪些服务启动失败
grep "Starting.*FAIL" boot.log

# 检查是否到达登录提示符
grep -E "Welcome to|login:|sportcam" boot.log

# 识别 rootfs 类型
grep -m1 "Welcome to" boot.log | grep -E "Ubuntu|Buildroot"
```

##### 7.6 服务启动异常诊断

| 停在哪个日志之后 | 最可能的故障 |
|-----------------|------------|
| 无 `Run /sbin/init` | 根文件系统挂载失败 (检查 root=PARTUUID / rootfstype) |
| `Populating /dev using udev: done` 后无后续服务 | udev 挂死 (驱动 probe 循环) |
| `Starting network: FAIL` | 网卡驱动未加载或 DTS 未启用网络节点 |
| `Starting connmand: OK` 后长时间无 progress | connman 等待 DHCP (无网络时正常) |
| `Starting sshd: FAIL` | 主机密钥缺失 (`/etc/ssh/ssh_host_*_key` 未生成) |
| 启动后长时间无 login | getty 未配置在 ttyFIQ0 上 (检查 /etc/inittab) |
| systemui 启动无显示 | Wayland compositor 未运行 (检查 weston 是否启动) |

---

#### 完整的 grep 分段时间分析脚本

```bash
#!/bin/bash
# 分析启动日志，输出各阶段耗时点

LOG=boot.log

echo "=== 阶段分界点 ==="
echo "SPL 入口:       $(grep -m1 'U-Boot SPL board init' $LOG | awk '{print $1}')"
echo "FIT 加载开始:   $(grep -m1 'Trying fit image at' $LOG | awk '{print $1}')"
echo "FIT 加载结束:   $(grep '## Checking' $LOG | tail -1 | awk '{print $1}')"
echo "U-Boot 版本:    $(grep -m1 '^U-Boot' $LOG | awk '{print $1}')"
echo "Kernel 启动:    $(grep -m1 'Starting kernel' $LOG | awk '{print $1}')"
echo "Kernel 第一条:  $(grep -m1 'Booting Linux' $LOG | awk '{print $1}')"
echo "Kernel 版本:    $(grep -m1 'Linux version' $LOG | awk '{print $1}')"
echo "Kernel 完成:    $(grep -m1 'Freeing unused kernel' $LOG | awk '{print $1}')"
echo "登录提示符:     $(grep -m1 -E 'login:|Welcome to' $LOG | awk '{print $1}')"
echo ""
echo "=== 异常检测 ==="
echo "DDR 初始化失败: $(grep -c 'DRAM init failed' $LOG)"
echo "FIT magic 错误: $(grep -c 'Not fit magic' $LOG)"
echo "Hash 校验失败:  $(grep -c 'error!' $LOG)"
echo "验签失败:       $(grep -c 'verify.*failed' $LOG)"
echo "回滚拒绝:       $(grep -c 'reject rollback' $LOG)"
echo "hang 死锁:      $(grep -c 'failed to boot from all' $LOG)"
```

#### 日志流异常快速诊断表

| 停在哪个日志之后 | 最可能的故障 |
|-----------------|------------|
| 始终无输出 | 电源问题 / 串口波特率不匹配 / DDR bin 不兼容 |
| `U-Boot SPL board init` 后无输出 | DM 初始化失败 (检查 DTS `u-boot,dm-spl` 标记) |
| `Trying fit image` 后无输出 | eMMC 扇区不可读 / FIT 文件损坏 |
| `Not fit magic` | FIT 写入不完整 / 分区偏移错误 |
| `sha256 error!` | FIT 中某段二进制损坏 / 烧录不完整 |
| 某条 `## Checking` 后无 `OK` | 对应段的 hash 或签名验证失败 |
| 最后一条 `OK` 后无输出 | ATF BL31 崩溃 (可能 entry point 或 DDR 问题) |
| `U-Boot` 版本号后无 `Starting kernel` | autoboot 被打断 / bootargs 错误 |
| `Starting kernel` 后无 `Booting Linux` | Kernel Image 损坏 / DTB 不兼容 / ATF 未正确返回 |
| Kernel panic/opps | 驱动 probe 失败 / 内存不足 / DTS 配置错误 |

---

##### 7.7 BoardRoot systemd init 日志分析

> 板上部署的是 **BoardRoot Ubuntu 22.04**，实际 init 系统采用 **systemd**，非前文 Buildroot 的 BusyBox init。
> 本节补充 systemd 环境下的日志特征、分析工具和异常诊断方法。

---

###### 7.7.1 内核→systemd 过渡细节

```
[    1.234567] Freeing unused kernel memory: 2048K           ← Kernel 完成
[    1.345678] Run /sbin/init as init process                 ← Kernel 启动 init (PID 1)
               │
               ├─ kernel_execve("/sbin/init", ...)           ← exec system call
               │    [进程上下文切换, 进入用户空间]
               │
               ├─ systemd[1]: systemd 255 running in system mode (+PAM +AUDIT...)
               │    ← PID 1 第一条日志
               │
               ├─ systemd[1]: Detected architecture arm64.
               │
               ├─ systemd[1]: Set hostname to <sportcam>.
               │
               ├─ systemd[1]: Initializing timer...
               ├─ systemd[1]: Reached target Local File Systems.
               ├─ systemd[1]: Reached target Network.
               │
               ├─ systemd[1]: Starting Getty on ttyFIQ0...
               │
Welcome to Ubuntu 22.04.5 LTS!                                ← /etc/issue (getty 打印)
sportcam login:                                               ← 登录提示符
```

**关键差异 vs BusyBox init：**

| 特性 | Buildroot (BusyBox) | BoardRoot (systemd) |
|------|--------------------|--------------------|
| PID 1 | `/sbin/init` → BusyBox init | `/sbin/init` → systemd |
| 启动顺序 | `/etc/inittab` → `rcS` 遍历 S?? 脚本 | `default.target` → 并行启动 service unit |
| 日志格式 | `Starting syslogd: OK` (脚本串行输出) | `systemd[1]: Starting Network Manager...` (带单元名) |
| 依赖管理 | 手动 S?? 编号 | Unit `After=` / `Requires=` 自动排序 |
| 失败处理 | 脚本返回非0 → 继续下一个 | `OnFailure=`, `Restart=` 自动恢复 |
| 速度 | 串行启动，较慢 | 并行启动，更快 |

---

###### 7.7.2 systemd 完整启动日志序列

```
[    1.234567] Run /sbin/init as init process
               │
[    1.345678] systemd[1]: systemd 255 running in system mode (+PAM +AUDIT +SELINUX +APPARMOR +IMA +SMACK +SECCOMP +GCRYPT +GNUTLS +OPENSSL +ACL +BLKID +CURL +ELFUTILS +FIDO2 +IDN2 -IDN +IPTC +KMOD +LIBCRYPTSETUP +LIBFDISK +PCRE2 -PWQUALITY +P11KIT +QRENCODE +TPM2 +BZIP2 +LZ4 +XZ +ZLIB +ZSTD +BPF_FRAMEWORK +XKBCOMMON +UTMP +SYSVINIT default-hierarchy=unified)
[    1.367890] systemd[1]: Detected architecture arm64.
[    1.378901] systemd[1]: Set hostname to <sportcam>.
[    1.412345] systemd[1]: /etc/machine-id missing, using random UUID.
               │
               │ ── 核心启动阶段 ──
[    1.456789] systemd[1]: Initializing timer...
[    1.467890] systemd[1]: Reached target Timer Units.
[    1.478901] systemd[1]: Listening on Syslog Socket.
[    1.489012] systemd[1]: Starting Journal Service...
[    1.523456] systemd[1]: Started Journal Service.
[    1.534567] systemd[1]: Starting Load Kernel Modules...
[    1.578901] systemd[1]: Starting Apply Kernel Variables...
               │
               │ ── 文件系统 ──
[    1.623456] systemd[1]: Started Remount Root and Kernel File Systems.
[    1.667890] systemd[1]: Started Create Static Device Nodes in /dev.
[    1.712345] systemd[1]: Reached target Local File Systems.
               │
               │ ── 网络 ──
[    1.756789] systemd[1]: Starting Network Manager...
[    2.123456] systemd[1]: Started Network Manager.
[    2.234567] systemd[1]: Reached target Network.
[    2.345678] systemd[1]: Reached target Network is Online.
               │
               │ ── 服务启动 ──
[    2.456789] systemd[1]: Started SSH Server (started at boot).
[    2.567890] systemd[1]: Started D-Bus System Message Bus.
[    2.678901] systemd[1]: Started Getty on ttyFIQ0.
               │
               │ ── 达到默认目标 ──
[    2.789012] systemd[1]: Reached target Multi-User System.
[    2.890123] systemd[1]: Reached target Graphical Interface.
[    2.901234] systemd[1]: Startup finished in 1.234s (kernel) + 1.567s (init) = 2.801s.
               │
Welcome to Ubuntu 22.04.5 LTS!                                ← /etc/issue
sportcam login:                                               ← 登录提示符
```

**grep/sed 定位：**

```bash
# systemd 第一条日志
grep -m1 "systemd\[1\]" boot.log

# 内核→init 切换点
grep -m1 "Run /sbin/init" boot.log

# systemd 启动完成
grep -m1 "Startup finished in" boot.log

# 各阶段 target 到达
grep "Reached target" boot.log

# 服务启动 (Starting = 开始, Started = 完成)
grep -E "Starting|Started" boot.log

# 失败检测
grep -E "FAILED|failed|timed out" boot.log
```

---

###### 7.7.3 systemd 分析工具

```bash
# 板端执行

# 1. 总启动时间分析
systemd-analyze
# 输出: Startup finished in 1.234s (kernel) + 1.567s (init) = 2.801s
#                  └─内核阶段      └─systemd阶段    └─总耗时

# 2. 各服务启动耗时排序
systemd-analyze blame
# 输出格式:
# 1.234s networkd-dispatcher.service
# 0.567s NetworkManager-wait-online.service
# 0.345s systemd-resolved.service
# 0.123s ssh.service
# ...

# 3. 关键链分析 (找出最长的依赖链)
systemd-analyze critical-chain
# 输出:
# graphical.target @2.801s
# └─multi-user.target @2.789s
#   └─NetworkManager.service @2.123s +123ms
#     └─network-pre.target @2.000s
#       └─systemd-sysctl.service @1.999s +5ms
#         └─...

# 4. 单元启动耗时图 (SVG)
systemd-analyze plot > boot_plot.svg
# 可在 PC 端用浏览器查看

# 5. 查看服务依赖树
systemctl list-dependencies multi-user.target

# 6. 查看单元状态
systemctl status ssh.service
# 输出: active (running) / failed / inactive

# 7. 列出所有失败的服务
systemctl --failed

# 8. 查看启动日志 (指定优先级)
journalctl -b -p err         # 仅错误
journalctl -b -p warning     # 警告及以上
journalctl -b --no-pager     # 当前启动全部日志

# 9. 查看特定服务的启动日志
journalctl -b -u NetworkManager.service
journalctl -b -u ssh.service

# 10. 启动时间 json 导出 (供脚本分析)
systemd-analyze dump | grep -E "time=|name="
```

**完整启动分析脚本：**

```bash
#!/bin/bash
# 分析 systemd 启动性能

echo "=== systemd 总启动时间 ==="
systemd-analyze

echo ""
echo "=== 最耗时的服务 (top 10) ==="
systemd-analyze blame | head -10

echo ""
echo "=== 关键依赖链 ==="
systemd-analyze critical-chain | head -20

echo ""
echo "=== 失败的服务 ==="
systemctl --failed

echo ""
echo "=== 启动错误日志 ==="
journalctl -b -p err --no-pager | head -20

echo ""
echo "=== 内核 vs init 耗时对比 ==="
systemd-analyze | awk '{print $3, $5, $7}'
```

---

###### 7.7.4 BoardRoot Ubuntu 服务单元对照表

板上部署的 BoardRoot Ubuntu 22.04 关键服务单元：

| 服务单元 | systemd 单元名 | 对应 Buildroot S?? 脚本 | 作用 |
|---------|--------------|----------------------|------|
| SSH 服务器 | `ssh.service` | S50sshd | OpenSSH 远程登录 |
| D-Bus | `dbus.service` | S30dbus | 系统消息总线 |
| 网络管理器 | `NetworkManager.service` | S40network | 网络配置 (替代 ifup/ifdown) |
| 蓝牙 | `bluetooth.service` | S40bluetoothd | BlueZ 蓝牙栈 |
| 连接管理器 | `connman.service` | S45connman | WiFi/以太网连接 |
| NTP | `chrony.service` | S49chronyd | 时间同步 |
| Cron | `cron.service` | S50crond | 定时任务 |
| PulseAudio | `pulseaudio.service` | S50pulseaudio | 声音服务 |
| DNS/DHCP | `dnsmasq.service` | S80dnsmasq | DNS 转发 / DHCP 服务器 |
| 系统日志 | `syslog.service` / `rsyslog.service` | S01syslogd | 系统日志记录 |
| 内核日志 | `systemd-journald.service` | S02klogd | 内核日志 (systemd 内置) |
| 设备管理 | `systemd-udevd.service` | S10udev | udev 设备管理器 (systemd 内置) |
| 终端登录 | `getty@ttyFIQ0.service` | — | getty 在串口上 |
| **Camera** | `rkaiq_3A_server.service` | S40rkaiq_3A | **3A 服务 (需手动添加)** |
| **IPC** | `rkipc.service` | — | **运动相机管线 (需手动集成)** |

> **注意**：Camera 相关的 `rkaiq_3A_server` 和 `rkipc` 在 BoardRoot 中可能未自动启用，需要：
> ```bash
> systemctl enable rkaiq_3A_server.service
> systemctl start rkaiq_3A_server.service
> ```

---

###### 7.7.5 systemd 特有异常诊断

| 现象 | systemd 诊断命令 | 最可能的故障 |
|------|-----------------|------------|
| 启动后无 login | `systemctl status getty@ttyFIQ0.service` | getty 未启用或 tty 不存在 |
| 网络不能用 | `systemctl status NetworkManager.service` | 驱动未加载 / DTS 未启用网口 |
| SSH 连不上 | `systemctl status ssh.service` | 主机密钥缺失 / 端口被占用 |
| 系统卡住 X 秒 | `systemd-analyze critical-chain` | 某服务等待超时 (network-online 常见) |
| USB 设备不识别 | `journalctl -b -u systemd-udevd.service` | udev 规则缺失 / 驱动未 probe |
| 蓝牙不工作 | `journalctl -b -u bluetooth.service` | 固件缺失 / 硬件不兼容 |
| 内核崩溃重启 | `journalctl -b -k -p err` | 驱动 bug / OOM / 看门狗 |
| 某服务反复重启 | `systemctl status <unit> --no-pager` | `Restart=` 策略 + 连续崩溃 |
| **启动慢 (>30s)** | `systemd-analyze blame \| head -5` | NetworkManager-wait-online 最常背锅 |

**systemd 启动超时调试：**

```bash
# 1. 哪个服务最慢
systemd-analyze blame | head -5

# 2. 关键链等待点
systemd-analyze critical-chain

# 3. 禁用 network-online 等待 (常见优化)
systemctl mask NetworkManager-wait-online.service
systemctl mask systemd-networkd-wait-online.service

# 4. 查看特定服务的超时设置
systemctl show ssh.service | grep Timeout
```

---

###### 7.7.6 systemd 的 BusyBox/boot.log 兼容适配

如果需要在 **同一个文档** 中同时支持两套 rootfs 分析：

```bash
# 自动检测 init 类型
if systemctl --version &>/dev/null; then
    echo "Init: systemd"
    systemd-analyze
else
    echo "Init: BusyBox (无 systemd)"
    grep "Starting.*OK" /var/log/boot.log 2>/dev/null || echo "No boot.log found"
fi

# 兼容的 grep 方式 (同时匹配 BusyBox 和 systemd 日志)
# 登录就绪:
grep -m1 -E "Welcome to|login:|sportcam|Startup finished" boot.log

# 服务失败:
grep -E "FAIL|failed|error!|timed out" boot.log

# 内核完成:
grep -m1 -E "Freeing unused kernel|Run /sbin/init" boot.log
```

---

###### 7.7.7 实验：对比 BusyBox vs systemd 启动耗时

```bash
# 条件: 分别用 Buildroot (BusyBox) 和 BoardRoot (systemd) 启动

# 1. 内核统计 (两套 rootfs 通用)
dmesg | grep -E "Freeing unused|Run /sbin/init"
# 预期: 内核耗时基本一致 (因为内核相同)

# 2. 用户空间启动
# Buildroot: 从 Run /sbin/init 到 login:
#   → 串行 S?? 脚本, ~500-1000ms
# BoardRoot (systemd):
#   → 并行启动, ~800-1500ms (依赖服务更多)
systemd-analyze | grep "init ="

# 3. 优化预期
# systemd 可通过 mask 不必要的服务、启用并行、调整 Timeout 提速
```

---

## 二、板级适配的 4 层架构

```
┌────────────────────────────────────────────────────────────────┐
│ 层次        路径                            适配新板时           │
├────────────────────────────────────────────────────────────────┤
│ 1. Defconfig configs/myboard_defconfig       ★ 核心配置          │
│ 2. Kconfig   arch/.../rv1126b/Kconfig        ★ 新增选项          │
│              board/.../myboard/Kconfig        ★ 映射路径          │
│ 3. Board      board/rockchip/myboard/*.c      ★ 板级差异          │
│ 4. SoC        arch/.../rv1126b/rv1126b.c      △ 通常不改         │
└────────────────────────────────────────────────────────────────┘
```

---

## 三、Defconfig 层 — 启动配置的决策中心

### 3.1 核心配置项解析

> 文件：`configs/rv1126b_sportcam_defconfig` (206 行)

```kconfig
# === SoC 选型 ===
CONFIG_ROCKCHIP_RV1126B=y           # ARM64, Cortex-A53 ×4
CONFIG_TARGET_EVB_RV1126B=y          # 选择 evb_rv1126b 板级目标

# === 启动介质 ===
CONFIG_ROCKCHIP_EMMC_IOMUX=y         # 从 eMMC 启动 (不是 SD 或 SPI)
# CONFIG_ROCKCHIP_SDMMC_IOMUX is not set
# CONFIG_ROCKCHIP_SFC_IOMUX is not set (SPI Flash)

# === 镜像格式 ===
CONFIG_ROCKCHIP_FIT_IMAGE=y          # 使用 FIT 格式
CONFIG_SPL_LOAD_FIT=y                # SPL 加载 FIT 镜像
CONFIG_SPL_ATF=y                     # SPL 加载 ARM Trusted Firmware
CONFIG_SPL_AB=y                      # A/B 分区支持

# === DTB 选择 ===
CONFIG_DEFAULT_DEVICE_TREE="rv1126b-alientek"  # U-Boot 使用的 DTB
CONFIG_USING_KERNEL_DTB_V2=y         # 启动内核时复用内核 DTB

# === 调试 ===
CONFIG_DEBUG_UART=y
CONFIG_DEBUG_UART_BASE=0x20810000    # UART0 基地址
CONFIG_BAUDRATE=1500000              # 波特率 1.5Mbps

# === 驱动 ===
CONFIG_MMC_DW_ROCKCHIP=y             # DesignWare MMC (eMMC/SD)
CONFIG_DM_PMIC=y
CONFIG_PMIC_RK801=y                  # RK801 电源管理
CONFIG_SYS_I2C_ROCKCHIP=y            # I2C 驱动
```

### 3.2 启动介质选择的影响范围

| 配置项 | IOMUX 行为 | SPL 启动源 | 分区查找目标 |
|--------|-----------|-----------|------------|
| `ROCKCHIP_EMMC_IOMUX` | GPIO1A/B → eMMC 功能 | `/dev/mmcblk0` | eMMC GPT 分区 |
| `ROCKCHIP_SDMMC_IOMUX` | GPIO2A → SD 功能 | `/dev/mmcblk1` | SD 卡分区 |
| `ROCKCHIP_SFC_IOMUX` | GPIO1B/0A → FSPI | mtdblock0 | SPI Flash |

---

## 四、Kconfig 层 — 从符号到文件路径

### 4.1 三层 Kconfig 链条

```
rv1126b_sportcam_defconfig
  │  CONFIG_ROCKCHIP_RV1126B=y
  │  CONFIG_TARGET_EVB_RV1126B=y
  │
  ├─► arch/arm/mach-rockchip/Kconfig (顶层)
  │   config ROCKCHIP_RV1126B
  │     select ARM64
  │     select SUPPORT_SPL
  │     ...
  │
  ├─► arch/arm/mach-rockchip/rv1126b/Kconfig (SoC 层)
  │   if ROCKCHIP_RV1126B
  │     config TARGET_EVB_RV1126B
  │       bool "EVB_RV1126B"
  │       select BOARD_LATE_INIT
  │     source board/rockchip/evb_rv1126b/Kconfig  ← 引入板级 Kconfig
  │   endif
  │
  └─► board/rockchip/evb_rv1126b/Kconfig (板级)
      if TARGET_EVB_RV1126B
        config SYS_BOARD       default "evb_rv1126b"    ← 板子目录名
        config SYS_VENDOR      default "rockchip"        ← 厂家名
        config SYS_CONFIG_NAME default "evb_rv1126b"    ← 配置文件前缀
        config BOARD_SPECIFIC_OPTIONS  def_bool y
      endif
```

**映射结果**：
```
SYS_BOARD=evb_rv1126b  +  SYS_VENDOR=rockchip
  → 编译 board/rockchip/evb_rv1126b/
  → 包含 include/configs/evb_rv1126b.h
```

### 4.2 新增板子时的 Kconfig 修改

```kconfig
# arch/arm/mach-rockchip/rv1126b/Kconfig 新增:
config TARGET_MYBOARD
    bool "My Custom Board"
    select BOARD_LATE_INIT
    source board/rockchip/myboard/Kconfig
```

---

## 五、Board 层 — 板级差异代码

### 5.1 当前板级代码

> 文件：`board/rockchip/evb_rv1126b/evb_rv1126b.c`

非常薄的一层，仅处理 USB gadget 配置：

```c
// DWC3 USB 设备配置
static struct dwc3_device_data dwc3_device_data = {
    .maximum_speed        = USB_SPEED_HIGH,
    .base                 = 0x21500000,  // USB DWC3 控制器基址
    .dr_mode              = USB_DR_MODE_PERIPHERAL,
    .index                = 0,
};

int board_usb_init(int index, enum usb_init_type init) {
    // 初始化 USB gadget (RockUSB 下载模式)
    return dwc3_uboot_init(&dwc3_device_data);
}
```

### 5.2 适配新板子时可能添加的内容

```c
// 示例: 自定义 LED、按键、特定传感器初始化
int board_early_init_f(void) {
    // I2C GPIO 扩展器初始化
    // 特定电源上电时序
    return 0;
}

int board_late_init(void) {
    // 设置 MAC 地址
    // 设置序列号
    // 设置 board_name 环境变量
    return 0;
}
```

---

## 六、SoC 层 — `rv1126b.c` 核心分析 (773行)

### 6.1 MMU 内存映射

```c
// rv1126b.c:144-178
static struct mm_region rv1126b_mem_map[] = {
    // Device Memory: 外设寄存器区
    { .virt = 0x20000000, .phys = 0x20000000,
      .size = 0x02800000, .attrs = DEVICE_NGNRNE },  // 40MB 外设空间
    
    // PMU/SRAM 区域
    { .virt = 0x3ff1e000, .phys = 0x3ff1e000,
      .size = 0x000e2000, .attrs = DEVICE_NGNRNE },  // SRAM 区域
    
    // Normal Memory: DDR
    { .virt = 0x40000000, .phys = 0x40000000,
      .size = 0xc0000000, .attrs = NORMAL },          // 最大 3GB DDR
};
```

### 6.2 关键函数拆解

#### `board_set_iomux()` — GPIO 复用配置 (行 188-242)

根据启动介质自动设置引脚功能：

```c
void board_set_iomux(enum if_type if_type, int devnum, int routing) {
    switch (if_type) {
    case IF_TYPE_MMC:  // eMMC 或 SD
        if (devnum == 0) {
            // eMMC: GPIO1A → eMMC_D0-7
            //       GPIO1B → eMMC_CMD/CLK/RSTN
            writel(0xffff1111, GPIO1A_IOMUX_SEL_L);
            writel(0xffff1111, GPIO1A_IOMUX_SEL_H);
            writel(0xf0f01010, GPIO1B_IOMUX_SEL_L);
        } else if (devnum == 1) {
            // SDMMC: GPIO2A → SD_DATA0-3/CMD/CLK
            // 先拉低再上拉 (power cycle 复位 SD 卡)
            writel(0xffff1111, GPIO2A_IOMUX_SEL_L);
            writel(0x00ff0011, GPIO2A_IOMUX_SEL_H);
        }
        break;
    case IF_TYPE_MTD:  // SPI Flash
        // routing=0: FSPI0  → GPIO1B
        // routing=1: FSPI1  → GPIO0A/0B
        // routing=2: FSPI1M1 → GPIO1A/1B
        break;
    }
}
```

**适配新板子时的注意事项**：如果板子的 eMMC/SD 引脚和 EVB 不同，需要在此函数中添加新的 case 分支，或者直接在 defconfig 中禁用 `ROCKCHIP_EMMC_IOMUX` 后在 DTS 中手动配置 pinctrl。

#### `arch_cpu_init()` — SPL 核心初始化 (行 417-519)

这个函数是 SPL 阶段**唯一**的硬件初始化入口，包含三大部分：

**Part 1: 安全防火墙 (行 419-443)**
```c
// 设置存储控制器为 master secure
writel(0x10000, SGRF_SYS_AHB_SECURE_SGRF_CON);  // eMMC
writel(0x20000, SGRF_SYS_AHB_SECURE_SGRF_CON);  // FSPI
writel(0x40000, SGRF_SYS_AHB_SECURE_SGRF_CON);  // SDMMC0
writel(0x80000, SGRF_SYS_AHB_SECURE_SGRF_CON);  // SDMMC1

// 所有外设 slave 设为 non-secure
for (i = 0; i < 6; i++)
    writel(0xffff0000, FIREWALL_SLV_CON0 + i*4);

// OTP 设为 non-secure (允许 Linux 读取芯片 ID)
writel(0x00020000, OTP_SGRF_CON);
```

**Part 2: USB 和温度传感器 (行 445-463)**
```c
// USB3 PHY 初始化 (clamp enable, pipe phy reset)
// TSADC 温度传感器使能、校准配置
// SARADC 采样带宽设置
```

**Part 3: PVTPLL + IOMUX (行 478-515)**
```c
// ISP/ENC/AIISP PVTPLL 使能 (功耗优化)
writel(0x01ff0064, PVTPLL_ISP_GCK_LEN);

// 根据 defconfig 设置启动设备 IOMUX
board_set_iomux(IF_TYPE_MMC, 0, 0);  // eMMC

// FSPI IO 驱动强度适配 (1.8V→level3, 3.3V→level4)
```

### 6.3 SPL 独有的编译控制

```makefile
# arch/arm/mach-rockchip/rv1126b/Makefile
ifneq ($(CONFIG_TPL_BUILD)$(CONFIG_TPL_TINY_FRAMEWORK),yy)
obj-y += syscon_rv1126b.o      # TPL 不使用 DM，跳过
endif
obj-y += rv1126b.o
obj-y += clk_rv1126b.o
```

`rv1126b.c` 中的很多代码通过 `#if defined(CONFIG_SPL_BUILD)` 控制 SPL 和 U-Boot proper 的不同行为。

---

## 七、U-Boot Driver Model 核心机制

### 7.1 `u-boot,dm-spl` — SPL 设备树标记

> 文件：`arch/arm/dts/rv1126b-u-boot.dtsi`

SPL 阶段**内存极其有限**，需要通过 DTS 标记精确控制哪些驱动被加载：

```dts
// SPL 阶段必须初始化的设备
&cru      { u-boot,dm-spl; status = "okay"; };  // 时钟
&grf      { u-boot,dm-spl; status = "okay"; };  // 寄存器文件
&uart0    { u-boot,dm-spl; status = "okay"; };  // 调试串口
&emmc     { u-boot,dm-spl; status = "okay"; };  // eMMC (启动设备)
&pinctrl  { u-boot,dm-spl; status = "okay"; };  // 引脚控制
&saradc0  { u-boot,dm-pre-reloc; status = "okay"; }; // ADC 按键

// SPL 启动设备顺序
chosen {
    u-boot,spl-boot-order = &emmc;  // 仅从 eMMC 启动
};
```

| 标记 | 含义 | 加载阶段 |
|------|------|---------|
| `u-boot,dm-spl` | SPL 阶段探活 | `board_init_f` 后的 DM 扫描 |
| `u-boot,dm-pre-reloc` | 重定位前探活 | 比 dm-spl 稍早 |
| `u-boot,dm-pre-proper` | U-Boot proper 前探活 | SPL→U-Boot 过渡期 |
| 无标记 | 仅 U-Boot proper 探活 | `board_init_r` 后 |

### 7.2 CRU 时钟驱动初始化示例

```c
// drivers/clk/rockchip/clk_rv1126b.c
U_BOOT_DRIVER(rockchip_rv1126b_cru) = {
    .name     = "rockchip_rv1126b_cru",
    .id       = UCLASS_CLK,
    .of_match = rv1126b_clk_ids,          // compatible="rockchip,rv1126b-cru"
    .ops      = &rv1126b_clk_ops,
    .bind     = rv1126b_clk_bind,         // 创建 reset 子设备
    .probe    = rv1126b_clk_probe,        // 初始化 PLL 和时钟树
};
```

Probe 过程：
```
1. ofdata_to_platdata → 从 DTS 读取 reg=<0x20000000>
2. probe:
   ├─ 修正 maskrom 改写的 GPLL 频率
   ├─ rv1126b_clk_init() → 初始化 GPLL/CPLL/AUPLL
   └─ 处理 DTS 中的 assigned-clocks
```

### 7.3 DM SPL 设备扫描流程

```
board_init_f()
  └─ spl_early_init()
       └─ dm_init_and_scan(true)     // 预重定位 DM 扫描
            ├─ 解析 DTB
            ├─ 遍历所有 u-boot,dm-spl 标记的节点
            ├─ 对每个节点查找匹配的 U_BOOT_DRIVER
            ├─ 调用驱动的 bind() 创建设备实例
            └─ 调用驱动的 probe() 初始化硬件
```

---

## 八、DDR 初始化 — 闭源固件

RV1126B 的 DDR 训练由 Rockchip 提供的**预编译二进制**完成，是整个适配链中唯一不可修改的部分。

```
rkbin/bin/rv11/rv1126b_ddr_1332MHz_v1.09.bin  → DDR 初始化代码
rkbin/bin/rv11/rv1126b_spl_v1.05.bin           → SPL 加载器
  ↓ 合并
rkbin/RKBOOT/rv1126b_spl_loader_v1.09.105.bin  → 烧录到 eMMC uboot 分区
```

SPL 代码中的 DDR 驱动只是一个**空壳** (`drivers/ram/rockchip/sdram_rv1126b.c`)：

```c
static int sdram_init(struct udevice *dev) {
    // RV1126B DDR 已由 TPL 初始化，SPL 只需确认状态
    return -1;  // 返回 -1 表示 DDR 不由 SPL 管理
}
```

**适配新板子时**：
- 如果 DDR 颗粒与 EVB 相同 → 不需要修改，直接使用现有 DDR bin
- 如果更换 DDR 颗粒/容量/频率 → 需向 Rockchip 申请匹配的 DDR bin
- 这是整个适配链中**唯一的外部依赖项**

---

## 九、完整启动时序与日志解读

### 9.1 各阶段耗时参考

```
[阶段]                [标志日志]                       [典型耗时]
BootROM               (无日志)                          ~50ms
TPL/DDR Init          (无日志, DDR 初始化)              ~100ms
SPL board_init_f      (无串口输出)                      ~10ms
SPL load FIT          "Trying fit image at ..."         ~50ms
  └─ 验签 + 解压       "## Checking atf-1 ... OK"       ~5ms/段
ATF BL31              (切换到 EL3)                      ~10ms
U-Boot proper init    "U-Boot 2017.09-..."              ~200ms
Kernel boot           "Starting kernel ..."             ~2s
```

### 9.2 调试 sopt 命令 (U-Boot shell)

```bash
# U-Boot 命令行中查看启动参数
printenv                     # 查看所有环境变量
printenv bootargs            # 查看内核启动参数
bdinfo                       # 查看板级信息 (DRAM 大小/地址)
mmcinfo                      # 查看 eMMC 信息
mmc part                     # 查看 eMMC 分区表
mmc dev 0                    # 切换到 eMMC
fatls mmc 0:3                # 列出 boot 分区内容

# FIT 镜像调试
iminfo 0x40200000            # 查看 FIT 镜像头信息

# 内存/寄存器调试
md 0x40000000 0x100          # 查看 DDR 起始 256 字节
md 0x20000000 0x10           # 查看 CRU 寄存器 (时钟)
```

---

## 十、适配新 RV1126B 板子 — 完整 Checklist

### Step 1: 创建板级目录和文件

```bash
cd u-boot
mkdir -p board/rockchip/myboard

# board/rockchip/myboard/evb_rv1126b.c → myboard.c
# 修改 board_usb_init() 适配板子的 USB 接口
```

### Step 2: 创建 defconfig

```bash
cp configs/rv1126b_sportcam_defconfig configs/myboard_defconfig

# 修改:
CONFIG_DEFAULT_DEVICE_TREE="myboard"     # DTS 文件名
# 根据实际启动介质选择:
#   CONFIG_ROCKCHIP_EMMC_IOMUX=y         # 或
#   CONFIG_ROCKCHIP_SDMMC_IOMUX=y        # 或
#   CONFIG_ROCKCHIP_SFC_IOMUX=y
```

### Step 3: 创建板级 DTS

```bash
cp arch/arm/dts/rv1126b-alientek.dts arch/arm/dts/myboard.dts

# 修改:
/ {
    model = "My Custom RV1126B Board";
    compatible = "mycompany,myboard", "rockchip,rv1126b";
};
```

### Step 4: 更新 Kconfig

```bash
# arch/arm/mach-rockchip/rv1126b/Kconfig 添加:
config TARGET_MYBOARD
    bool "My Custom Board"
    select BOARD_LATE_INIT

# board/rockchip/myboard/Kconfig:
if TARGET_MYBOARD
    config SYS_BOARD       default "myboard"
    config SYS_VENDOR      default "rockchip"
    config SYS_CONFIG_NAME default "myboard"
    config BOARD_SPECIFIC_OPTIONS  def_bool y
endif
```

### Step 5: 创建 board Makefile

```makefile
# board/rockchip/myboard/Makefile
obj-y += myboard.o
```

### Step 6: 创建 board config header

```c
// include/configs/myboard.h
#include <configs/rv1126b_common.h>
// 板级特定宏定义
```

### Step 7: 适配板级代码

```c
// board/rockchip/myboard/myboard.c
#include <common.h>

// 板级外设初始化
int board_early_init_f(void) {
    // 特定电源上电时序
    // GPIO 扩展器初始化
    return 0;
}

// 板级 USB 配置
int board_usb_init(int index, enum usb_init_type init) {
    // USB PHY 初始化
    return 0;
}
```

### Step 8: 确认 DDR bin 兼容性

```bash
ls rkbin/bin/rv11/rv1126b_ddr_*_v*.bin
# 确保 DDR 颗粒型号匹配
# 不匹配 → 联系 Rockchip 获取
```

### Step 9: 编译和烧录

```bash
# 编译
make myboard_defconfig
make -j$(nproc)

# 烧录 (通过 USB 下载模式)
./rkbin/tools/rkdeveloptool db rkbin/RKBOOT/rv1126b_spl_loader_*.bin
./rkbin/tools/rkdeveloptool wl 0x40 u-boot.itb  # 写入 uboot 分区
```

---

## 十一、相关笔记

- [[bsp-boot-flow]] — 阶段一主笔记 (Boot Chain, FIT, 分区表, ATF 分段)
- [[bsp-spl-fit]] — SPL FIT 镜像解析与验证源码追踪
- [[bsp-device-model-dtb]] — 阶段二: 设备模型 + 设备树
- [[MOC-嵌入式Linux]] — 嵌入式 Linux 学习路线 MOC