---
tags:
  - embedded-linux
  - allwinner
  - arm
  - orangepi
  - uboot
  - linux-kernel
  - ubuntu
  - sherpa-onnx
  - voice-assistant
category: embedded-linux/allwinner
created: 2026-06-09
status: in-progress
aliases:
  - OrangePi PC
  - Orange Pi 开发板
---

# OrangePi PC 嵌入式 Linux 开发项目

## 项目概述

这是一个针对 **Orange Pi PC** 开发板的完整嵌入式 Linux 系统构建项目，基于 Allwinner H3 (sun8i) SoC，涵盖从 U-Boot 引导加载程序、Linux 内核编译、Ubuntu 22.04 rootfs 制作到最终 SD 卡镜像生成的全流程，并集成了 sherpa-onnx 语音识别引擎用于语音助手应用开发。

## 技术栈

- **SoC**: Allwinner H3 (ARM Cortex-A7, armv7)
- **Bootloader**: U-Boot 2024.01 (sunxi 平台)
- **Linux Kernel**: linux-orange-pi-6.11
- **文件系统**: Ubuntu 22.04 base (armhf)
- **交叉编译工具链**: gcc-arm-11.2-2022.02 (arm-none-linux-gnueabihf)
- **镜像生成工具**: genimage
- **AI 推理引擎**: sherpa-onnx (ONNX Runtime 语音识别)
- **构建系统**: Makefile, CMake
- **版本管理**: Git + Git Submodule + Git LFS
- **代码托管**: 阿里云 CodeUp

## 项目架构

```
orangepi-pc/
├── 1.uboot/           # U-Boot 引导加载程序 (submodule)
│   └── u-boot-orangepi/   # U-Boot 2024.01 源码 (已编译)
├── 2.kernel/          # Linux 内核
│   └── linux-orange-pi-6.11/  # 内核 6.11 源码 (submodule)
├── 3.mkubuntu-2204/   # Ubuntu 22.04 rootfs 制作 (submodule)
│   ├── Makefile       # 构建流程编排
│   ├── setup_chroot.sh # chroot 环境配置脚本
│   ├── run_chroot.sh  # chroot 执行脚本
│   ├── mount.sh       # 文件系统挂载/卸载
│   ├── bootscr/       # U-Boot 启动脚本 (boot.cmd/boot.scr)
│   ├── dtb/           # 设备树二进制文件 (sun8i-h3 系列)
│   ├── kernel/        # 内核镜像
│   ├── uboot/         # U-Boot 二进制文件
│   └── rootfs.ext4    # 生成的 ext4 根文件系统 (1.8GB)
├── 4.app/             # 应用层
│   └── voice_assistant1/  # 语音助手 (backend + frontend)
├── 5.mkimage/         # SD 卡镜像生成 (submodule)
│   ├── genimage/      # genimage 工具源码
│   ├── input/         # 输入文件 (u-boot-sunxi-with-spl.bin, boot.scr, rootfs.ext4)
│   ├── images/        # 输出镜像 (my.img)
│   └── root/genimage.cfg  # 镜像分区配置
├── 6.sherpa-onnx/     # sherpa-onnx 语音识别引擎 (submodule)
└── tools/             # 交叉编译工具链 (Git LFS)
    └── gcc-arm-11.2-2022.02-x86_64-arm-none-linux-gnueabihf/
```

## 关键设计决策

### 1. 分阶段构建流程
项目采用明确的编号目录结构 (1-6)，将嵌入式系统构建分解为独立阶段：U-Boot -> Kernel -> Rootfs -> App -> Image。每个阶段可独立编译调试，降低耦合度。

### 2. chroot + QEMU 用户态模拟
在 x86 宿主机上通过 `qemu-arm-static` 实现 ARM 用户态模拟，配合 `chroot` 直接在宿主机上运行 armhf 的 apt 包管理器安装软件，避免了完整的 QEMU 系统模拟开销。

```bash
# chroot 进入 ARM rootfs 的核心流程
sudo mount -t proc /proc ${TEMP}/proc
sudo mount -t sysfs /sys ${TEMP}/sys
sudo mount -o bind /dev ${TEMP}/dev
sudo mount -o bind /dev/pts ${TEMP}/dev/pts
sudo DEBIAN_FRONTEND=noninteractive chroot ${TEMP} /bin/bash -c "..."
```

### 3. DTB Overlay 动态加载
启动脚本支持通过环境变量 `overlays` 动态加载设备树 overlay，实现硬件功能的灵活配置而无需重新编译内核：

