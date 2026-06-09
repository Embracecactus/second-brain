---
tags:
  - h618
  - buildroot
  - retropie
  - allwinner
  - arm64
  - embedded-linux
  - uboot
  - kernel
  - docker
  - retroarch
category: embedded-linux/allwinner
created: 2026-06-09
---

# H618 AI TV Box 完整开发笔记

## 项目概述

基于 Allwinner H618 SoC 的 AI TV 盒子项目，硬件参考设计为 `sun50i-h618-bananapi-m4-berry`。系统基于 ARM64 Ubuntu 22.04，从零构建完整的 Linux 系统（ATF -> U-Boot -> Kernel -> Rootfs -> 打包镜像），并在此基础上部署 Docker、CasaOS、桌面环境、VNC、摄像头推流、RetroPie 游戏模拟器等应用。

**技术栈：** ARM Trusted Firmware / U-Boot v2026.01 / Linux Kernel v6.18 / Ubuntu 22.04 rootfs / genimage 打包

## 关键知识点

### 1. 交叉编译环境搭建

- 编译器：`gcc-arm-11.2-2022.02-x86_64-aarch64-none-linux-gnu`
- 下载地址：https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads
- 解压至 `tools/` 目录后设置 `PATH` 环境变量

```sh
export PATH=$PWD/tools/gcc-arm-11.2-2022.02-x86_64-aarch64-none-linux-gnu/bin:$PATH
```

### 2. ARM Trusted Firmware (ATF)

- 源码：`git clone https://github.com/ARM-software/arm-trusted-firmware.git`
- 编译目标：`bl31`
- 使用 H616 平台配置（H618 兼容）

```sh
make CROSS_COMPILE=aarch64-none-linux-gnu- PLAT=sun50i_h616 DEBUG=1 bl31
export BL31=$PWD/build/sun50i_h616/debug/bl31.bin
```

### 3. U-Boot 编译

- 版本：v2026.01
- 需要自定义 defconfig 和设备树

```sh
git clone -b v2026.01 https://github.com/u-boot/u-boot.git --depth=1
# 创建 configs/sun50i-h618-ai-tv_defconfig
# 创建 dts/upstream/src/arm64/allwinner/sun50i-h618-ai-tv.dts
make ARCH=arm CROSS_COMPILE=aarch64-none-linux-gnu- sun50i-h618-ai-tv_defconfig
make ARCH=arm CROSS_COMPILE=aarch64-none-linux-gnu- -j32
```

### 4. Linux Kernel 编译

- 版本：v6.18（从 torvalds/linux 主线拉取）
- 需要打 Armbian 补丁（来自 `armbian/build` v26.2.0-trunk.239）
- 补丁位于 `build/patch/kernel/archive/sunxi-6.18/`
- 注意：部分 patch 位置不匹配需手动修改（如 bluetooth-h5、usb-typec 相关补丁）

```sh
git clone -b v6.18 https://github.com/torvalds/linux.git --depth=1
# 创建 arch/arm64/configs/sun50i-h618-ai-tv_defconfig
# 创建 arch/arm64/boot/dts/allwinner/sun50i-h618-ai-tv.dts
# 修改 arch/arm64/boot/dts/allwinner/Makefile 添加 dtb 条目
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- sun50i-h618-ai-tv_defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- -j32
```

内核模块安装到 rootfs：

```sh
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- INSTALL_MOD_PATH=../3.rootfs modules_install
```

### 5. Rootfs 文件系统制作

目录结构：
```
3.rootfs/
├── Makefile      # 打包脚本
├── bootscr/      # 启动脚本 (boot.cmd)
├── dtb/          # 设备树
├── etc/          # 配置文件 (netplan 等)
├── kernel/       # 内核 Image
├── lib/          # 内核模块
├── rootfs.ext4   # 根文件系统
└── temp/         # 临时文件
```

**启动脚本 boot.cmd（SD 卡启动）：**

```sh
load mmc ${devnum} ${kernel_addr_r} /boot/Image
load mmc ${devnum} ${fdt_addr_r} /boot/dtb/${fdtfile}
fdt addr ${fdt_addr_r}
fdt resize
setenv bootargs "console=ttyS0,115200 earlyprintk root=mmcblk1p1 rootfstype=ext4 rw rootwait"
booti ${kernel_addr_r} - ${fdt_addr_r}
```

