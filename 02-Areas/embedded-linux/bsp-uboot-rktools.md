---
tags:
  - embedded-linux
  - bsp
  - bootloader
  - u-boot
  - rockchip
  - tools
  - rkdeveloptool
  - flashing
  - rv1126b
category: embedded-linux
created: 2026-06-23
updated: 2026-06-23
status: active
soc: Rockchip RV1126B
kernel: Linux 6.1.141
parent: bsp-boot-flow
---

# Rockchip 工具链深度解析

> **前置笔记**：[[bsp-boot-flow]] — Boot Chain 全景 & 分区表
>
> **前置笔记**：[[bsp-uboot-adaptation]] — U-Boot 板级适配 (编译流程)
>
> 本文聚焦 Rockchip SDK 配套的工具链：从固件打包到烧录部署的完整工作流。

---

## 一、工具链全景

Rockchip 的固件工具链分为三个层次：

```
┌─────────────────────────────────────────────────────────┐
│ 层次 1: 构建工具 (SDK Makefile / build.sh)              │
│   └─ 调用层次 2 和层次 3，用户通常只接触这一层           │
│   tools/build.sh, device/rockchip/common/scripts/mk-*.sh │
├─────────────────────────────────────────────────────────┤
│ 层次 2: 固件组装工具 (rkbin/tools/)                     │
│   └─ 将各部件组装为可烧录的镜像                          │
│   boot_merger, trust_merger, mkimage, loaderimage,       │
│   resource_tool, fit-sign.sh                             │
├─────────────────────────────────────────────────────────┤
│ 层次 3: 烧录/部署工具 (rkbin/tools/)                    │
│   └─ 将组装好的镜像写入板端存储                          │
│   rkdeveloptool, upgrade_tool, rkflash.sh                │
└─────────────────────────────────────────────────────────┘
```

---

## 二、固件组装管线

### 2.1 固件组成

RV1126B 最终烧录到 eMMC 的固件包含：

```
boot media (eMMC) 布局:
┌─────────────────────┬──────────┬──────────┐
│ 分区                │ 固件文件   │ 大小      │
├─────────────────────┼──────────┼──────────┤
│ uboot (p1)          │ 见下方    │ 4 MB     │
│ misc (p2)           │ misc.img  │ 4 MB     │
│ boot (p3)           │ boot.img  │ 64 MB    │
│ amp (p4)            │ amp.img   │ 4 MB     │
│ recovery (p5)       │ recovery  │ 128 MB   │
│ backup (p6)         │ -         │ 32 MB    │
│ rootfs (p7)         │ rootfs    │ ~6 GB    │
│ oem (p8)            │ -         │ 128 MB   │
│ userdata (p9)       │ -         │ 剩余     │
└─────────────────────┴──────────┴──────────┘

uboot 分区内部结构 (4MB 连续写入):
┌─────────────────────────────────────────────────────┐
│ MiniLoaderAll.bin  (boot_merger 合并产物)            │
│   ├─ DDR 初始化固件 (rv1126b_ddr_1332MHz_v1.09.bin) │
│   ├─ USB 下载固件 (rv1126b_usbplug_v1.03.bin)       │
│   └─ SPL 加载器 (rv1126b_spl_v1.05.bin)             │
│                                                     │
│ uboot.img (mkimage 打包的 FIT 镜像)                  │
│   ├─ U-Boot proper (u-boot-nodtb.bin)               │
│   ├─ U-Boot DTB (u-boot.dtb)                        │
│   └─ signature (可选)                                │
│                                                     │
│ trust.img (trust_merger 合并产物)                    │
│   ├─ ATF BL31 (rv1126b_bl31_v1.13.elf)              │
│   └─ OP-TEE BL32 (rv1126b_bl32_v1.04.bin)           │
└─────────────────────────────────────────────────────┘
```

### 2.2 `boot_merger` — Loader 合并

