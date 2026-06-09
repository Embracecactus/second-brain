---
title: Rockchip Linux SDK
category: embedded-linux/rockchip
tags:
  - rockchip
  - embedded-linux
  - sdk
  - rk3588
  - buildroot
  - debian
  - u-boot
  - kernel-6.1
  - arm64
created: 2026-06-09
status: active
---

# Rockchip Linux SDK

## 项目概述

Rockchip Linux SDK 是瑞芯微电子 (Rockchip) 为其 ARM SoC 芯片 (如 RK3588/RK3576/RK3568 等) 提供的完整 Linux 开发套件，涵盖 bootloader、内核、文件系统、多媒体中间件、NPU 推理框架等全栈组件。该项目通过统一的 `build.sh` 构建脚本和 Kconfig 配置系统，支持一键编译并生成可烧写的完整固件。

## 技术栈

| 层级 | 技术/组件 |
|------|----------|
| SoC | Rockchip RK3588 (ARM Cortex-A76 + A55, Mali GPU, 6 TOPS NPU) |
| Bootloader | U-Boot 2017.09 (Rockchip 定制版) |
| Kernel | Linux 6.1 (Rockchip 定制版, 含 RT 补丁选项) |
| Rootfs | Buildroot 2022.08 / Debian 12 (bookworm) |
| 构建系统 | Bash hook 架构 + Kconfig + Makefile |
| 多媒体 | MPP (Media Process Platform), GStreamer, Rockit |
| NPU | RKNN-Toolkit2, RKNPU2 runtime |
| 烧写工具 | Linux_Upgrade_Tool (USB), rkflash.sh |
| 交叉编译 | prebuilts/gcc (aarch64/rockchip-linux-gnu) |

## 架构与关键设计

### SDK 目录结构

```
rk/
├── build.sh              # 主构建入口, 支持 chip/defconfig 参数
├── Makefile              # 顶层 Makefile, 封装 build.sh 调用
├── envsetup.sh           # 环境初始化 (已废弃, 由 build.sh 内部处理)
├── rkflash.sh            # USB 烧写脚本
├── kernel -> kernel-6.1  # 内核源码符号链接
├── u-boot/               # U-Boot 源码 (已编译)
├── buildroot/            # Buildroot 根文件系统构建
├── debian/               # Debian 根文件系统构建脚本
├── rkbin/                # Rockchip 闭源二进制 (DDR/Miniloader)
├── device/rockchip/      # 芯片级配置与构建脚本
│   ├── common/           # 通用构建脚本、Kconfig、hooks
│   └── rk3588/           # RK3588 专用 defconfig 与参数
├── external/             # 第三方组件 (MPP, RKNN, GStreamer 等)
├── app/                  # 应用层 (rkipc, rkadk, lvgl_demo)
├── prebuilts/            # 预编译工具链
├── tools/                # 平台工具 (Linux/Mac/Windows)
└── docs/                 # 开发文档 (中英文)
```

### 构建系统 Hook 架构

`build.sh` 采用 **四阶段 hook 模型**，所有构建逻辑通过可插拔的 shell 脚本实现：

```
init → pre-build → build → post-build
```

- **Hook 目录**: `device/rockchip/common/build-hooks/` 和 `device/rockchip/<chip>/build-hooks/`
- **Post hooks**: `post-hooks/` 用于 rootfs 后处理
- **脚本发现**: `parse_scripts()` 自动扫描并缓存可用命令

### 启动流程

```
DDR Init (rkbin) → Miniloader/U-Boot SPL → U-Boot → Kernel → Rootfs
```

rkbin 中的二进制命名规则: `[chip]_[module]_[feature]_[version].[postfix]`
合并后的 loader 版本格式: `vx.yy.zzz` (DDR版本 + Miniloader版本)

## 核心知识点

### 1. 构建命令体系

```bash
# 选择芯片 + defconfig
./build.sh rk3588:rockchip_defconfig

# 配置 (Kconfig)
make menuconfig

# 全量编译
./build.sh

# 单模块编译
./build.sh kernel    # 编译内核
./build.sh uboot     # 编译 U-Boot
./build.sh rootfs    # 构建根文件系统
./build.sh recovery  # 构建 recovery

# 清理
./build.sh cleanall
./build.sh clean-kernel
```

### 2. 设备分区表

