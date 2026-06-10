---
title: H3 U-Boot Pack - Orange Pi PC U-Boot 打包工具
category: embedded-linux/allwinner
tags:
  - uboot
  - allwinner
  - h3
  - orangepi
  - arm
  - debian-packaging
  - embedded-linux
  - sunxi
created: 2026-06-09
status: active
platform: Orange Pi PC (Allwinner H3)
u-boot-version: v2021.04
toolchain: arm-linux-gnueabihf
---

# H3 U-Boot Pack

## 项目概述

一个针对 **Orange Pi PC**（Allwinner H3 SoC）的 U-Boot 构建与打包系统，基于 U-Boot v2021.04 源码，提供一键编译和 Debian 包生成功能，方便在 SD 卡或 eMMC 上部署 U-Boot 固件。

## 技术栈

- **SoC**: Allwinner H3 (ARM Cortex-A7, sun8i)
- **Bootloader**: U-Boot v2021.04
- **交叉编译工具链**: `arm-linux-gnueabihf-gcc`
- **构建系统**: GNU Make（自定义 Makefile 封装）
- **打包格式**: Debian (.deb)，使用 `fakeroot dpkg-deb`
- **设备树**: `sun8i-h3-orangepi-pc`
- **平台**: Linux (WSL2 / Ubuntu)

## 项目结构

```
h3-uboot-pack/
├── build/
│   ├── Makefile              # 主构建脚本
│   ├── config.mak            # 工具链与路径定义
│   ├── color.mk              # 终端彩色输出
│   ├── config/
│   │   └── orangepi_pc_defconfig-not-used
│   └── platform_install.sh   # U-Boot 写入脚本 (dd 命令)
├── src/
│   └── v2021.04/             # U-Boot 源码（含 .config）
├── bin/
│   └── orangepi_pc/
│       ├── u-boot-sunxi-with-spl.bin   # 编译产物
│       ├── .config                      # 完整配置
│       └── orangepi_pc_defconfig        # defconfig
├── DEBIAN/
│   ├── control               # Debian 包元信息
│   ├── install               # 安装文件列表
│   ├── postinst              # 安装后自动写入 U-Boot
│   └── prerm                 # 卸载前脚本
├── ubootdeb/                 # dpkg-deb 打包临时目录
├── uboot-orangepi_pc-v2021.04.deb  # 已生成的 deb 包
└── readme.md
```

## 架构与设计决策

### 1. Makefile 封装层

项目没有直接修改 U-Boot 源码的构建系统，而是在外层用一个 Makefile 做封装，通过 `make -C $(UBOOT_DIR)` 委托给 U-Boot 原生 Kbuild。这样做的好处是：

- 保持 U-Boot 源码纯净，方便升级版本
- 统一入口，简化操作流程
- 自动处理 clean → defconfig → build → install 全流程

### 2. 关键 defconfig 定制

```
CONFIG_MACH_SUN8I_H3=y           # H3 芯片
CONFIG_DRAM_CLK=624              # DRAM 时钟 624MHz
CONFIG_ENV_IS_IN_MMC=y           # 环境变量存储在 MMC/eMMC
CONFIG_ENV_SIZE=0x20000          # 环境变量区 128KB
CONFIG_ENV_OFFSET=0xC0000       # 环境变量起始偏移 768KB
CONFIG_BOOTDELAY=1               # 启动延迟 1 秒
CONFIG_PREBOOT=""                # 清空 preboot（关闭 USB 初始化加速启动）
```

关键改动：
- `CONFIG_ENV_SIZE` 从默认 0x10000 扩大到 0x20000，为更大环境变量留空间
- `CONFIG_ENV_OFFSET=0xC0000`，与 SPL 区域保持安全距离
- 关闭 `CONFIG_VIDEO_DE2` 减小固件体积
- `CONFIG_PREBOOT=""` 跳过 USB 扫描，加快启动

### 3. 固件格式：u-boot-sunxi-with-spl.bin

Allwinner 平台使用 SPL（Secondary Program Loader）引导链：
- SPL 位于 SD 卡偏移 8KB 处（`seek=8`，单位 1024 字节）
- U-Boot 主体紧跟其后
- `u-boot-sunxi-with-spl.bin` 是合并后的单一镜像

### 4. Debian 包自动化

通过 `make deb` 生成 `.deb` 包，安装时 `postinst` 脚本自动：
1. 检测目标块设备（通过 PARTUUID 或 eGON.BT0 签名扫描）
2. 用 `dd` 将 U-Boot 写入 SD 卡/eMMC 的指定偏移

```bash
# 写入逻辑（platform_install.sh）
dd if=/dev/zero of=$2 bs=1k count=1023 seek=1 status=noxfer
dd if=$1/u-boot-sunxi-with-spl.bin of=$2 bs=1024 seek=8 status=noxfer
```

先清零前 1MB（保留第一个 1KB 扇区），再从 8KB 偏移写入 SPL+U-Boot。

### 5. 设备探测机制

`setup_write_uboot_platform` 使用两种方式定位目标设备：
- **方式一**: 解析 `/proc/cmdline` 中的 `ubootpart` 参数获取 PARTUUID
- **方式二**: 扫描所有块设备，查找 `eGON.BT0` SPL 签名（Allwinner SPL 标识）

## 构建与使用

### 依赖安装

```bash
sudo apt install swig gcc-arm-linux-gnueabihf fakeroot
```

### 常用命令

```bash
cd h3-uboot-pack/build

# 清理构建产物
make clean

# 加载默认配置
make defconfig

# 编译 U-Boot（并行 24 线程）
make

# 生成 Debian 安装包
make deb
```

### 安装 U-Boot

```bash
sudo dpkg -i uboot-orangepi_pc-v2021.04.deb
# 安装后自动执行 postinst，将 U-Boot 写入 /dev/mmcblk0
```

### 输出产物

- `bin/orangepi_pc/u-boot-sunxi-with-spl.bin` — 可直接 dd 写入的固件
- `bin/orangepi_pc/.config` — 完整编译配置
- `uboot-orangepi_pc-v2021.04.deb` — Debian 安装包

## 关键学习点

1. **Allwinner H3 启动流程**: BROM → SPL (eGON.BT0) → U-Boot → Kernel，SPL 必须位于 SD 卡 8KB 偏移处
2. **u-boot-sunxi-with-spl.bin** 是 sunxi 平台的标准打包格式，将 SPL 和 U-Boot 合并为单一二进制
3. **Debian 包 + postinst 脚本** 是 Armbian 项目分发 U-Boot 的标准做法，可以实现开箱即用的固件更新
4. **环境变量布局**: H3 的 U-Boot 环境变量通常放在 SPL 和 U-Boot 之后的空闲区域，偏移和大小需要与分区表对齐
5. **Git 提交记录** 显示了项目演进：从基础构建 → 添加 deb 支持 → 调整启动参数（关闭 USB、缩短 bootdelay）→ 尝试 eMMC 环境变量存储

## 相关笔记

- [[h3]] — Allwinner H3 系统构建全栈笔记
- [[h3-dtb-custom]] — H3 DTB 设备树自定义分析
- [[h3-dtb-ref]] — H3 DTB 设备树参考分析
- [[h5]] — Allwinner H5 Crust Firmware 项目
- [[h618]] — H618 TV Box 定制 Linux 系统
- [[orangepi]] — OrangePi PC 嵌入式 Linux 开发