```
boot_merger RKBOOT/RV1126BMINIALL.ini
  │
  └─ 根据 ini 配置顺序拼接固件:
      [CODE471_OPTION]
      Path1=bin/rv11/rv1126b_ddr_1332MHz_v1.09.bin    ← DDR 固件
      加载地址: 0xFF000000 (ISRAM, 芯片内部 SRAM)

      [CODE472_OPTION]
      Path1=bin/rv11/rv1126b_usbplug_v1.03.bin         ← USB 下载固件
      加载地址: 0xFF000000

      [LOADER_OPTION]
      FlashData=bin/rv11/rv1126b_ddr_1332MHz_v1.09.bin ← DDR 数据
      FlashBoot=bin/rv11/rv1126b_spl_v1.05.bin          ← SPL 引导
      加载地址: 0x4FE00000 (DRAM 高端)

      [OUTPUT]
      PATH=rv1126b_spl_loader_v1.09.105.bin             ← 合并输出
```

**INI 字段含义**：

| 字段 | 含义 |
|------|------|
| `CODE471_OPTION` | DDR 初始化代码, 在 ISRAM 中执行 |
| `CODE472_OPTION` | USB 下载代码, 在 ISRAM 中执行 |
| `LOADER_OPTION.FlashData` | DDR 配置数据 (DRAM 地址) |
| `LOADER_OPTION.FlashBoot` | SPL 代码 (DRAM 高端, 0x4FE00000) |

### 2.3 `trust_merger` — Trust 镜像合并

```
trust_merger RKTRUST/RV1126BTRUST.ini
  │
  └─ 合并 ATF + OP-TEE:
      [BL31_OPTION]
      SEC=1                               ← 安全固件
      PATH=bin/rv11/rv1126b_bl31_v1.13.elf ← ATF
      ADDR=0x0                             ← 加载地址 (由 SPL 设置)

      [BL32_OPTION]
      SEC=0                               ← 非安全固件
      PATH=bin/rv11/rv1126b_bl32_v1.04.bin  ← OP-TEE
      ADDR=0x48400000                      ← 加载地址

      [OUTPUT]
      PATH=trust.img                       ← 合并输出
```

### 2.4 `mkimage` — FIT 镜像打包

```bash
# mkimage 是 U-Boot 编译产物, 用于:
# 1. 打包 FIT 镜像 (boot.img, recovery.img)
# 2. 签名和验签 FIT 镜像

# 打包 FIT:
mkimage -f boot.its -r -E boot.img
# -f: FIT ITS 描述文件
# -r: required 模式 (签名强制)
# -E: external data (数据在 FDT 结构体之后)

# 查看 FIT 内容:
mkimage -l boot.img

# 签名:
mkimage -f boot.its -k keys/ -r boot_signed.img
```

### 2.5 `resource_tool` — 资源镜像

```bash
# 管理 boot.img 中的 resource 分区 (logo, 开机动画)
resource_tool --add logo.bmp              # 添加 logo
resource_tool --add --type=animation boot_animation.zip
resource_tool --list                       # 查看已有资源
resource_tool --unpack                     # 提取资源
```

### 2.6 `fit-sign.sh` / `fit-repack.sh` / `fit-unpack.sh`

```bash
# FIT 签名/解包工具, 用于产线场景:

# 解包 FIT → 提取其中的各段数据
./fit-unpack.sh boot.img boot_repack/

# 修改后重新打包 (保留原 ITS 结构)
./fit-repack.sh boot_repack/ boot_new.img

# 签名已存在的 FIT
./fit-sign.sh boot.img keys/dev.key boot_signed.img
```

---

## 三、烧录/部署工具链

### 3.1 `rkdeveloptool` — 主力开发工具

`rkdeveloptool` 是 Rockchip 在 Maskrom/Loader 模式下通过 USB 烧录的主要工具。

**工作模式**：

```
板子状态           USB 通信方式          可执行操作
──────────────────────────────────────────────────
Maskrom 模式       rkdeveloptool db    下载 loader, OTP 烧录
Loader 模式        rkdeveloptool ul    全盘烧录, 分区烧录
Normal 模式        不适用              系统内操作 (dd/scp)
```

**进入 Maskrom 模式**：

```bash
# 方法 1: 按住板子上的 MASKROM/RECOVERY 按键 → 上电
# 方法 2: 连接 USB, 在 U-Boot shell 中:
rockusb 0 0x20000000 0x20000000
# 方法 3: EMMC 未烧录有效的 bootloader 时自动进入
```

