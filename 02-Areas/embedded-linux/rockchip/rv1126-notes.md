---
tags:
  - rv1126
  - buildroot
  - rockchip
  - uboot
  - mcu
  - rtos
  - rtt
  - amp
  - thunder-boot
  - preempt-rt
category: embedded-linux/rockchip
created: 2026-06-09
platform: RV1126B (Quad core Cortex-A53)
sdk_version: rv1126b_linux6.1_release_v1.1.0
kernel: Linux 6.1.118
board: Forlinx OK1126B-S
---

# RV1126B 嵌入式开发笔记

## 项目概述

RV1126B 是 Rockchip 推出的嵌入式 SoC，采用四核 Cortex-A53 架构，支持 Linux 系统运行。本文档基于飞凌嵌入式 OK1126B-S 开发板，整合了 SDK 解压、内核实时补丁、U-Boot 编译、MCU (RT-Thread) 开发与打包等关键开发流程。SDK 版本为 `rv1126b_linux6.1_release_v1.1.0`，内核基于 Linux 6.1.118。

## SDK 解压

SDK 采用分卷压缩，需先合并再解压：

```sh
cat rv1126b_linux6.1_release_v1.1.0_.tar.gz.part-* > rv1126b_linux6.1_release_v1.1.0_.tar.gz
tar xvf rv1126b_linux6.1_release_v1.1.0_.tar.gz
```

## 实时性补丁 (PREEMPT_RT)

基于 Linux 6.1.118 内核，需要按顺序打以下 RT 补丁。若提示 `The next patch would create the file arch/arm/configs/rockchip_rt.config, which already exists! Assume -R? [n]`，**输入 n**。

```sh
patch -p1 < ../docs/Patches/Real-Time-Performance/PREEMPT_RT/kernel-6.1/kernel-6.1.118/0001-patch-6.1.99-rt36-on-rockckip-base-5c295c763974.patch
patch -p1 < ../docs/Patches/Real-Time-Performance/PREEMPT_RT/kernel-6.1/kernel-6.1.118/0002-sched-isolation-remove-HK_FLAG_TICK-for-nohz_full-fo.patch
patch -p1 < ../docs/Patches/Real-Time-Performance/PREEMPT_RT/kernel-6.1/kernel-6.1.118/0003-mm-Kconfig-remove-selection-of-MIGRATION-for-CMA-to-.patch
patch -p1 < ../docs/Patches/Real-Time-Performance/PREEMPT_RT/kernel-6.1/kernel-6.1.118/0004-ARM-configs-add-rockchip_rt.config-for-PREEMPT_RT.patch
patch -p1 < ../docs/Patches/Real-Time-Performance/PREEMPT_RT/kernel-6.1/kernel-6.1.118/0005-arm64-configs-optimize-latency-for-PREEMPT_RT.patch
patch -p1 < ../docs/Patches/Real-Time-Performance/PREEMPT_RT/kernel-6.1/kernel-6.1.118/0006-phy-rockchip-inno-usb2-Fix-DEBUG_LOCKS_WARN_ON-in-ch.patch
```

## 关键文件路径

| 类别 | 路径 |
|------|------|
| 板级 Buildroot defconfig | `device/rockchip/.chips/rv1126b/OK1126B_S_buildroot_defconfig` |
| Linux DTS | `kernel-6.1/arch/arm64/boot/dts/rockchip/myzr-rv1126b-evb.dtsi` |
| Linux defconfig | `kernel-6.1/arch/arm64/configs/OK1126B-S-linux_defconfig` |
| U-Boot defconfig | `u-boot/configs/myzr_rv1126b_defconfig` |
| U-Boot DTS | `u-boot/arch/arm/dts/rv1126b-evb.dts` |
| AMP MCU ITS | `device/rockchip/.chips/rv1126b/amp_mcu.its` |

## U-Boot 编译

### 固件组成

RV1126B U-Boot 编译依赖两组关键 INI 配置文件：

| 维度 | TRUST.ini | MINIALL.ini |
|------|-----------|-------------|
| 所在目录 | `rkbin/RKTRUST/` | `rkbin/RKBOOT/` |
| 核心产物 | `trust.img`（安全启动镜像） | `loader.bin`（MiniLoaderAll.bin） |
| 作用阶段 | BL31/BL32 安全固件打包 | TPL/SPL 基础硬件初始化与加载 |
| 包含内容 | ARM Trusted Firmware (BL31)、OP-TEE (BL32) | DDR 初始化 (TPL)、MiniLoader (SPL)、USB Plug |
| 使用工具 | `trust_merger` | `boot_merger` |
| 启动流程 | BootROM -> MiniLoader -> TRUST -> U-Boot | BootROM -> MiniLoader -> TRUST -> U-Boot |

### 配置文件指定方式

- **MINIALL.ini**：由 `u-boot/make.sh` 中 `${RKBIN}/RKBOOT/${RKCHIP_LOADER}MINIALL.ini` 指定
- **Trust.ini**：由 U-Boot defconfig 中的配置项指定

### 配置文件关系图

```
make.sh
   |
   +-- RKBOOT 指定 --> RV1126MINIALL.ini
   |     +-- DDR 初始化配置
   |     +-- USB plug 配置
   |     +-- SPL loader 配置
   |
   +-- RKTRUST 指定 --> RV1126TOS.ini
         +-- Trust OS 配置
         +-- TEE OS 配置
         +-- MCU 镜像配置
```

