---
tags:
  - rv1126
  - myzr
  - board
  - rockchip
  - embedded-linux
  - uboot
  - buildroot
  - mcu
  - amp
category: embedded-linux/rockchip
created: 2026-06-09
board: Forlinx OK1126B-S / MYZR-RV1126B
soc: Rockchip RV1126B
kernel: Linux 6.1
sdk_version: rv1126b_linux6.1_release_v1.1.0_20250920_sync20251107
---

# MYZR-RV1126B 开发资料全集

## 一、项目概述

### 1.1 芯片与板卡信息

| 项目 | 详情 |
|------|------|
| SoC | Rockchip RV1126B |
| CPU 架构 | Quad-core ARM Cortex-A53 (ARM64) |
| MCU | RISC-V (用于 AMP 非对称多处理) |
| 适用场景 | IPC 网络摄像机、运动相机、智能门铃、智能玩具 |
| 参考板 | Forlinx OK1126B-S |
| SDK 版本 | `rv1126b_linux6.1_release_v1.1.0_20250920_sync20251107` |
| 内核版本 | Linux 6.1.118 |
| 模块型号 | MYZR-RV1126B-LB221-REVA |

### 1.2 资料目录结构

```
MYZR-RV1126/
├── 1.通用资料/
│   ├── 1.1-文档/              # FAQ、Camera IQ 文件
│   ├── 1.2-固件/              # Buildroot / Debian 预编译固件
│   └── 1.3-工具/              # ADB、PuTTY、WinSCP
├── 2.硬件资料/
│   ├── 2.1-原理和PCB/         # AD / Cadence / DXF 格式
│   ├── 2.2-生产相关/
│   ├── 2.3-设计指导/
│   ├── 2.4-模块参考电路/      # 明远模块设计库 (百度网盘)
│   └── 2.5-管脚复用/          # MYZR-RV1126B-LB221-REVA-OPEN.pdf
├── 3.软件资料/
│   ├── 3.2-工具/              # RKDevTool v3.15、DriverAssitant v5.13
│   └── 3.3-开发指导/          # 刷机/启动/编译/测试手册 (PDF)
└── 4.生产指导/
```

---

## 二、关键知识点

### 2.1 SDK 解压

```bash
cat rv1126b_linux6.1_release_v1.1.0_.tar.gz.part-* > rv1126b_linux6.1_release_v1.1.0_.tar.gz
tar xvf rv1126b_linux6.1_release_v1.1.0_.tar.gz
```

SDK 分卷文件位于 `3.软件资料/rv1126b-amp/`，共 6 个 part 文件 (`part_aa` ~ `part_af`)。

### 2.2 SDK 关键路径

| 文件 | 路径 |
|------|------|
| 板级 defconfig | `device/rockchip/.chips/rv1126b/OK1126B_S_buildroot_defconfig` |
| Linux DTS | `kernel-6.1/arch/arm64/boot/dts/rockchip/myzr-rv1126b-evb.dtsi` |
| Linux defconfig | `kernel-6.1/arch/arm64/configs/OK1126B-S-linux_defconfig` |
| U-Boot defconfig | `u-boot/configs/myzr_rv1126b_defconfig` |
| U-Boot DTS | `u-boot/arch/arm/dts/rv1126b-evb.dts` |
| parameter.txt | `device/rockchip/.chips/rv1126b/parameter.txt` |
| MCU ITS | `device/rockchip/.chips/rv1126b/amp_mcu.its` |

### 2.3 启动流程

```
BootROM → MiniLoaderAll.bin (TPL/SPL) → trust.img (BL31/BL32) → U-Boot → Kernel
```

| 阶段 | 固件 | INI 配置文件 |
|------|------|-------------|
| TPL/SPL | `MiniLoaderAll.bin` | `rkbin/RKBOOT/RV1126BMINIALL.ini` |
| Trust (BL31/BL32) | `trust.img` | `rkbin/RKTRUST/RV1126BTRUST.ini` |
| U-Boot | `uboot.img` | - |

### 2.4 烧录工具

- **RKDevTool v3.15** (Windows) — 瑞芯微官方刷机工具
- **DriverAssitant v5.13** — USB 驱动安装工具
- 支持 Loader 模式和 Maskrom 模式烧录

RKDevTool config.cfg 中的烧录分区映射 (Linux 配置):

```
Loader  → MiniLoaderAll.bin
Parameter → parameter.txt
Uboot   → uboot.img
Misc    → misc.img
Boot    → boot.img
Rootfs  → rootfs.img
```

---

## 三、技术细节

### 3.1 parameter.txt 分区表

默认 Buildroot 分区布局:

```
FIRMWARE_VER: 1.0
MACHINE_MODEL: RV1126B
MACHINE_ID: 007
MAGIC: 0x5041524B
TYPE: GPT

CMDLINE: mtdparts=:
  0x00002000@0x00004000(uboot),      # 4 MB
  0x00002000@0x00006000(misc),       # 4 MB
  0x00020000@0x00008000(boot),       # 64 MB
  0x00040000@0x00028000(recovery),   # 128 MB
  0x00010000@0x00068000(backup),     # 32 MB
  0x00c00000@0x00078000(rootfs),     # 6 GB
  0x00040000@0x00c78000(oem),        # 128 MB
  -@0x00cb8000(userdata:grow)        # 剩余空间
```

**Robot 版本 (带 AMP MCU)** 在 `boot` 和 `recovery` 之间插入了 `amp` 分区 (4 MB)。

UUID 配置:
```
uuid:rootfs=614e0000-0000-4b53-8000-1d28000054a9
uuid:boot=7A3F0000-0000-446A-8000-702F00006273
```

### 3.2 U-Boot 配置体系

#### INI 文件指定方式

| 维度 | TRUST.ini | MINIALL.ini |
|------|-----------|-------------|
| 目录 | `rkbin/RKTRUST/` | `rkbin/RKBOOT/` |
| 产物 | `trust.img` (安全启动镜像) | `loader.bin` (MiniLoaderAll.bin) |
| 作用 | BL31/BL32 安全固件打包 | TPL/SPL 硬件初始化 |
| 工具 | `trust_merger` | `boot_merger` |

#### 编译命令

```bash
./build.sh uboot
```

#### 自定义 Trust INI

在 `u-boot/configs/myzr_rv1126b_defconfig` 中添加:
```
CONFIG_TRUST_INI="RV1126BTRUST_MCU.ini"
```

### 3.3 MCU (AMP) 编译

MCU 使用 RT-Thread RTOS，运行在 RISC-V 核心上。

```bash
# 安装构建工具
sudo apt install scons

# 编译 MCU
cd rtos/bsp/rockchip/rv1126b-mcu
scons --useconfig=board/rv1126b_evb1/defconfig
scons --genconfig
scons -j16
```

#### 打包方式

**方法 1: 独立 amp.img**
```bash
cp rtthread.bin ../../../../rkbin/bin/rv11/rtthread.bin
./mkimage.sh amp
# 输出: amp.img → 烧录到 amp 分区
```

**方法 2: 打包到 uboot.img**
```bash
cp rtthread.bin ../../../../rkbin/bin/rv11/rtthread.bin
# 通过 amp_mcu.its 自动集成
```

#### amp_mcu.its 关键配置

```dts
hpmcu {
    load  = <0x48c02000>;    // MCU 加载地址
    entry = <0x48c02200>;    // MCU 入口地址
    arch  = "arm";           // 实际是 RISC-V，但 U-Boot 只接受 arm/arm64
    compile {
        sys      = "rtt";    // RT-Thread
        core     = "mcu";    // MCU 核心
    };
};

share {
    rpmsg_base = <0x48c3c000>;  // RPMSG 共享内存基地址
    rpmsg_size = <0x00040000>;  // RPMSG 共享内存大小 (256 KB)
};
```

MCU 加载宏配置:
```
RK_AMP=y
RK_AMP_RISCV=y
RK_AMP_FIT_ITS="amp_mcu.its"
RK_RECOVERY=y
RK_PARAMETER="parameter-robot.txt"
```

### 3.4 实时性补丁 (PREEMPT_RT)

针对 Linux 6.1.118 内核:

```bash
cd kernel-6.1

patch -p1 < ../docs/Patches/Real-Time-Performance/PREEMPT_RT/kernel-6.1/kernel-6.1.118/0001-patch-6.1.99-rt36-on-rockckip-base-5c295c763974.patch
patch -p1 < ../docs/Patches/Real-Time-Performance/PREEMPT_RT/kernel-6.1/kernel-6.1.118/0002-sched-isolation-remove-HK_FLAG_TICK-for-nohz_full-fo.patch
patch -p1 < ../docs/Patches/Real-Time-Performance/PREEMPT_RT/kernel-6.1/kernel-6.1.118/0003-mm-Kconfig-remove-selection-of-MIGRATION-for-CMA-to-.patch
patch -p1 < ../docs/Patches/Real-Time-Performance/PREEMPT_RT/kernel-6.1/kernel-6.1.118/0004-ARM-configs-add-rockchip_rt.config-for-PREEMPT_RT.patch
patch -p1 < ../docs/Patches/Real-Time-Performance/PREEMPT_RT/kernel-6.1/kernel-6.1.118/0005-arm64-configs-optimize-latency-for-PREEMPT_RT.patch
patch -p1 < ../docs/Patches/Real-Time-Performance/PREEMPT_RT/kernel-6.1/kernel-6.1.118/0006-phy-rockchip-inno-usb2-Fix-DEBUG_LOCKS_WARN_ON-in-ch.patch
```