**常用命令**：

```bash
# 1. 下载 loader (进入 Maskrom 模式后的第一步)
rkdeveloptool db rv1126b_spl_loader_v1.09.105.bin
# db = download boot → 将 loader 下载到板子 ISRAM 并执行
# 成功后板子进入 Loader 模式

# 2. 检查通信
rkdeveloptool poll
# 预期: 返回设备信息, 确认 USB 连接正常

# 3. 列出存储设备
rkdeveloptool list
# 预期: "mmc 0:01000000" 等

# 4. 读取 Flash 信息
rkdeveloptool rfi
# 返回 Flash 类型, 块大小, 总容量

# 5. 全片烧录
# 方法 A: 写单个分区
rkdeveloptool wl 0x40 uboot.img       # 写 uboot 分区 (偏移 0x40 * 512 = 0x8000 字节)
# 注意: 偏移按 512 字节扇区, 与 parameter.txt 一致

# 方法 B: 写 GPT 表 + 所有分区
rkdeveloptool gpt parameter.txt       # 写入 GPT 分区表
rkdeveloptool wl 0x40 uboot.img
rkdeveloptool wl 0x80 boot.img
rkdeveloptool wl 0x2800 rootfs.img

# 方法 C: 写完整固件 (update.img)
rkdeveloptool wl 0x0 update.img

# 6. 读取分区
rkdeveloptool rl 0x40 0x2000 uboot_backup.img   # 读取 uboot 分区

# 7. 重启到 Flash 启动
rkdeveloptool rd
# rd = reset device

# 8. OTP 操作 (警告: 不可逆!)
rkdeveloptool otp           # 读取 OTP 状态
rkdeveloptool otp -s key    # 烧录 OTP 密钥

# 9. 列出分区表
rkdeveloptool pt            # 读取 GPT 表
```

**典型烧录流程**：

```bash
#!/bin/bash
# 完整烧录脚本

# 1. 进入 Maskrom → 下载 loader
rkdeveloptool db rv1126b_spl_loader_v1.09.105.bin
sleep 1

# 2. 等待进入 Loader 模式
rkdeveloptool poll || { echo "连接失败"; exit 1; }

# 3. 烧录 GPT 分区表
rkdeveloptool gpt parameter-amp.txt

# 4. 依次烧录各分区
rkdeveloptool wl 0x40 uboot.img
rkdeveloptool wl 0x60 misc.img
rkdeveloptool wl 0x80 boot.img
rkdeveloptool wl 0x2800 amp.img
rkdeveloptool wl 0x2a00 recovery.img
rkdeveloptool wl 0x7a000 rootfs.img

# 5. 重启
rkdeveloptool rd
```

### 3.2 `upgrade_tool` — 产线烧录工具

`upgrade_tool` 是 `rkdeveloptool` 的前身，接口类似但有以下区别：

| 特性 | rkdeveloptool | upgrade_tool |
|------|--------------|-------------|
| 开源 | ✅ GPL | ❌ 闭源 |
| 更新频率 | 社区维护 | Rockchip 发布 |
| OTP 烧录 | ✅ | ✅ |
| 批量烧录 | 一般 | 支持配置文件 |
| 目前推荐使用 | 开发调试 | 产线烧录 |

```bash
# upgrade_tool 用法:
upgrade_tool db loader.bin
upgrade_tool wl 0x40 uboot.img
upgrade_tool rd
```

### 3.3 `rkflash.sh` — SDK 封装脚本

SDK 提供 `rkflash.sh` 脚本封装了 `rkdeveloptool`：

```bash
# 在 SDK 根目录:
./rkflash.sh                    # 查看帮助
./rkflash.sh all                # 烧录全部固件
./rkflash.sh boot               # 仅烧录 boot 分区
./rkflash.sh uboot              # 仅烧录 uboot 分区
./rkflash.sh trust              # 仅烧录 trust
./rkflash.sh rootfs             # 仅烧录 rootfs
./rkflash.sh parameter          # 仅烧录分区表
./rkflash.sh misc               # 仅烧录 misc
./rkflash.sh db                 # 下载 loader (进入 Maskrom 后)
./rkflash.sh gpt                # 写入 GPT 分区表
```

