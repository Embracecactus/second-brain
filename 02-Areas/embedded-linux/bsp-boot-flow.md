---
tags:
  - embedded-linux
  - bsp
  - bootloader
  - u-boot
  - boot-flow
  - rockchip
  - fit-image
  - amp
category: embedded-linux
created: 2026-06-22
updated: 2026-06-22
status: active
soc: Rockchip RV1126B
kernel: Linux 6.1.141
uboot_config: rv1126b_sportcam
---

# 阶段一：Bootloader + 系统启动流程

> **JD对标**：Bootloader移植、Kernel移植、系统启动优化
>
> 本章从上电瞬间到 Linux 用户空间就绪，完整拆解 RV1126B 的启动链路。每个阶段都有对应的实验，最终目标是用工具测量出启动各阶段的精确耗时并做优化。

---

## 一、Boot Chain 全景图

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ BootROM  │───→│ DDR Init │───→│  SPL     │───→│ ATF BL31 │
│ (芯片内置) │    │ (DDR bin)│    │ (spl bin)│    │ (bl31 elf)│
└──────────┘    └──────────┘    └──────────┘    └──────────┘
                                                      │
                 ┌──────────┐    ┌──────────┐         │
                 │  Kernel  │←──│ U-Boot   │←─────────┘
                 │ (FIT img)│    │(uboot.img)│    ┌──────────┐
                 └──────────┘    └──────────┘    │ TEE BL32 │
                      │              │           │(bl32 bin)│
                      ▼              │           └──────────┘
                 ┌──────────┐        │
                 │  Rootfs  │←───────┘
                 │ (ext4)   │    + AMP MCU 固件
                 └──────────┘    (amp.img, RISC-V)