注意: 如果提示 `rockchip_rt.config already exists`，输入 `n` 跳过。

### 3.5 设备树结构

```dts
#include "OK1126B-S-common.dtsi"
#include "OK1126B-S-camera.dtsi"
#include "OK1126B-S-display.dtsi"
#include "rv1126b-amp.dtsi"

/ {
    model = "Forlinx OK1126B-S Board";
    processor = "Rockchip RV1126B (Quad core Cortex A53)";
    compatible = "rockchip,rv1126b-evb1-v10", "rockchip,rv1126b";

    chosen {
        bootargs = "earlycon=uart8250,mmio32,0x20810000 console=ttyFIQ0 rw root=PARTUUID=614e0000-0000 rootfstype=ext4 rootwait snd_soc_core.prealloc_buffer_size_kbytes=16 coherent_pool=32K";
    };
};

&fiq_debugger {
    compatible = "rockchip,fiq-debugger";
    rockchip,serial-id = <0>;
    rockchip,baudrate = <1500000>;  /* Only 115200 and 1500000 */
    interrupts = <GIC_SPI 240 IRQ_TYPE_LEVEL_HIGH>;
    status = "okay";
};
```

### 3.6 Camera ISP 配置 (OV5695)

OV5695 传感器参数 (2592x1944 分辨率):

```json
{
    "sensor_calib": {
        "resolution": { "width": 2592, "height": 1944 },
        "Gain2Reg": {
            "GainMode": "EXPGAIN_MODE_LINEAR",
            "GainRange": [1, 15.5, 16, 0, 1, 16, 248],
            "GainRegDBUnit": 0.3
        },
        "CISGainSet": {
            "CISAgainRange": { "Min": 1, "Max": 15.5 },
            "CISDgainRange": { "Min": 1, "Max": 1 },
            "CISIspDgainRange": { "Min": 1, "Max": 1 }
        }
    }
}
```

将 `ov5695_default_default.json` 和 `ov5695_dphy2-ov5695_default.json` 复制到 `buildroot/output/rockchip_rv1126b/target/etc/iqfiles` 目录后重新编译即可。

---

## 四、代码/配置片段

### 4.1 运动相机板级 defconfig

```
# device/rockchip/.chips/rv1126b/rv1126b_sportcam_defconfig
RK_ROOTFS_PREBUILT_TOOLS=y
RK_WIFIBT_CHIP="RTL8821CS"
RK_UBOOT_SPL=y
RK_KERNEL_CFG="rv1126b_sportcam_linux"
RK_KERNEL_DTS_NAME="rv1126b-sportcam"
RK_USE_FIT_IMG=y
```

### 4.2 运动相机 U-Boot defconfig 核心配置

```
CONFIG_ARM=y
CONFIG_ARCH_ROCKCHIP=y
CONFIG_ROCKCHIP_RV1126B=y
CONFIG_DEFAULT_DEVICE_TREE="rv1126b-sportcam"

# 调试串口
CONFIG_DEBUG_UART=y
CONFIG_DEBUG_UART_BASE=0x20810000
CONFIG_DEBUG_UART_CLOCK=24000000
CONFIG_BAUDRATE=1500000

# 存储 (eMMC + SD)
CONFIG_MMC_DW=y
CONFIG_MMC_DW_ROCKCHIP=y

# 显示 (LCD 预览)
CONFIG_DM_VIDEO=y
CONFIG_DRM_ROCKCHIP=y
CONFIG_DRM_ROCKCHIP_RGB=y

# USB Device (数据传输/充电)
CONFIG_USB_GADGET=y
CONFIG_USB_DWC3_GADGET=y

# WiFi 图传
CONFIG_DM_ETH=y
CONFIG_GMAC_ROCKCHIP=y

# 裁剪不需要的功能
# CONFIG_AVB_LIBAVB is not set
# CONFIG_SPL_AB is not set
# CONFIG_CMD_NFS is not set
```

### 4.3 WiFi/BT 芯片选项

| 芯片型号 | 说明 |
|---------|------|
| `RK960` | Rockchip 自有 WiFi |
| `RTL8821CS` | Realtek 8821CS (运动相机推荐) |
| `RTL8723DS` | Realtek 8723DS |
| `AP6212` | Ampak AP6212 (BCM43438) |
| `AP6255` | Ampak AP6255 (BCM43455) |

### 4.4 build.sh 构建命令