```bash
# boot.cmd 中的 overlay 加载逻辑
if test -n "${overlays}"; then
    for overlay in ${overlays}; do
        setenv overlay_path "/boot/dtb/overlays/sun8i-h3-${overlay}.dtbo"
        load mmc 0:2 ${ramdisk_addr_r} ${overlay_path}
        fdt apply ${ramdisk_addr_r}
    done
fi
```

### 4. genimage 统一镜像生成
使用 genimage 工具将 U-Boot SPL、boot.scr 和 rootfs.ext4 打包为统一的 SD 卡镜像，分区布局为：
- **offset 8K**: u-boot-sunxi-with-spl.bin (FEL 模式兼容)
- **Partition 1 (FAT)**: boot.scr + u-boot-sunxi-with-spl.bin
- **Partition 2 (ext4)**: rootfs (2048MB)

### 5. sherpa-onnx 跨平台语音识别
集成 sherpa-onnx 作为 AI 推理引擎，通过 CMake 交叉编译为 ARM 平台，支持 ALSA 音频采集和 WebSocket 通信。

## 关键配置参数

| 参数 | 值 | 说明 |
|------|-----|------|
| 架构 | armhf (armv7) | 32 位 ARM 硬浮点 |
| SoC | sun8i-h3 | Allwinner H3 |
| 内核版本 | 6.11 | linux-orange-pi 分支 |
| U-Boot 版本 | 2024.01 | sunxi 平台 |
| rootfs 大小 | 1850MB (dd) / 2048MB (partition) | ext4 文件系统 |
| 默认用户 | lianxipi / 123456 | sudo 权限 |
| 网络 | 静态 IP 192.168.1.101 | eth0, DHCP 也已配置 |
| 串口 | ttyS0, 115200 | 调试串口 |
| APT 镜像 | mirrors.tuna.tsinghua.edu.cn | 清华大学镜像源 |

## 构建指南

### 前置依赖
```bash
sudo apt-get install -y qemu-user-static genimage mkimage
```

### 制作 rootfs
```bash
cd 3.mkubuntu-2204
make all    # 完整流程: 解压 -> 拷贝文件 -> chroot 安装软件 -> 生成 ext4
```

### 生成 SD 卡镜像
```bash
cd 5.mkimage
# 将 u-boot-sunxi-with-spl.bin, boot.scr, rootfs.ext4 放入 input/
./genimage/genimage --input input --output images --config root/genimage.cfg
# 输出: images/my.img
```

### 烧录到 SD 卡
```bash
sudo dd if=5.mkimage/images/my.img of=/dev/sdX bs=1M status=progress
```

### 交叉编译 sherpa-onnx
```bash
cd 6.sherpa-onnx
./build-arm-linux-gnueabihf.sh
```

## 关键学习与洞察

1. **Allwinner H3 启动流程**: SPL -> U-Boot -> boot.scr -> kernel + DTB -> rootfs，其中 SPL 嵌入在 u-boot-sunxi-with-spl.bin 中，偏移 8K 写入是为了兼容 FEL 救砖模式。

2. **Git LFS 管理大文件**: 交叉编译工具链 (106MB) 和内核源码压缩包 (296MB) 使用 Git LFS 管理，避免仓库膨胀。

3. **chroot 环境清理**: 使用 trap 捕获错误信号，确保 chroot 失败时自动卸载 proc/sys/dev，防止挂载点泄漏。

4. **systemd 服务优化**: 禁用了 ModemManager、NetworkManager-wait-online 等不必要的服务，减少嵌入式系统的启动时间和资源占用。

5. **设备树 overlay 机制**: sun8i-h3 系列支持丰富的 DTB overlay（dtb/overlays/ 目录），可通过修改 orangepipcEnv.txt 中的 overlays 变量启用 I2C、SPI、UART 等外设。

## 相关笔记

- [[h3]] — Allwinner H3 系统构建全栈笔记
- [[h3-dtb-custom]] — H3 DTB 设备树自定义分析
- [[h3-dtb-ref]] — H3 DTB 设备树参考分析
- [[h3-uboot-pack]] — H3 U-Boot 打包工具
- [[h5]] — Allwinner H5 Crust Firmware 项目
- [[h618]] — H618 TV Box 定制 Linux 系统
- [[h618-buildroot]] — H618 完整开发笔记
- [[temperature-sensor]] — MAX31865 温度传感器驱动（同为 Orange Pi 平台）