```

### 1.1 各阶段职责

| 阶段 | 固件文件 | 加载地址 | 职责 | 运行权限 |
|------|---------|---------|------|---------|
| **BootROM** | 芯片内置 | — | 从 eMMC/SPI NAND/USB 读取 loader | EL3 |
| **DDR Init** | `rv1126b_ddr_1332MHz_v1.09.bin` | — | 初始化 DDR 控制器、配置时序 | EL3 |
| **SPL** | `rv1126b_spl_v1.05.bin` | 0x4FE00000 | 加载 BL31/BL32/U-Boot | EL3 |
| **ATF BL31** | `rv1126b_bl31_v1.13.elf` | 0x0 | ARM Trusted Firmware 运行时服务 (SMC) | EL3 |
| **TEE BL32** | `rv1126b_bl32_v1.04.bin` | 0x48400000 | OP-TEE 安全操作系统 (可选) | S-EL1 |
| **U-Boot** | `uboot.img` (FIT) | — | 加载内核 FIT 镜像、传递 bootargs/dtb | EL1 |
| **Kernel** | `boot.img` (FIT) | — | Linux 内核 + dtb + resource | EL1 |
| **AMP MCU** | `amp.img` | — | RISC-V MCU 固件 (RT-Thread) | MCU |

### 1.2 固件组成文件

Loader 由 `rkbin/RKBOOT/RV1126BMINIALL.ini` 定义：

```ini
# rkbin/RKBOOT/RV1126BMINIALL.ini
[CODE471_OPTION]
Path1=bin/rv11/rv1126b_ddr_1332MHz_v1.09.bin   # DDR 初始化
[CODE472_OPTION]
Path1=bin/rv11/rv1126b_usbplug_v1.03.bin        # USB 下载模式
[LOADER_OPTION]
FlashData=bin/rv11/rv1126b_ddr_1332MHz_v1.09.bin
FlashBoot=bin/rv11/rv1126b_spl_v1.05.bin         # SPL
[OUTPUT]
PATH=rv1126b_spl_loader_v1.09.105.bin            # 合并后的 loader
```

Trust 镜像由 `rkbin/RKTRUST/RV1126BTRUST.ini` 定义：

```ini
# rkbin/RKTRUST/RV1126BTRUST.ini
[BL31_OPTION]
SEC=1
PATH=bin/rv11/rv1126b_bl31_v1.13.elf   # ATF BL31
ADDR=0x0
[BL32_OPTION]
SEC=0
PATH=bin/rv11/rv1126b_bl32_v1.04.bin   # OP-TEE BL32
ADDR=0x48400000
[OUTPUT]
PATH=trust.img
```

> **关键理解**：BootROM → DDR Init → SPL 是 Rockchip 闭源固件，开发者无法修改源码。开发者可控的起点是 **U-Boot**。

---

## 二、U-Boot 层

### 2.1 U-Boot 配置

当前板子使用 `rv1126b_sportcam_defconfig`：

```bash
# SDK 芯片级配置文件
# device/rockchip/.chips/rv1126b/rv1126b_sportcam_defconfig
RK_UBOOT_CFG="rv1126b_sportcam"
RK_UBOOT_CFG_FRAGMENTS="rk-amp"       # AMP 架构 fragment
RK_PARAMETER="parameter-amp.txt"       # 分区表
RK_AMP=y                               # 启用 AMP
RK_AMP_RTT_TARGET="rv1126b-mcu"        # RISC-V MCU 固件目标
```

U-Boot 自身的 defconfig 在 `u-boot/configs/rv1126b_sportcam_defconfig`，关键配置：

| 配置项 | 值 | 说明 |
|--------|-----|------|
| `CONFIG_ROCKCHIP_RV1126B` | y | SoC 选型 |
| `CONFIG_TARGET_EVB_RV1126B` | y | 板级目标 |
| `CONFIG_DEFAULT_DEVICE_TREE` | rv1126b-alientek | U-Boot 使用的 DTB |
| `CONFIG_USING_KERNEL_DTB_V2` | y | **复用内核 DTB**（而非 U-Boot 自己的） |
| `CONFIG_SPL_LOAD_FIT` | y | SPL 加载 FIT 镜像 |
| `CONFIG_FIT` | y | 启用 FIT 镜像支持 |
| `CONFIG_SYS_I2C_ROCKCHIP` | y | U-Boot 阶段 I2C 驱动 |
| `CONFIG_PMIC_RK8XX` | y | PMIC 驱动 (RK801) |
| `CONFIG_MMC_DW_ROCKCHIP` | y | eMMC 驱动 |
| `CONFIG_ROCKCHIP_SFC` | y | SPI Flash 控制器 |
| `CONFIG_BAUDRATE` | 1500000 | 串口波特率 1.5Mbps |
| `CONFIG_DEBUG_UART_BASE` | 0x20810000 | 调试串口 = UART0 |

### 2.2 U-Boot 设备树

```
u-boot/arch/arm/dts/
├── rv1126b.dtsi           # SoC 基础 dtsi
├── rv1126b-alientek.dts   # 板级 dts (sportcam 使用)
├── rv1126b-u-boot.dtsi    # U-Boot 特有补充
├── rv1126b-pinctrl.dtsi   # 引脚复用
└── rv1126b-pinconf.dtsi   # 引脚配置
```

> **重要**：`CONFIG_USING_KERNEL_DTB_V2=y` 意味着 U-Boot 在加载内核时直接使用内核的 DTB (`rv1126b-sportcam.dtb`)，而不是 U-Boot 自己编译的 `rv1126b-alientek.dtb`。U-Boot 自身的 DTB 仅用于 SPL/U-Boot 阶段的硬件初始化（如 eMMC、I2C、PMIC）。

### 2.3 U-Boot 启动流程

```
SPL 完成 DDR 初始化后:
  → 加载 U-Boot proper (uboot.img, FIT 格式)
  → U-Boot board_init() → 初始化 eMMC/I2C/PMIC
  → 读取 boot 分区 → 解析 FIT 镜像
  → 验签 (sha256,rsa2048)
  → 加载 kernel Image 到内存
  → 加载 dtb 到内存
  → bootm <kernel_addr> - <dtb_addr>
  → 跳转到内核 entry point