**启动脚本 boot.cmd（eMMC 自适应启动，使用 PARTUUID）：**

```sh
echo "Booting from mmc ${mmc_bootdev}"
part uuid mmc ${mmc_bootdev} partuuid;
load mmc ${mmc_bootdev} ${kernel_addr_r} /boot/Image
load mmc ${mmc_bootdev} ${fdt_addr_r} /boot/dtb/${fdtfile}
fdt addr ${fdt_addr_r}
fdt resize
setenv bootargs "console=ttyS0,115200 root=PARTUUID=${partuuid} rootfstype=ext4 rw rootwait"
booti ${kernel_addr_r} - ${fdt_addr_r}
```

**网络配置（netplan）：**

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    end0:
      dhcp4: no
      addresses: [192.168.1.109/24]
      nameservers: { addresses: [8.8.8.8, 114.114.114.114] }
      routes:
        - on-link: true
          to: default
          via: 192.168.1.100
    end1:
      dhcp4: true
```

> 注意：Ubuntu 22.04 之前使用 `eth0`，之后使用 `end0` 命名。进入系统后需执行 `sudo chmod 600 /etc/netplan/01-network.yaml`。

### 6. 镜像打包

使用 `genimage` 工具将 U-Boot 和 rootfs 打包为最终镜像。

**genimage.cfg 配置：**

```
image my.img {
    hdimage { disk-signature = 0x12345678 }
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

```sh
genimage --input input --output images --config root/genimage.cfg
```

### 7. SD 卡复制到 eMMC 启动

通过 `sd2emmc.sh` 脚本完成复制，之后修改 boot.cmd 使用 PARTUUID 实现自适应启动。

**扩充存储空间：**

```sh
sudo parted /dev/mmcblk1 resizepart 1 100%
sudo resize2fs /dev/mmcblk1p1
```

## 技术细节

### Docker 安装与配置

内核需要开启的关键配置项：

```
CONFIG_IP_NF_FILTER=y, CONFIG_IP_NF_MANGLE=y, CONFIG_IP_NF_TARGET_MASQUERADE=y
CONFIG_IP6_NF_FILTER=y, CONFIG_IP6_NF_MANGLE=y, CONFIG_IP6_NF_TARGET_MASQUERADE=y
CONFIG_IP_NF_RAW=y, CONFIG_IP_NF_NAT=y, CONFIG_IP6_NF_RAW=y, CONFIG_IP6_NF_NAT=y
CONFIG_IP_NF_TARGET_REDIRECT=y, CONFIG_IP_SCTP=y
CONFIG_EXT3_FS=y, CONFIG_EXT3_FS_XATTR=y, CONFIG_EXT3_FS_POSIX_ACL=y
CONFIG_EXT3_FS_SECURITY=y, CONFIG_EXT4_FS_POSIX_ACL=y, CONFIG_EXT4_FS_SECURITY=y
```

Docker 国内镜像源（daemon.json）：

```json
{
  "registry-mirrors": [
    "https://docker.1ms.run",
    "https://docker-0.unsee.tech",
    "https://docker.m.daocloud.io",
    "https://hub.1panel.dev",
    "https://fwh64xdy.mirror.aliyuncs.com"
  ],
  "live-restore": true,
  "features": { "buildkit": true },
  "ipv6": false
}
```

### CasaOS 安装

- Docker 方式：`docker run -it --rm --name casa -p 8080:8080 -v "${PWD:-.}/casa:/DATA" -v "/var/run/docker.sock:/var/run/docker.sock" --stop-timeout 60 hub.1panel.dev/dockurr/casa:latest`
- 系统安装方式：`curl -fsSL https://get.casaos.io | sudo bash`
- 注意：CasaOS 内嵌 Docker CLI 版本过旧（API 1.43）可能导致应用列表刷新失败，需升级客户端
- 离线安装：可手动下载各组件 tar.gz 到 `/tmp/casaos-installer/` 后执行 `casaos-install.sh -p /tmp/casaos-installer/build`

### 桌面环境 (XFCE4)

```sh
sudo apt-get install xorg xfce4
sudo apt-get install lightdm  # Ubuntu 22.04 下 lightdm + XFCE 有兼容问题
# 解决 "failed to start session"：清除 lightdm 改用 gdm3
sudo apt purge lightdm lightdm-gtk-greeter lightdm-settings
sudo apt install gdm3
```

**防休眠配置**（/etc/systemd/logind.conf）：

```ini
HandleSuspendKey=ignore
HandleHibernateKey=ignore
HandleLidSwitch=ignore
IdleAction=ignore
```

```sh
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

### VNC 远程桌面

使用 x11vnc，端口 5900，配置 systemd 自启动服务 `/etc/systemd/system/x11vnc.service`。

### VPU 视频处理

- H618 内置 VPU（cedrus 驱动），支持 MPEG-2 / H.264 / HEVC / VP8 硬解
- 设备节点：`/dev/video0`（cedrus）、`/dev/video1`（USB 摄像头）
- FFmpeg 需从源码编译安装（板上编译约 30 分钟）
- 推流命令示例：`ffmpeg -f v4l2 -input_format mjpeg -video_size 1280x720 -framerate 30 -i /dev/video1 -vcodec copy -f mpegts udp://192.168.1.100:1234`

### 摄像头使用

- mjpg-streamer 推流：`./mjpg_streamer -i "./input_uvc.so -d /dev/video1 -u -f 30" -o "./output_http.so -w ./www"`，访问 `http://ip:8080`
- fswebcam 抓图：`fswebcam -d /dev/video1 --no-banner -r 1280x720 -s 5 ./image.jpg`
- OpenCV + Flask / C++ 实现自定义视频流

### RetroArch Docker

```sh
docker run -d --name=retroarch \
  -e PUID=1000 -e PGID=1000 -e TZ=Etc/UTC \
  -p 3000:3000 -p 3001:3001 \
  -v /home/zzx/config:/config \
  --shm-size="2gb" --restart unless-stopped \
  linuxserver/retroarch:latest
```

访问 `http://ip:3000`。

### RetroPie 安装

```sh
git clone --depth=1 https://github.com/RetroPie/RetroPie-Setup.git
cd RetroPie-Setup && sudo ./retropie_setup.sh
# 选择 Basic install
```

- PS1 推荐使用 `pcsx_rearmed` 模拟器
- PS2 不支持 ARM 架构
- BIOS 文件和 ROM 资源见相关链接

### 性能测试命令

```sh
sysbench cpu --cpu-max-prime=20000 run
sysbench memory --memory-total-size=1G run
sysbench fileio --file-total-size=1G --file-test-mode=rndrw run
dd if=/dev/zero of=./test_write.img bs=1M count=1024 oflag=direct status=progress
sudo stress --cpu 4 --io 2 --vm 1 --vm-bytes 128M --timeout 60s
```

### U-Boot 启动流程

```
遍历设备分区列表 -> 扫描 bootfstype
  -> 查找 extlinux.conf
  -> 查找 boot.scr / boot.cmd
  -> 扫描 EFI 系统分区
```

### Rootfs 构建中的 systemd-sysusers 原理

chroot 构建阶段出现 `Failed to resolve user 'xxx'` 警告是正常的。原因：
1. `systemd-sysusers` 在首次启动时根据 `/usr/lib/sysusers.d/*.conf` 自动创建系统用户
2. `systemd-tmpfiles` 在启动时根据 `/usr/lib/tmpfiles.d/*.conf` 创建运行时目录
3. chroot 环境没有真正的 init 进程，无法执行这些初始化流程，首次启动后自动修复

## 相关笔记

- [[h3]] — Allwinner H3 系统构建全栈笔记
- [[h5]] — Allwinner H5 Crust Firmware 项目
- [[h618]] — H618 TV Box 定制 Linux 系统
- [[h618-readme]] — H618 BSP 开发笔记
- [[orangepi]] — OrangePi PC 嵌入式 Linux 开发
- [[ffmpeg]] — FFmpeg 多媒体处理框架
- [[docker-alicloud]] — Docker 镜像推送到阿里云 ACR
- [[redbook]] — 小红书 H618 相关技术内容

## 外部链接

- ARM GNU Toolchain: https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads
- ARM Trusted Firmware: https://github.com/ARM-software/arm-trusted-firmware
- U-Boot: https://github.com/u-boot/u-boot (分支 v2026.01)
- Linux Kernel: https://github.com/torvalds/linux (分支 v6.18)
- Armbian Build: https://github.com/armbian/build
- RetroPie Setup: https://github.com/RetroPie/RetroPie-Setup
- CasaOS 官方安装: https://get.casaos.io
- FFmpeg 源码: https://github.com/FFmpeg/FFmpeg