### 3.4 板端部署 (dd)

通过 SSH 在板端直接写入 eMMC：

```bash
# 板端: 查看分区
cat /proc/partitions
ls -la /dev/mmcblk0p*

# 写入 boot 分区 (p3 = /dev/mmcblk0p3)
sudo dd if=boot.img of=/dev/mmcblk0p3 bs=512

# 写入 uboot 分区 (p1)
sudo dd if=uboot.img of=/dev/mmcblk0p1 bs=512

# 对比: dd vs rkdeveloptool
# dd: 简单直接, 适合单分区更新
# rkflash.sh: 需要 USB 连接, 适合完整烧录
```

---

## 四、固件打包管线

### 4.1 完整构建流程

SDK `build.sh` 背后的调用链：

```
make menuconfig / make rv1126b_sportcam_defconfig
  │
./build.sh kernel
  │  └─ 编译 Linux 内核 → arch/arm64/boot/Image
  │                       arch/arm64/boot/dts/rockchip/rv1126b-sportcam.dtb
  │
./build.sh loader (或: ./build.sh uboot)
  │  └─ 编译 U-Boot:
  │      ├─ u-boot-nodtb.bin (U-Boot proper)
  │      ├─ u-boot.dtb (U-Boot 设备树)
  │      ├─ u-boot.img (mkimage FIT 打包)
  │      ├─ spl/u-boot-spl.bin (SPL)
  │      ├─ rv1126b_spl_loader_v1.xx.xxx.bin (boot_merger 合并)
  │      └─ trust.img (trust_merger 合并)
  │
./build.sh buildroot
  │  └─ 编译 Buildroot rootfs → rootfs.ext2/ext4
  │
./build.sh firmware
  │  └─ 组装固件到 output/firmware/:
  │      ├─ MiniLoaderAll.bin     → boot_merger 产物
  │      ├─ uboot.img             → U-Boot FIT
  │      ├─ trust.img             → ATF + OP-TEE
  │      ├─ boot.img              → Kernel FIT
  │      ├─ rootfs.img            → rootfs
  │      ├─ amp.img               → MCU 固件
  │      ├─ misc.img              → recovery 标志
  │      └─ parameter.txt         → 分区表
  │
./build.sh updateimg
  │  └─ 打包 update.img (完整烧录镜像, 包含所有分区)
```

### 4.2 update.img 结构

`update.img` 是固件的**打包分发格式**，包含：

```
update.img
  ├─ 头部 (镜像描述, 校验和)
  ├─ MiniLoaderAll.bin
  ├─ parameter.txt (GPT 分区表)
  ├─ uboot.img
  ├─ misc.img
  ├─ boot.img
  ├─ amp.img
  ├─ recovery.img
  ├─ rootfs.img
  ├─ oem.img (可选)
  └─ userdata.img (可选)
```

生成方式：

```bash
# SDK 中:
./build.sh updateimg

# 或手动:
# tools目录下的 firmwareMerger 或 programmer_image_tool
programmer_image_tool -c firmware.cfg
```

---

## 五、签名工具

### 5.1 `rk_sign_tool` — Rockchip 签名工具

用于对 loader/uboot 进行 Rockchip 专有格式签名（不同于 FIT 签名）：

```bash
rk_sign_tool help

# 签名 loader:
rk_sign_tool sign --loader rv1126b_spl_loader_v1.09.105.bin \
                   --key private_key.pem

# 签名 uboot:
rk_sign_tool sign --uboot u-boot.img \
                   --key private_key.pem

# 签名 boot:
rk_sign_tool sign --boot boot.img \
                   --key private_key.pem
```

### 5.2 `fit_check_sign` — FIT 签名验证

```bash
# 验证 FIT 镜像的签名:
fit_check_sign -f boot.img -k dev.pub
# 输出: "signature verification passed" / "failed"

# 用于产线自动化测试:
if fit_check_sign -f boot.img -k prod.pub; then
    echo "✅ 签名验证通过, 准备烧录"
else
    echo "❌ 签名验证失败, 拒绝烧录"
    exit 1
fi
```

---

## 六、诊断工具

### 6.1 DDR 配置工具