```

### 2.4 bootargs 传递机制

bootargs 定义在内核设备树的 `chosen` 节点中：

```dts
/* kernel-6.1/arch/arm64/boot/dts/rockchip/rv1126b-sportcam.dts */
/ {
    chosen {
        bootargs = "net.ifnames=0 \
                    earlycon=uart8250,mmio32,0x20810000 \
                    console=ttyFIQ0 \
                    rw root=PARTUUID=614e0000-0000 \
                    rootfstype=ext4 rootwait \
                    snd_soc_core.prealloc_buffer_size_kbytes=16 \
                    coherent_pool=32K";
    };
};
```

| bootargs 参数 | 含义 |
|--------------|------|
| `earlycon=uart8250,mmio32,0x20810000` | 内核早期控制台 (UART0) |
| `console=ttyFIQ0` | 主控制台使用 FIQ debugger (非标准 ttyS) |
| `root=PARTUUID=614e0000-0000` | 根文件系统分区 UUID |
| `rootfstype=ext4` | 根文件系统类型 |
| `rootwait` | 等待根设备就绪 |
| `coherent_pool=32K` | DMA 一致性内存池大小 |

> **传递路径**：DTS `chosen.bootargs` → U-Boot 读取 → 可被 U-Boot 环境变量 `bootargs` 覆盖 → 内核 `cmdline`

---

## 三、分区表

### 3.1 GPT 分区布局

分区表文件：`device/rockchip/.chips/rv1126b/parameter-amp.txt`

```
CMDLINE: mtdparts=:
  0x00002000@0x00004000(uboot)       → U-Boot + loader
  0x00002000@0x00006000(misc)        → recovery 标志
  0x00020000@0x00008000(boot)        → 内核 FIT 镜像
  0x00002000@0x00028000(amp)         → AMP MCU 固件
  0x00040000@0x0002a000(recovery)    → 恢复内核
  0x00010000@0x0006a000(backup)      → 备份
  0x00c00000@0x0007a000(rootfs)      → 根文件系统 (12MB×512=6GB)
  0x00040000@0x00c7a000(oem)         → OEM 数据
  -@0x00cba000(userdata:grow)        → 用户数据 (剩余空间)
```

| 分区 | 偏移 (LBA) | 大小 (LBA) | 实际大小 | 内容 |
|------|-----------|-----------|---------|------|
| uboot | 0x4000 | 0x2000 | 4MB | `MiniLoaderAll.bin` + `uboot.img` + `trust.img` |
| misc | 0x6000 | 0x2000 | 4MB | recovery 标志位 |
| boot | 0x8000 | 0x20000 | 64MB | `boot.img` (FIT: kernel+dtb+resource) |
| amp | 0x28000 | 0x2000 | 4MB | `amp.img` (RISC-V MCU 固件) |
| recovery | 0x2a000 | 0x40000 | 128MB | `recovery.img` |
| backup | 0x6a000 | 0x10000 | 32MB | 备份分区 |
| rootfs | 0x7a000 | 0xc00000 | ~6GB | `rootfs.img` (ext4) |
| oem | 0xc7a000 | 0x40000 | 128MB | OEM 数据 |
| userdata | 0xcba000 | grow | 剩余 | 用户数据 |

> 单位说明：LBA = 512 bytes。`0x2000` LBA = 4MB。

### 3.2 板端验证

```bash
# 查看实际分区
ls -la /dev/mmcblk0p*
# 预期: mmcblk0p1(uboot) ~ mmcblk0p9(userdata)

# 查看 GPT 分区表
sudo fdisk -l /dev/mmcblk0

# 查看 rootfs 的 PARTUUID
sudo blkid /dev/mmcblk0p7
# 预期: PARTUUID="614e0000-0000-4b53-8000-1d28000054a9"
```

---

## 四、FIT 镜像

### 4.1 FIT 镜像结构

FIT (Flattened Image Tree) 是 U-Boot 的标准镜像格式，类似设备树的 DTS 结构。定义文件 `boot.its`：

```dts
/* device/rockchip/.chips/rv1126b/boot.its */
/ {
    images {
        fdt {
            data = /incbin/("@KERNEL_DTB@");    // → rv1126b-sportcam.dtb
            type = "flat_dt";
            arch = "arm64";
            hash { algo = "sha256"; }
        };
        kernel {
            data = /incbin/("@KERNEL_IMG@");   // → arch/arm64/boot/Image
            type = "kernel";
            os = "linux";
            arch = "arm64";
            entry = <0xffffff01>;              // 占位符，编译时替换
            load = <0xffffff01>;
            hash { algo = "sha256"; }
        };
        resource {
            data = /incbin/("@RESOURCE_IMG@"); // → logo 等资源
            type = "multi";
            hash { algo = "sha256"; }
        };
    };
    configurations {
        conf {
            fdt = "fdt";
            kernel = "kernel";
            multi = "resource";
            signature {
                algo = "sha256,rsa2048";       // RSA2048 签名
                padding = "pss";
                key-name-hint = "dev";
                sign-images = "fdt", "kernel", "multi";
            };
        };
    };
};
```

### 4.2 FIT 镜像打包流程

```bash
# SDK 编译流程 (mk-fitimage.sh)
# 1. 编译内核 → 生成 Image + rv1126b-sportcam.dtb
./build.sh kernel

