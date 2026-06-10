---
tags:
  - allwinner
  - h618
  - uboot
  - kernel
  - rootfs
  - atf
  - docker
  - retroarch
  - armbian
  - genimage
category: embedded-linux/allwinner
created: 2026-06-09
updated: 2026-06-09
status: active
---

# H618 AI电视盒子 BSP 完整文档

## 项目概述

基于全志 H618 SoC 的 AI TV Box 完整 BSP 开发文档，涵盖从 ARM Trusted Firmware、U-Boot、Linux Kernel 到 Ubuntu rootfs 的全链路构建流程。包含系统级构建（ATF → U-Boot → Kernel → Rootfs → Pack）和应用级配置（Docker、RetroArch 游戏模拟、桌面环境、VNC 远程、USB 摄像头推流）。

## 技术栈

| 层级 | 技术 |
|------|------|
| SoC | Allwinner H618 (sun50i, ARM Cortex-A53, AArch64) |
| ATF | ARM Trusted Firmware BL31 (sun50i_h616) |
| U-Boot | v2026.01 (sun50i-h618-ai-tv_defconfig) |
| Kernel | Linux v6.18 (含 Armbian 社区补丁) |
| Rootfs | Ubuntu Jammy 22.04 arm64 (debootstrap) |
| 工具链 | gcc-arm-11.2 (aarch64-none-linux-gnu) |
| 打包 | genimage |
| 补丁源 | Armbian build v26.2.0-trunk.239 |
| 应用层 | Docker, CasaOS, XFCE4, VNC, RetroArch |

## 构建流程

### 1. 环境搭建

```bash
# 下载交叉编译工具链
# https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads
cd tools
tar -xvf gcc-arm-11.2-2022.02-x86_64-aarch64-none-linux-gnu.tar.xz
export PATH=$PWD/tools/gcc-arm-11.2-2022.02-x86_64-aarch64-none-linux-gnu/bin:$PATH
```

### 2. ARM Trusted Firmware

```bash
git clone https://github.com/ARM-software/arm-trusted-firmware.git 0.arm-trusted-firmware
make CROSS_COMPILE=aarch64-none-linux-gnu- PLAT=sun50i_h616 DEBUG=1 bl31
export BL31=$PWD/build/sun50i_h616/debug/bl31.bin
```

### 3. U-Boot

```bash
git clone -b v2026.01 https://github.com/u-boot/u-boot.git --depth=1 1.uboot
# 新建 defconfig 和 DTS
touch 1.uboot/configs/sun50i-h618-ai-tv_defconfig
touch 1.uboot/dts/upstream/src/arm64/allwinner/sun50i-h618-ai-tv.dts
# 编译
make ARCH=arm CROSS_COMPILE=aarch64-none-linux-gnu- sun50i-h618-ai-tv_defconfig
make ARCH=arm CROSS_COMPILE=aarch64-none-linux-gnu- -j32
```

### 4. Linux Kernel

```bash
git clone -b v6.18 https://github.com/torvalds/linux.git --depth=1 2.kernel
# 拉取 Armbian 补丁
git clone -b v26.2.0-trunk.239 https://github.com/armbian/build.git --depth=1 build
# 打补丁 (参考 kernel-6.18-patch.sh)
# 新建 defconfig 和 DTS
touch 2.kernel/arch/arm64/configs/sun50i-h618-ai-tv_defconfig
touch 2.kernel/arch/arm64/boot/dts/allwinner/sun50i-h618-ai-tv.dts
# Makefile 添加: dtb-$(CONFIG_ARCH_SUNXI) += sun50i-h618-ai-tv.dtb
```

### 5. Rootfs (debootstrap)

```bash
# 四阶段构建: env → firststage → secondstage → thirdstage → fourthstage → mkrootfsext4
cd 3.rootfs
make
# boot.cmd 启动脚本配置 SD/eMMC 启动
# thirdstage.sh chroot 内系统配置
```

### 6. 打包 (genimage)

```bash
# pack.sh 一键编译打包
# genimage.cfg 配置分区布局
# SD-to-eMMC 迁移脚本 sd2emmc.sh
```

## 关键配置点

### 补丁管理
- 使用 Armbian build 框架的 `series.conf` 管理补丁顺序
- 部分补丁位置不匹配需手动调整（如 bluetooth-h5、usb-typec 补丁）
- `kernel-6.18-patch.sh` 自动化补丁应用

### 网络配置
- netplan 配置静态IP/动态IP
- 网口参考硬件 `sun50i-h618-bananapi-m4-berry`

### Docker 支持
- 内核需开启 cgroup、namespace、overlay 等配置
- `_userdocker_defconfig` 专用内核配置

### 性能测试
- sysbench CPU/内存基准测试
- dd 磁盘IO测试

## 关键学习收获

1. chroot 内 systemd 有局限性，部分服务无法正常启动
2. Ubuntu 新版本 APT 格式变化需注意兼容性
3. 网络命名规则（predictable network names）可能影响配置
4. 桌面环境 suspend 问题需禁用相关服务
5. VPU 编解码支持需正确配置设备树和内核模块
6. Docker 离线部署需预装依赖包

## 相关笔记

- [[h3]] — Allwinner H3 系统构建
- [[h5]] — Allwinner H5 Crust Firmware
- [[h618]] — H618 TV Box 定制系统
- [[h618-buildroot]] — H618 Buildroot 构建
- [[docker-alicloud]] — Docker 镜像推送到阿里云
- [[orangepi]] — OrangePi 开发