### 常见启动场景配置

| 场景 | defconfig | Loader INI | Trust INI |
|------|-----------|------------|-----------|
| 常规 eMMC 启动 | `rv1126_defconfig` | `RV1126MINIALL.ini` | `RV1126TOS.ini` |
| 快速启动 (Thunder Boot) | `rv1126-emmc-tb.config` | `RV1126MINIALL_EMMC_TB.ini` | `RV1126TOS_TB.ini` |
| SPI Nor Flash 启动 | `rv1126-spi-nor.config` | `RV1126MINIALL_SPI_NOR.ini` | `RV1126TOS_SPI_NOR.ini` |

### 修改 defconfig 添加自定义 INI

在 `u-boot/configs/myzr_rv1126b_defconfig` 中添加：

```sh
CONFIG_LOADER_INI="RV1126BMINIALL_FASTBOOT.ini"
CONFIG_TRUST_INI="RV1126BTRUST_MCU.ini"
```

### 编译与验证

```sh
./build.sh uboot

# 验证配置文件是否生效
grep "pack.*okay" ./make.sh.log
# 期望输出：
# pack loader okay! Input: rkbin/RKBOOT/RV1126MINIALL.ini
# pack uboot.img okay! Input: rkbin/RKTRUST/RV1126TOS.ini
```

### Thunder Boot 快速启动关键配置

```sh
CONFIG_SPL_KERNEL_BOOT=y          # 开启快速开机
CONFIG_SPL_BLK_READ_PREPARE=y     # 开启预加载
CONFIG_SPL_MISC_DECOMPRESS=y      # 开启解压功能
CONFIG_ROCKCHIP_THUNDER_BOOT=y    # Thunder Boot 支持
```

> [!note] 快速启动模式下，uboot 分区实际打包了 MCU 镜像 + Trust 镜像 + U-Boot 镜像，这些都由 SPL 加载。

## MCU (RT-Thread) 编译

### 环境准备

```sh
sudo apt install scons
```

### 编译流程

```sh
cd rtos/bsp/rockchip/rv1126b-mcu
scons --useconfig=board/rv1126b_evb1/defconfig
scons --genconfig
scons -j16
```

### 打包方式

**方法一：独立 FIT 镜像 (amp.img)**

根据 `Image/amp.its` 打包：

```sh
./mkimage.sh amp
# 输出文件为 Image/amp.img
file -L Image/amp.img
# Image/amp.img: Device Tree Blob version 17, size=1024, boot CPU=0
```

可通过 Rockchip 开发工具将 `amp.img` 烧录到开发板的 `amp` 分区。

**方法二：打包到 U-Boot 镜像**

```sh
cp rtthread.bin ../../../../rkbin/bin/rv11/rtthread.bin
```

### amp_mcu.its 关键参数

```
hpmcu 节点:
  load    = <0x48c02000>    # MCU 加载地址
  entry   = <0x48c02200>    # MCU 入口地址
  arch    = "arm"           # 实际为 RISC-V，但 U-Boot 仅接受 arm/arm64
  size    = <0x0003a000>    # 编译大小限制
  core    = "mcu"           # mcu 或 ap

share 节点:
  rpmsg_base = <0x48c3c000> # RPMsg 通信基地址
  rpmsg_size = <0x00040000> # RPMsg 共享内存大小 (256KB)
```

> [!warning] arch 字段虽然实际是 RISC-V 架构，但必须写为 `"arm"`，因为 U-Boot 的 FIT 解析只接受 arm/arm64。

### 检查 MCU 代码区域

```sh
devmem 0x48c02000    # 检查加载地址
devmem 0x48c02200    # 检查入口地址
```

## 设备树配置 (DTS)

OK1126B-S 开发板 DTS 包含以下子文件：

```c
#include "OK1126B-S-common.dtsi"
#include "OK1126B-S-camera.dtsi"
#include "OK1126B-S-display.dtsi"
#include "rv1126b-amp.dtsi"

/ {
    model = "Forlinx OK1126B-S Board";
    processor = "Rockchip RV1126B (Quad core Cortex A53)";
    compatible = "rockchip,rv1126b-evb1-v10", "rockchip,rv1126b";

    chosen {
        bootargs = "earlycon=uart8250,mmio32,0x20810000 console=ttyFIQ0 rw "
                   "root=PARTUUID=614e0000-0000 rootfstype=ext4 rootwait "
                   "snd_soc_core.prealloc_buffer_size_kbytes=16 coherent_pool=32K";
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

## 问题排查

- **fiq_debugger 波特率不一致**：如果 fiq_debugger 中波特率和 U-Boot 不一致，可能导致 MCU 加载失败。RV1126B 仅支持 `115200` 和 `1500000` 两种波特率。

## 相关链接

- Rockchip SDK 文档：`docs/Patches/Real-Time-Performance/PREEMPT_RT/`
- MCU 源码：`rtos/bsp/rockchip/rv1126b-mcu`
- RKBin 工具：`rkbin/` (包含 TRUST/BOOT INI 配置和预编译二进制)
- 编译入口脚本：`build.sh` (顶层)、`u-boot/make.sh` (U-Boot)