# 2. 用 mkimage 打包 FIT 镜像
#    @KERNEL_IMG@ → arch/arm64/boot/Image
#    @KERNEL_DTB@ → arch/arm64/boot/dts/rockchip/rv1126b-sportcam.dtb
#    @RESOURCE_IMG@ → resource.img (logo)
mkimage -f boot.its -r boot.img

# 3. 产物 → output/firmware/boot.img
#    写入 boot 分区 (mmcblk0p3)
```

### 4.3 验签机制

```
U-Boot 加载 boot.img 时:
  → 解析 FIT 结构
  → 读取 configuration 的 signature 节点
  → 用 RSA 公钥验签 (sha256,rsa2048)
  → 验签通过 → 加载 kernel + dtb
  → 验签失败 → 停止启动 (安全启动模式)
```

> **注意**：当前 sportcam 配置未严格启用安全启动 (`RK_SECUREBOOT` 未在 defconfig 中设置)，但 FIT 结构本身支持验签。大疆等安全敏感产品会强制启用。

---

## 五、AMP 架构

### 5.1 什么是 AMP

AMP (Asymmetric Multi-Processing) 是 RV1126B 的双系统架构：

```
┌─────────────────────────────┐
│        Cortex-A53 ×4        │
│    Linux (主核, EL1)        │
│    ├── V4L2 / MPP / NPU     │
│    ├── 网络 / 文件系统       │
│    └── 通过 mailbox 与 MCU  │
│        通信                  │
├─────────────────────────────┤
│     RISC-V MCU (协处理器)    │
│    RT-Thread (MCU, 实时)     │
│    ├── 低延迟 IO 控制        │
│    ├── 电机 / 传感器         │
│    └── 实时任务调度          │
└─────────────────────────────┘
```

### 5.2 AMP 固件配置

```bash
# device/rockchip/.chips/rv1126b/rv1126b_sportcam_defconfig
RK_AMP=y
RK_AMP_RISCV=y
RK_AMP_RTT_TARGET="rv1126b-mcu"       # RT-Thread MCU 目标
RK_AMP_FIT_ITS="amp_mcu.its"           # MCU 固件 FIT 镜像
```

### 5.3 AMP 启动顺序

```
1. U-Boot 加载 Linux kernel (boot 分区)
2. U-Boot 加载 MCU 固件 (amp 分区 → amp.img)
3. Linux 内核启动 → 通过 mailbox 通知 MCU 启动
4. 两系统各自独立运行，通过 mailbox/hwlock 通信
```

---

## 六、实验 1：解析完整 Boot Chain

### 6.1 实验目标

通过串口日志和 SDK 文件，画出 RV1126B 从上电到登录提示符的完整启动流程图。

### 6.2 操作步骤

```bash
# 板端：抓取完整启动日志
# 1. 连接串口 (UART0, 1500000 baud)
# 2. 重启板子，抓取从上电到 login 的完整输出

sudo reboot

# 串口终端抓取日志，保存到 boot_full.log

# 3. 标记各阶段分界点
# BootROM → 搜索 "BootDev"
# SPL     → 搜索 "SPL" 或 "DDR"
# U-Boot  → 搜索 "U-Boot"
# Kernel  → 搜索 "Linux version"
# Userspace → 搜索 "Welcome" 或 login
```

### 6.3 日志分段分析

```bash
# PC 端分析
# 统计各阶段行数
grep -n "DDR\|SPL\|U-Boot\|Linux version\|Welcome\|login" boot_full.log

# 测量各阶段时间戳
# U-Boot 阶段: 搜索 "U-Boot" 到 "Starting kernel"
# Kernel 阶段: 搜索 "Linux version" 到 "Freeing unused kernel memory"
# Userspace: 搜索 init 开始到 login
```

### 6.4 预期输出

```
[阶段]               [起始标志]                    [结束标志]
BootROM + DDR Init   (上电)                        "DDR 1332MHz"
SPL                  "SPL"                         "U-Boot"
U-Boot               "U-Boot"                      "Starting kernel"
Kernel              "Linux version 6.1.141"       "Freeing unused kernel memory"
Userspace           "init" / "systemd"             "login:"
```

---

## 七、实验 2：修改 bootargs 验证传递机制

### 7.1 实验目标

修改设备树 `chosen` 节点的 bootargs，重新编译并部署，验证参数确实传递到内核。

### 7.2 操作步骤

```bash
# 1. 修改 DTS 中的 bootargs
# kernel-6.1/arch/arm64/boot/dts/rockchip/rv1126b-sportcam.dts
# 在 chosen.bootargs 末尾添加: "initcall_debug loglevel=8"