```bash
# 选择板子配置
./build.sh lunch rv1126b_sportcam_defconfig

# 完整构建
./build.sh

# 单独编译
./build.sh uboot        # 仅 U-Boot
./build.sh kernel       # 仅内核
./build.sh dtb          # 仅设备树

# 清理
./build.sh clean
./build.sh cleanall
```

### 4.5 NPU 未使能修复

在 `myzr-rv1126b-evb1.dtsi` 中将 `&pwm0_8ch_0` 节点的 `status` 从 `"disabled"` 改为 `"okay"`:

```dts
&pwm0_8ch_0 {
    status = "okay";
};
```

### 4.6 FIQ Debugger 波特率注意事项

fiq_debugger 中波特率必须与 U-Boot 一致，否则可能导致驱动加载失败。支持的波特率: `115200` 和 `1500000`。

### 4.7 MCU 内存检查

```bash
devmem 0x48c02000   # 检查 MCU 加载区域
devmem 0x48c02200   # 检查 MCU 入口区域
```

---

## 五、RKDevTool 烧录配置

### 5.1 Linux 标准烧录配置 (linux.cfg)

```
Loader    → image/MiniLoaderAll.bin
Parameter → image/parameter.txt
Uboot     → image/uboot.img
Misc      → image/misc.img
Boot      → image/boot.img
Rootfs    → image/rootfs.img
```

### 5.2 扩展启动配置 (linux_extboot.cfg)

```
Loader    → image/MiniLoaderAll.bin
Parameter → image/parameter.txt
Uboot     → image/uboot.img
Boot      → image/extboot.img
Rootfs    → image/rootfs.img
```

### 5.3 config.ini 关键设置

```ini
SUPPORTLOWUSB=true        # 支持低速 USB
RESET_AFTER_DOWNLOAD=TRUE # 烧录后自动重启
FW_NOT_CHECK=TRUE         # 跳过固件版本检查
RB_CHECK_OFF=TRUE         # 跳过 RB 检查
AUTO=true                 # 自动开始烧录
```

---

## 六、构建产物

编译成功后输出目录:

```
output/
├── .config                    # 当前板级配置
├── firmware/
│   ├── u-boot.bin             # U-Boot 二进制
│   ├── u-boot-spl.bin         # SPL 二进制
│   ├── trust.bin              # ATF/Trust 固件
│   ├── boot.img               # 启动镜像 (kernel + DTB)
│   ├── rootfs.img             # 根文件系统
│   ├── misc.img               # Misc 分区
│   ├── amp.img                # AMP MCU 固件
│   └── update.img             # 完整升级包
└── rockdev/Image/             # 可烧录镜像目录
```

---

## 七、故障排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 串口无输出 | fiq_debugger 波特率与 U-Boot 不一致 | 统一为 1500000 |
| NPU 未使能 | `pwm0_8ch_0` 节点 disabled | 修改 dtsi 设为 okay |
| Camera 不显示 | 缺少 ISP IQ 文件 | 复制 ov5695 json 到 iqfiles 目录 |
| MCU 加载失败 | amp_mcu.its 配置错误或 rtthread.bin 未正确拷贝 | 检查 load/entry 地址和 bin 路径 |
| 分区重叠 | parameter.txt 偏移计算错误 | 确保当前起始 = 上一起始 + 上一大小 |
| rootfs 挂载失败 | UUID 不匹配 | 检查 parameter.txt 中 uuid 与 bootargs 中 PARTUUID |
| PREEMPT_RT 补丁冲突 | rockchip_rt.config 已存在 | 输入 n 跳过覆盖 |

---

## 八、相关链接

- 明远模块设计库 (百度网盘): `https://pan.baidu.com/s/1zS4ZEl99PzyoD3dJ-ByhJg` (提取码: MYDF)
- RKDevTool 用户手册: `3.软件资料/3.2-工具/RKDevTool_Release_v3.15/开发工具用户手册_v1.0.pdf`
- 刷机手册: `3.软件资料/3.3-开发指导/刷机手册-rv1126b.pdf`
- 启动手册: `3.软件资料/3.3-开发指导/启动手册-rv1126b.pdf`
- 编译手册: `3.软件资料/3.3-开发指导/编译手册-rv1126b.pdf`
- 测试手册: `3.软件资料/3.3-开发指导/测试手册-rv1126b.pdf`
- DSI 5 寸屏连接图: `3.软件资料/3.3-开发指导/RV1126B_DSI_5寸屏连接图.pdf`
- 管脚复用表: `2.硬件资料/2.5-管脚复用/MYZR-RV1126B-LB221-REVA-OPEN.pdf`
- LubanCat-RK 硬件资料集合: `2.硬件资料/LubanCat-RK系列硬件资料集合_20260324/`