```bash
# rkbin/tools/ddrbin_tool.py
# 用于配置 DDR 参数 (频率、时序、颗粒型号)

python3 ddrbin_tool.py -p ddrbin_param.txt rv1126b_ddr_1332MHz_v1.09.bin

# ddrbin_param.txt 关键参数:
# ddr_freq: 1332                 # DDR 频率 (MHz)
# ddr_type: 3                    # 0=DDR3, 1=DDR4, 2=LPDDR3, 3=LPDDR4
# ddr_capacity: 512              # 总容量 (MB)
# ddr_chip: "hynix"|"samsung"|"micron"  # 颗粒品牌

# 当更换 DDR 颗粒时, 需要:
# 1. 向 Rockchip 获取对应颗粒的 DDR bin
# 2. 或使用 ddrbin_tool.py 调整参数
# 3. 用新 bin 重新合并 loader
```

### 6.2 `loaderimage` — Loader 分析

```bash
# 解析 loader 内容
loaderimage -l rv1126b_spl_loader_v1.09.105.bin

# 输出示例:
# Chip: RV1126B
# Version: v1.09.105
# Size: 0x...
# [Entry 0] DDR Init       @0xFF000000  size=0x...
# [Entry 1] USB Plug       @0xFF000000  size=0x...
# [Entry 2] SPL Boot       @0x4FE00000  size=0x...
```

### 6.3 `gpt2env` — GPT 转环境变量

```bash
# 将分区表转换为 U-Boot 环境变量
gpt2env parameter-amp.txt

# 输出:
# mmcparts=...
# uboot_part=0x4000
# boot_part=0x8000
# rootfs_part=0x7a000
# ...

# 可用于在 U-Boot shell 中动态设置分区
```

### 6.4 `bmp2gray16` — Logo 转换

```bash
# 将 BMP 图片转换为 U-Boot 支持的灰度 16 位格式
bmp2gray16 logo.bmp logo_gray16.bmp

# 然后用 resource_tool 添加到 boot.img:
resource_tool --add logo_gray16.bmp
# 重启后 U-Boot 启动阶段显示 logo
```

---

## 七、产线场景工具流程

### 7.1 产线烧录流程

```
产线 PC (运行 Windows/Linux)
  │
  ├─ USB Hub (同时连接 8-16 台板子)
  │
  ├─ 每个板子处于 Maskrom 模式 (出厂未烧录)
  │
  ├─ 步骤 1: 统一烧录 preloader
  │   rkdeveloptool db MiniLoaderAll.bin
  │
  ├─ 步骤 2: 写入 GPT 分区表
  │   rkdeveloptool gpt parameter.txt
  │
  ├─ 步骤 3: 烧录固件
  │   rkdeveloptool wl 0x40 uboot.img
  │   rkdeveloptool wl 0x80 boot.img
  │   rkdeveloptool wl 0x7a000 rootfs.img
  │   ...
  │
  ├─ 步骤 4 (可选): 烧录 OTP 密钥
  │   rkdeveloptool otp -s prod_key
  │
  └─ 步骤 5: 重启
      rkdeveloptool rd
```

### 7.2 固件签名 vs 产线烧录

```
开发环境:
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ dev.key  │───→│ mkimage  │───→│ firmware │
  │ (开发密钥) │    │ -f -k -r │    │ .img     │
  └──────────┘    └──────────┘    └──────────┘

产线环境:
  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ prod.key │───→│ 签名服务器│───→│ 校验     │───→│ 烧录     │
  │ (产品密钥) │    │ (离线)   │    │ fit_check │    │ rkdev     │
  └──────────┘    └──────────┘    └──────────┘    │ _tool    │
       ↑ 物理隔离                                          │
  ┌───────────────────────────────────────────────┘
  │ 产线 PC 不接触私钥, 只接收已签名的固件
```

---

## 八、常用工作流速查

### 8.1 快速部署单分区更新

