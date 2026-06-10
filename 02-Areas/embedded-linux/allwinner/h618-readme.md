---
tags:
  - embedded-linux
  - allwinner
  - h618
  - arm64
  - bsp
  - uboot
  - kernel
  - rootfs
  - docker
  - device-tree
category: embedded-linux/allwinner
created: 2026-06-09
status: active
---

# Allwinner H618 AI TV Box BSP

## 项目概述

这是一个基于 Allwinner H618 SoC 的 AI TV Box 完整 BSP（Board Support Package）项目，从源码构建可启动的 ARM64 Linux 镜像。硬件参考设计为 `sun50i-h618-bananapi-m4-berry`，仓库仅包含自定义/修改的文件（配置、设备树、补丁、脚本），U-Boot、内核和 ATF 源码在构建过程中从外部克隆。

## 技术栈

- **SoC**: Allwinner H618 (sun50i-h618, ARM Cortex-A53 四核)
- **架构**: ARM64 (aarch64)
- **交叉编译器**: gcc-arm-11.2-2022.02-x86_64-aarch64-none-linux-gnu
- **ATF**: ARM Trusted Firmware (sun50i_h616 平台)
- **U-Boot**: v2026.01
- **Linux Kernel**: v6.18 (主线内核 + Armbian sunxi-6.18 补丁集)
- **Rootfs**: Ubuntu 24.04 (noble) 或 Debian 11 (bullseye)，通过 debootstrap 构建
- **文件系统**: ext4
- **镜像打包**: genimage
- **桌面环境**: ubuntu-desktop-minimal / xfce4（可选）
- **容器化**: Docker / Podman（可选）
- **网络管理**: netplan (networkd renderer)

## 架构与关键设计决策

### 构建流水线

严格顺序执行：**ATF -> U-Boot -> Kernel -> Rootfs -> Pack**

```
0.arm-trusted-firmware/   # ATF BL31 固件
1.uboot/                  # U-Boot defconfig + 设备树（仅自定义部分）
2.kernel/                 # 内核 defconfig + 设备树（仅自定义部分）
3.rootfs/                 # 根文件系统构建系统（Makefile 驱动）
pack/                     # 最终镜像打包（genimage 配置）
app/                      # 应用层文档和 Docker 配置
```

### 目录设计理念

仓库不存储完整源码，只保存增量自定义文件。ATF/U-Boot/内核源码通过 `git clone` 外部获取，补丁通过 Armbian build 工具链应用。这使得仓库保持轻量，同时完整记录了所有定制化改动。

### 启动流程

```
BL31 (ATF) -> U-Boot SPL -> U-Boot -> boot.scr -> Linux Kernel -> rootfs
```

U-Boot 通过 `boot.scr` 脚本加载内核 Image 和设备树，关键参数包括：
- `PARTUUID` 用于分区识别（与 genimage.cfg 中 disk-signature 一致）
- `${mmc_bootdev}` 变量实现 SD/eMMC 自适应启动
- 串口控制台 `ttyS0,115200`

### 设备树策略

U-Boot 和内核各自维护独立的 `sun50i-h618-ai-tv.dts` 设备树文件，分别位于 `1.uboot/` 和 `2.kernel/` 目录。这种分离设计允许引导阶段和运行时阶段的硬件配置独立调整。

### 网络配置

使用 netplan 配置，注意 Ubuntu 22.04+ 使用 `end0`/`end1` 接口名（非传统 `eth0`）。netplan 文件权限必须设置为 600 才能正常工作。

### Docker 支持

内核需启用特定配置（iptables NAT、ext4 ACL/xattr/security），通过 `chek-config.sh` 脚本检测。关键配置项：

```sh
CONFIG_IP_NF_FILTER=y
CONFIG_IP_NF_MANGLE=y
CONFIG_IP_NF_TARGET_MASQUERADE=y
CONFIG_IP6_NF_FILTER=y
CONFIG_IP6_NF_MANGLE=y
CONFIG_IP6_NF_TARGET_MASQUERADE=y
CONFIG_IP_NF_RAW=y
CONFIG_IP_NF_NAT=y
CONFIG_IP6_NF_RAW=y
CONFIG_IP6_NF_NAT=y
CONFIG_IP_NF_TARGET_REDIRECT=y
CONFIG_IP_SCTP=y
CONFIG_EXT3_FS=y
CONFIG_EXT3_FS_XATTR=y
CONFIG_EXT3_FS_POSIX_ACL=y
CONFIG_EXT3_FS_SECURITY=y
CONFIG_EXT4_FS_POSIX_ACL=y
CONFIG_EXT4_FS_SECURITY=y
```

### 根文件系统构建

使用 Makefile 驱动的多阶段 debootstrap 构建流程：