# 2. 重新编译内核 (只需 dtb)
./build.sh kernel
# 或只编译 dtb:
make -C kernel-6.1 ARCH=arm64 CROSS_COMPILE=... dtbs

# 3. 重新打包 boot.img
./build.sh firmware   # 或 ./build.sh all

# 4. 部署到板子
# 方法 A: 刷写 boot.img
sudo dd if=output/firmware/boot.img of=/dev/mmcblk0p3 bs=512

# 方法 B: 通过 rkflash.sh
# 在 PC 端: ./rkflash.sh boot

# 5. 重启验证
sudo reboot

# 6. 板端检查 cmdline
cat /proc/cmdline
# 预期: 包含 "initcall_debug loglevel=8"
```

### 7.3 验证结果

```bash
cat /proc/cmdline
# 预期输出:
# net.ifnames=0 earlycon=uart8250,mmio32,0x20810000 console=ttyFIQ0 rw
# root=PARTUUID=614e0000-0000 rootfstype=ext4 rootwait
# snd_soc_core.prealloc_buffer_size_kbytes=16 coherent_pool=32K
# initcall_debug loglevel=8    ← 你新加的参数
```

> **理解**：bootargs 从 DTS `chosen` 节点 → U-Boot 读取 → 传给内核 → `/proc/cmdline`。这是 BSP 开发中调整内核行为最常用的手段。

---

## 八、实验 3：内核启动时间分析

### 8.1 实验目标

用 `initcall_debug` + `printk.time` 精确测量内核启动各阶段耗时，找到启动瓶颈。

### 8.2 操作步骤

```bash
# 1. 确保 bootargs 包含以下参数 (实验 2 已添加):
#    initcall_debug loglevel=8

# 2. 重启板子，抓取完整启动日志到 boot_timing.log

# 3. 板端：分析 initcall 耗时
# 提取所有 initcall 行及其时间戳
dmesg | grep "initcall" | head -50

# 4. 找出最耗时的 initcall
dmesg | grep "initcall" | \
  awk '{print $2, $NF}' | sort -rn | head -20
# $2 = 时间戳, $NF = "Xms" 耗时
```

### 8.3 进阶：用 Ftrace 追踪启动

```bash
# 1. 在 bootargs 中添加:
#    trace_event=initcall:* trace_buf_size=128M
#    ftrace=function_graph ftrace_filter="*" 
#    (注意：启动阶段 ftrace 需要内核配置 CONFIG_FTRACE=y)

# 2. 重启后提取 ftrace 数据
cat /sys/kernel/tracing/trace > /tmp/boot_trace.log

# 3. 分析函数调用时间
# 找出启动期间最耗时的函数
cat /tmp/boot_trace.log | grep "duration" | sort -rn | head -20
```

### 8.4 用 bootgraph 分析

```bash
# PC 端: 用 bootgraph 脚本可视化
# 如果 SDK 中有 scripts/bootgraph.pl:
perl kernel-6.1/scripts/bootgraph.pl boot_timing.log > bootgraph.svg

# 或用 Python 自行分析:
# 提取 initcall name + time，画柱状图
python3 -c "
import re, sys
calls = []
for line in open('boot_timing.log'):
    m = re.search(r'\[\s*(\d+\.\d+)\].*initcall (.+?)\+.*returned (\d+) after (\d+) usecs', line)
    if m:
        calls.append((m.group(2), int(m.group(4))))
calls.sort(key=lambda x: -x[1])
for name, us in calls[:20]:
    print(f'{us/1000:8.2f} ms  {name}')