```bash
# 场景: 只修改了内核 DTB, 需要快速部署

# PC 端:
# 1. 只编译 DTB
./build.sh kernel    # 或者: make dtbs in kernel dir

# 2. 重新打包 boot.img
./build.sh firmware

# 3. 方式 A: USB 烧录
./rkflash.sh boot

# 3. 方式 B: SCP + dd (更快, 不依赖 USB)
scp output/firmware/boot.img rooter@192.168.1.109:/tmp/
# 板端:
ssh rooter@192.168.1.109 "sudo dd if=/tmp/boot.img of=/dev/mmcblk0p3 bs=512"

# 4. 重启
ssh rooter@192.168.1.109 "sudo reboot"
```

### 8.2 快速部署 rootfs 更新

```bash
# PC 端:
# 1. 编译 rootfs
./build.sh buildroot

# 2. SCP + dd (rootfs 通常很大, 需要压缩传输)
scp output/firmware/rootfs.img rooter@192.168.1.109:/tmp/
# 或先压缩:
gzip -c output/firmware/rootfs.img | ssh rooter@192.168.1.109 "gunzip -c | sudo dd of=/dev/mmcblk0p7 bs=1M"

# 3. 重启
ssh rooter@192.168.1.109 "sudo reboot"
```

### 8.3 紧急恢复 (变砖恢复)

```bash
# 板子启动不了 (uboot 损坏/内核损坏)

# 进入 Maskrom 模式:
# 方式 1: 按住 MASKROM 按键 → 上电
# 方式 2: 短接 eMMC CLK 和 GND → 上电 → 松开

# 确认进入 Maskrom:
lsusb | grep Rockchip
# 预期: "ID 2207:350a" 或 "ID 2207:0011"

# 恢复步骤:
rkdeveloptool db rv1126b_spl_loader_v1.09.105.bin
sleep 2
rkdeveloptool gpt parameter-amp.txt
rkdeveloptool wl 0x40 uboot.img
rkdeveloptool wl 0x80 boot.img
# ... 其他分区
rkdeveloptool rd

# 注意: OTP 烧录后的板子只能烧录签名固件
```

---

## 九、工具版本与兼容性

| 工具 | 当前版本 | 来源 | 兼容性说明 |
|------|---------|------|-----------|
| `rkdeveloptool` | 社区版 (git) | GitHub/rockchip-linux | 所有 Rockchip SoC |
| `upgrade_tool` | v2.x | SDK rkbin/tools/ | 部分新特性支持 |
| `boot_merger` | v2.x | SDK rkbin/tools/ | 按芯片目录区分 |
| `trust_merger` | v2.x | SDK rkbin/tools/ | 与 BL31/BL32 版本匹配 |
| `mkimage` | U-Boot 2017.09 | SDK U-Boot 编译 | 版本与 U-Boot 匹配 |
| `fit_check_sign` | SDK 内部 | rkbin/tools/ | 与 FIT 格式兼容 |

---

## 十、思考题

1. **Maskrom vs Loader 模式**：进入 Maskrom 模式的 3 种方式中，为什么短接 eMMC CLK 可以"骗"过 BootROM？这对产线测试有什么意义？

2. **dd vs rkdeveloptool**：在板端已有系统时，`dd if=boot.img of=/dev/mmcblk0p3` 和 `rkdeveloptool wl 0x80 boot.img` 有什么区别？什么场景下必须用 rkdeveloptool？

3. **分区表一致性**：如果 parameter.txt 中的分区偏移和 boot.img 实际的 ITS/FIT 配置不一致，会出现什么问题？如何验证一致性？

4. **update.img 格式**：为什么 Rockchip 不直接用分区镜像的集合，而要用 update.img 这种打包格式？它的头部包含什么信息？

5. **OTP 烧录后**：如果 OTP 已烧录且私钥丢失，除了更换 SoC 还有没有其他恢复方法？产线应如何保护私钥安全？

---

## 相关笔记

- [[bsp-boot-flow]] — Boot Chain 全景 & 分区表
- [[bsp-uboot-adaptation]] — U-Boot 板级适配 (编译流程)
- [[bsp-spl-fit]] — SPL FIT 解析 (打包格式)
- [[bsp-uboot-secureboot]] — 安全启动 (签名工具)
- [[bsp-uboot-boottime]] — 启动速度 (部署优化)
- [[MOC-嵌入式Linux]] — 嵌入式 Linux 学习路线 MOC