```makefile
# 关键配置变量
HOSTNAME := h618-tv-box
ROOTFS_TYPE := noble          # noble=Ubuntu 24.04, bullseye=Debian 11
ARCH := arm64
HAS_DESKTOP := y              # 是否包含桌面环境
COUNT := 9000                 # rootfs.ext4 大小 (MB)
```

构建阶段：
1. **env**: 安装依赖（qemu, debootstrap）
2. **firststage**: debootstrap --foreign 下载基础包
3. **secondstage**: chroot 执行 debootstrap second-stage
4. **thirdstage**: 安装软件包、创建用户、配置系统
5. **fourthstage**: 复制内核、设备树、boot.scr 到 rootfs
6. **mkrootfsext4**: 创建 ext4 镜像文件

### 内核补丁策略

补丁来源为 Armbian 的 sunxi-6.18 系列，通过 `kernel-6.18-patch.sh` 脚本应用。两个已知无法自动应用的补丁需要手动处理：
- `0001-bluetooth-h5-Don-t-re-initialize-rtl8723cs-on-resume.patch`
- `0002-usb-typec-altmodes-displayport-Respect-DP_CAP_RECEPT.patch`

## 关键代码片段

### 一键构建脚本 (pack.sh)

```bash
#!/bin/bash
# 设置交叉编译器路径
# export PATH=$PWD/tools/gcc-arm-11.2-2022.02-x86_64-aarch64-none-linux-gnu/bin:$PATH

# 1. 编译 ATF
cd 0.arm-trusted-firmware && \
make CROSS_COMPILE=aarch64-none-linux-gnu- PLAT=sun50i_h616 DEBUG=1 bl31 && \
export BL31="$PWD/build/sun50i_h616/debug/bl31.bin"

# 2. 编译 U-Boot
cd ../1.uboot && \
make ARCH=arm CROSS_COMPILE=aarch64-none-linux-gnu- sun50i-h618-ai-tv_defconfig && \
make ARCH=arm CROSS_COMPILE=aarch64-none-linux-gnu- -j32

# 3. 编译内核
cd ../2.kernel && \
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- sun50i-h618-ai-tv_defconfig && \
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- -j32

# 4. 安装模块到 rootfs
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- INSTALL_MOD_PATH=../3.rootfs modules_install

# 5. 复制内核和设备树
cp -r arch/arm64/boot/Image ../3.rootfs/kernel/
cp -r arch/arm64/boot/dts/allwinner ../3.rootfs/dtb/allwinner/

# 6. 构建 rootfs
cd ../3.rootfs && make boot_scr && make fourthstage && make mkrootfsext4

# 7. 打包最终镜像
cd ../pack && genimage --input input --output images --config root/genimage.cfg
```

### 自适应启动 boot.cmd（支持 SD/eMMC）

```
echo "Booting from mmc ${mmc_bootdev}"
part uuid mmc ${mmc_bootdev} partuuid;
load mmc ${mmc_bootdev} ${kernel_addr_r} /boot/Image
load mmc ${mmc_bootdev} ${fdt_addr_r} /boot/dtb/${fdtfile}
fdt addr ${fdt_addr_r}
fdt resize
setenv bootargs "console=ttyS0,115200 root=PARTUUID=${partuuid} rootfstype=ext4 rw rootwait"
booti ${kernel_addr_r} - ${fdt_addr_r}
```

### genimage 磁盘布局配置

```cfg
image my.img {
    hdimage {
        disk-signature = 0x12345678    # 必须与 boot.cmd 中 PARTUUID 一致
    }
    partition u-boot {
        in-partition-table = false
        image = "u-boot-sunxi-with-spl.bin"
        offset = 8K
        size = 40M
    }
    partition rootfs {
        partition-type = 0x83
        image = "rootfs.ext4"
        size = 9000M
    }
}
```

### SD 卡迁移至 eMMC (sd2emmc.sh)

```bash
# 清除 eMMC 前部
sudo dd if=/dev/zero of=/dev/mmcblk0 bs=1M count=50
# 写入 U-Boot
sudo dd if=/dev/mmcblk1 of=/dev/mmcblk0 bs=512 count=81920 status=progress conv=fsync
# 创建分区表并复制文件系统
sudo fdisk /dev/mmcblk0 <<EOF
o n p 1 81920 w EOF
sudo dd if=/dev/mmcblk1p1 of=/dev/mmcblk0p1 bs=4M status=progress
sudo parted /dev/mmcblk0 resizepart 1 100%
sudo e2fsck -f -y /dev/mmcblk0p1
sudo resize2fs /dev/mmcblk0p1
```

## 关键学习与经验