"
```

### 8.5 预期结果

典型 RV1126B 内核启动耗时分布（参考）：

| initcall | 耗时 (估算) | 说明 |
|----------|------------|------|
| `mipi_csi2` / `rkisp` | ~50-100ms | ISP 子系统初始化 |
| `rknpu` | ~30-50ms | NPU 驱动加载 |
| `mpp_service` | ~20-30ms | MPP 编解码服务 |
| `rk8xx` / `regulator` | ~10-20ms | PMIC / 稳压器 |
| `dwc3` / `usb` | ~10-20ms | USB 控制器 |
| `clock` / `cru` | ~5-10ms | 时钟树初始化 |
| `i2c` / `spi` | ~5-10ms | 总线控制器 |
| **总内核启动** | **~500-800ms** | 从 "Starting kernel" 到 userspace |

---

## 九、实验 4：全链路启动时间测量

### 9.1 实验目标

精确测量 BootROM → U-Boot → Kernel → Userspace 各阶段耗时。

### 9.2 测量方法

```bash
# 方法 1: 串口日志时间戳
# U-Boot 和内核都有时间戳输出
# 在启动日志中找以下标记:

# U-Boot 阶段:
#   "U-Boot ... (Jun xx 2026)"        ← U-Boot 开始
#   "Starting kernel ..."             ← U-Boot 结束 / 内核开始

# 内核阶段:
#   "[    0.000000] Linux version..."  ← 内核时间戳起点
#   "[    X.XXXXX] Freeing unused..."  ← 内核初始化结束
#   "[    X.XXXXX] systemd[1]: ..."    ← 用户空间开始

# Userspace:
#   "Welcome to..." / "login:"        ← 完全就绪

# 方法 2: 内核 initcall_debug (实验 3 已做)

# 方法 3: U-Boot bootstage
# 在 U-Boot 命令行 (串口按任意键中断):
# => bootstage report
```

### 9.3 时间记录表

| 阶段 | 起始标志 | 结束标志 | 耗时 |
|------|---------|---------|------|
| BootROM + DDR | 上电 | DDR 1332MHz | ~50ms |
| SPL | SPL 开始 | U-Boot 开始 | ~100ms |
| U-Boot | U-Boot 输出 | Starting kernel | ~200-300ms |
| Kernel | Linux version | Freeing unused | ~500-800ms |
| Userspace | init/systemd | login prompt | ~500-1000ms |
| **总计** | | | **~1.5-2.5s** |

### 9.4 板端验证

```bash
# 查看内核启动后的 uptime
uptime
# 预期: 上电后 ~2s 内进入 userspace

# 查看 systemd 启动耗时
systemd-analyze
systemd-analyze blame | head -20
# 列出各服务启动耗时

# 查看内核 dmesg 时间戳
dmesg | head -5
dmesg | tail -5
# 第一个时间戳 [0.000000] = 内核启动时刻
# 最后一个 = 内核完全就绪时刻
```

---

## 十、实验 5：启动优化

### 10.1 优化方向

| 优化手段 | 影响阶段 | 预期收益 | 风险 |
|---------|---------|---------|------|
| 延迟非关键驱动 probe | Kernel | -100~200ms | 功能延迟可用 |
| 内核压缩 (LZ4) | U-Boot→Kernel | -50~100ms | 无 |
| 禁用不需要的驱动 | Kernel | -50~200ms | 需精确知道哪些不需要 |
| 减少 initcall 数量 | Kernel | -50ms | 功能缺失 |
| 精简 rootfs | Userspace | -100~500ms | 依赖分析 |
| 并行 systemd 服务 | Userspace | -200~500ms | 服务依赖冲突 |
| U-Boot 跳过不必要的检测 | U-Boot | -50ms | 可靠性降低 |

### 10.2 实验：延迟非关键驱动

```bash
# 1. 从实验 3 找到最耗时的 initcall
# 假设 rknpu 驱动耗时 50ms，但启动时不需要

# 2. 将驱动改为模块 (在内核 defconfig 中)
# CONFIG_RKNPU=y → CONFIG_RKNPU=m
# 重新编译内核

# 3. 部署，重新测量启动时间
# 预期: 内核启动减少 ~50ms
# rknpu.ko 可在 userspace 按需加载: modprobe rknpu
```

### 10.3 实验：内核压缩优化

```bash
# 检查当前内核压缩方式
grep CONFIG_KERNEL_ kernel-6./arch/arm64/configs/rv1126b_sportcam_defconfig
# 预期: CONFIG_KERNEL_GZIP=y (默认)

# 尝试改为 LZ4 (更快解压)
# 在 defconfig 中:
# CONFIG_KERNEL_GZIP is not set
# CONFIG_KERNEL_LZ4=y