分区定义在 `device/rockchip/rk3588/parameter.txt`，常见分区:
- `MiniLoaderAll.bin` — DDR 初始化 + SPL
- `uboot.img` / `trust.img` — U-Boot + 安全世界
- `boot.img` — Kernel + DTB
- `rootfs.img` — 根文件系统
- `userdata.img` — 用户数据
- `oem.img` — OEM 定制
- `misc.img` — Recovery 控制
- `recovery.img` — OTA 恢复系统

### 3. Debian Rootfs 构建

```bash
# 64位 Debian bookworm 基础系统
RELEASE=bookworm TARGET=desktop ARCH=arm64 ./mk-base-debian.sh

# 叠加 Rockchip 硬件加速层
RELEASE=bookworm ARCH=arm64 ./mk-rootfs.sh

# 生成 ext4 镜像
./mk-image.sh
```

### 4. Buildroot 构建

```bash
source build/envsetup.sh    # 初始化环境
make menuconfig              # 配置
make                         # 构建
bdeploy <pkg>               # 通过 adb 部署单个包到设备
```

### 5. 烧写方式

通过 USB OTG 线连接设备，使用 `rkflash.sh`:

```bash
# 烧写全部固件
./rkflash.sh all

# 烧写单个分区
./rkflash.sh boot
./rkflash.sh rootfs
./rkflash.sh loader [loader_path]
```

底层调用 `Linux_Upgrade_Tool` 的 `ul` (upload loader) 和 `di` (download image) 命令。

## 重要代码片段

### build.sh 环境初始化核心逻辑

```bash
setup_environments()
{
    export RK_SCRIPTS_DIR="$(dirname "$(realpath "$BASH_SOURCE")")"
    export RK_COMMON_DIR="$(realpath "$RK_SCRIPTS_DIR/..")"
    export RK_SDK_DIR="$(realpath "$RK_COMMON_DIR/../../..")"
    export RK_DEVICE_DIR="$RK_SDK_DIR/device/rockchip"
    export RK_CHIPS_DIR="$RK_DEVICE_DIR/.chips"
    export RK_CHIP_DIR="$RK_DEVICE_DIR/.chip"
    export RK_OUTDIR="$RK_SDK_DIR/output"
    export RK_CONFIG="$RK_OUTDIR/.config"
    export RK_FIRMWARE_DIR="$RK_OUTDIR/firmware"
    # ...
}
```

### 交叉编译工具链获取

```bash
get_toolchain()
{
    TC_ARCH="${2/arm64/aarch64}"
    TC_DIR="$RK_SDK_DIR/prebuilts/gcc/linux-x86/$TC_ARCH"
    # RV1126 使用特殊工具链 rockchip830
    if [ "$RK_CHIP_FAMILY" = "rv1126_rv1109" ]; then
        TC_VENDOR=rockchip830
    fi
    GCC="$(find "$TC_DIR" -name "*gcc" | grep -m 1 "/$TC_PATTERN$")"
}
```

## 构建/运行方法

### 首次构建

```bash
cd /home/lijian/project/rk

# 1. 选择目标芯片
./build.sh rk3588:rockchip_defconfig

# 2. (可选) 自定义配置
make menuconfig

# 3. 全量编译
./build.sh

# 4. 固件输出目录
ls rockdev/          # 合并后的固件
ls output/firmware/  # 各分区独立镜像
```

### 依赖环境

```bash
sudo apt-get install binfmt-support qemu-user-static
sudo dpkg -i debian/ubuntu-build-service/packages/*
sudo apt-get install -f
```

### 设备端调试

- 通过 `adb shell` 连接设备
- 使用 `bdeploy <pkg>` 热部署 Buildroot 包
- MPP 测试: `mpi_dec_test`, `mpi_enc_test`

## 相关笔记链接

- [[Embedded Linux 基础]]
- [[U-Boot 启动流程]]
- [[Linux Kernel 编译与配置]]
- [[Buildroot 构建系统]]
- [[RK3588 芯片手册]]
- [[Rockchip MPP 多媒体框架]]
- [[RKNN NPU 推理开发]]
- [[ARM 交叉编译工具链]]
- [[Device Tree 使用指南]]

## 相关笔记

- [[rk3528]] — RK3528 SDK 开发笔记
- [[rk3528-notes]] — RK3528 系统开发笔记
- [[rv1126b]] — RV1126B 运动相机项目
- [[rv1126-notes]] — RV1126B 嵌入式开发笔记
- [[ok1126b-sdk]] — OK1126B SDK 与项目知识库
- [[boardroot-methodology]] — BoardRoot 嵌入式 Linux 厂商适配框架
- [[ffmpeg-learning]] — FFmpeg 学习（RV1126B 多媒体）