### 1. chroot 环境中的 systemd 限制
在 debootstrap 构建 rootfs 时会看到类似 `Failed to resolve user 'gnome-remote-desktop': No such process` 的警告。这是正常的，因为 chroot 环境没有真正的 systemd init 进程。系统用户由 `systemd-sysusers` 在首次启动时自动创建，临时目录由 `systemd-tmpfiles` 在 early boot 阶段创建。构建阶段的警告只是"时机不对"，不是"功能缺失"。

### 2. Ubuntu 24.04 APT 源格式变更
从 Ubuntu 22.10 开始，APT 默认使用 deb822 格式（`.sources` 文件），不再使用传统的 `sources.list`。即使手动创建了 `sources.list`，Ubuntu 安装过程会自动生成 `/etc/apt/sources.list.d/ubuntu.sources` 并优先使用。

### 3. 网络接口命名
Ubuntu 22.04+ 使用 `end0`/`end1` 接口名，不再使用 `eth0`。netplan 配置文件必须设置 `chmod 600` 权限。

### 4. 桌面环境与休眠问题
安装 xfce4 桌面后，系统可能因休眠导致 SSH 和串口连接断开。解决方案是禁用所有休眠相关服务：
```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```
并在 `/etc/systemd/logind.conf` 中设置 `IdleAction=ignore`。

### 5. VPU 视频解码支持
H618 的 Cedrus VPU 支持以下格式：
- MPEG-2 Parsed Slice Data (MG2S)
- H.264 Parsed Slice Data (S264)
- HEVC Parsed Slice Data (S265)
- VP8 Frame (VP8F)

### 6. Docker 镜像离线部署
在无网络环境下，可通过 `docker save` / `docker load` 方式离线部署镜像。在主机上拉取并保存为 tar 包，再传输到开发板加载。

## 构建/运行指南

### 环境准备

```bash
# 1. 下载交叉编译器
# https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads
cd tools && tar -xvf gcc-arm-11.2-2022.02-x86_64-aarch64-none-linux-gnu.tar.xz

# 2. 设置环境变量
export PATH=$PWD/tools/gcc-arm-11.2-2022.02-x86_64-aarch64-none-linux-gnu/bin:$PATH

# 3. 安装构建依赖
sudo apt-get install qemu qemu-user-static binfmt-support debootstrap genimage mkimage
```

### 完整构建

```bash
bash pack.sh
```

### 单独构建各阶段

```bash
# ATF
cd 0.arm-trusted-firmware
make CROSS_COMPILE=aarch64-none-linux-gnu- PLAT=sun50i_h616 DEBUG=1 bl31
export BL31=$PWD/build/sun50i_h616/debug/bl31.bin

# U-Boot
cd 1.uboot
make ARCH=arm CROSS_COMPILE=aarch64-none-linux-gnu- sun50i-h618-ai-tv_defconfig
make ARCH=arm CROSS_COMPILE=aarch64-none-linux-gnu- -j32

# Kernel
cd 2.kernel
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- sun50i-h618-ai-tv_defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- -j32

# Rootfs
cd 3.rootfs && make all

# Pack
cd pack && genimage --input input --output images --config root/genimage.cfg
```

### 烧录后操作

```bash
# 扩展存储空间
sudo parted /dev/mmcblk1 resizepart 1 100%
sudo resize2fs /dev/mmcblk1p1

# 性能测试
sysbench cpu --cpu-max-prime=20000 run
sysbench memory --memory-total-size=1G run
dd if=/dev/zero of=./test_write.img bs=1M count=1024 oflag=direct status=progress
```

## 相关链接

- [[embedded-linux-basics]] - 嵌入式 Linux 基础知识
- [[allwinner-sunxi-platform]] - Allwinner SunXi 平台概述
- [[device-tree-guide]] - 设备树编写指南
- [[u-boot-customization]] - U-Boot 定制化
- [[arm-trusted-firmware]] - ARM Trusted Firmware 详解
- [[debootstrap-rootfs]] - 使用 debootstrap 构建根文件系统
- [[docker-on-arm64]] - ARM64 平台 Docker 部署
- [[kernel-patching-workflow]] - 内核补丁工作流

## 参考资源

- [ARM GNU Toolchain Downloads](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads)
- [U-Boot 官方仓库](https://github.com/u-boot/u-boot)
- [Linux 内核源码](https://github.com/torvalds/linux)
- [Armbian Build](https://github.com/armbian/build)
- [ARM Trusted Firmware](https://github.com/ARM-software/arm-trusted-firmware)
- [Megi's kernel patches (sunxi-6.18)](https://codeberg.org/megi/linux/src/branch/orange-pi-6.18)
- [Docker 内核配置检查脚本](https://github.com/moby/moby/blob/master/contrib/check-config.sh)