# 重新编译，对比:
# - Image 大小 (LZ4 略大但解压快)
# - 启动时间 (LZ4 解压更快)
```

---

## 十一、U-Boot 编译实验

### 11.1 单独编译 U-Boot

```bash
# 在 SDK 根目录
./build.sh loader    # 编译 U-Boot + loader

# 产物:
# u-boot/uboot.img                    → U-Boot FIT 镜像
# u-boot/rv1126b_spl_loader_*.bin     → 合并 loader (DDR+SPL)
# u-boot/trust.img                    → ATF BL31 + TEE BL32
```

### 11.2 修改 U-Boot 配置

```bash
cd u-boot

# 加载 sportcam 配置
make rv1126b_sportcam_defconfig

# 添加 AMP fragment
make rv1126b_sportcam_defconfig rk-amp

# 进入 menuconfig 修改
make menuconfig
# 例如: 修改 CONFIG_BOOTDELAY 改变启动等待时间

# 保存配置
make savedefconfig
cp defconfig configs/rv1126b_sportcam_defconfig

# 编译
make CROSS_COMPILE=aarch64-none-linux-gnu- all
```

### 11.3 U-Boot 命令行交互

```bash
# 串口连接后，重启板子
# 在 "Hit any key to stop autoboot" 时按任意键

# U-Boot 命令行:
=> printenv           # 打印所有环境变量
=> printenv bootargs  # 查看当前 bootargs
=> printenv bootcmd   # 查看启动命令
=> bdinfo             # 板级信息
=> bootstage report   # 启动各阶段耗时
=> mmc dev 0          # 切换到 eMMC
=> mmc info           # eMMC 信息
=> load mmc 0:3 0x20800000 boot.img  # 加载 boot 分区
=> iminfo 0x20800000  # 查看 FIT 镜像信息

# 修改环境变量 (临时)
=> setenv bootargs "console=ttyFIQ0 root=/dev/mmcblk0p7 rw initcall_debug"
=> saveenv            # 保存到环境变量存储区
=> reset              # 重启验证
```

---

## 十二、思考题

1. RV1126B 的 boot chain 中，DDR Init 为什么必须在 SPL 之前执行？如果 DDR 未初始化，后续阶段能否运行？

2. `CONFIG_USING_KERNEL_DTB_V2=y` 让 U-Boot 复用内核 DTB。这有什么好处？如果 U-Boot 需要在加载内核之前就使用某些硬件（如 eMMC），这些硬件的配置信息从哪里来？

3. FIT 镜像相比直接使用裸 kernel Image 有什么优势？在安全启动场景下，FIT 的签名机制如何防止固件被篡改？

4. AMP 架构中 Linux 和 MCU 通过 mailbox 通信。如果 MCU 需要访问 Linux 管理的内存，需要什么机制？有哪些同步问题？

5. 你的板子启动总耗时约 2 秒，如果要优化到 1 秒以内，你会从哪些阶段入手？每阶段的理论极限是多少？

---

## 十三、踩坑记录

| 日期 | 问题 | 原因 | 解决方案 |
|------|------|------|----------|
| | U-Boot 编译报错找不到 toolchain | 交叉编译器未在 PATH | `export PATH=.../aarch64-none-linux-gnu/bin:$PATH` |
| | 修改 bootargs 后 /proc/cmdline 没变 | 只改了 DTS 没重新打包 boot.img | `./build.sh firmware` 重新打包 |
| | boot.img 写入后板子不启动 | dd 写错分区 | boot 分区是 mmcblk0p3，不是 mmcblk0p1 |
| | 串口无输出 | 波特率不对 | RV1126B 默认 1500000 baud，不是 115200 |
| | initcall_debug 无输出 | loglevel 太低 | bootargs 加 `loglevel=8` |
| | U-Boot 命令行进不去 | bootdelay=0 | 修改 `CONFIG_BOOTDELAY` 或在启动时快速按键 |

---

## 十四、下阶段预告

阶段二：**Linux 设备模型 + 设备树**
- Linux 设备模型三要素 (Bus/Device/Driver)
- 设备树深度解析 (`rv1126b.dtsi` 4048 行)
- 编写第一个 platform driver
- `of_match_table` 匹配机制
- Ftrace 追踪 `driver_probe_device` 全流程

---

## 相关笔记

- [[MOC-嵌入式Linux]] — 学习路线总览
- [[kernel-debug-env]] — 附录A：内核调试环境
- [[rv1126b]] — RV1126B 运动相机项目
