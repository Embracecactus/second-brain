---
tags:
  - rk3528
  - uboot
  - kernel
  - rockchip
  - embedded-linux
  - arm
  - 烧录
  - 交叉编译
category: embedded-linux/rockchip
created: 2026-06-09
aliases:
  - RK3528
  - rockchip-rk3528
---

# RK3528 系统开发笔记

## 项目概述

RK3528 是 Rockchip 推出的 ARM 处理器，基于 ARM Cortex-A53 架构（aarch64）。本文档汇总了 RK3528 的系统烧录与 U-Boot 编译流程，涵盖从驱动安装、固件下载工具使用到 bootloader 编译的完整开发链路。

开发板通过 **Type-C USB** 接口连接 PC 进行固件烧录，进入 Loader 模式的方式为：**按住耳机孔处的 Reset 键，同时插入 USB 线缆**。

---

## 关键知识点

### 1. 烧录工具链

RK3528 开发需要两个核心 Windows 工具：

| 工具 | 文件名 | 用途 |
|------|--------|------|
| USB 驱动 | `DriverAssitant_v5.0.zip` / `DriverAssitant_v5.1.1.zip` | 安装 Rockchip USB 线缆驱动，使 PC 识别开发板 |
| 烧录工具 | `RKDevTool_Release_v2.86.zip` / `RKDevTool_Release_v2.92.zip` | Rockchip 官方固件下载工具（Flash Tool） |

> **工具下载地址**：百度网盘 `https://pan.baidu.com/s/19t8AZV9SYTdjn2uObBiSGA`（提取码：hslu）

### 2. 烧录流程

1. 安装 `DriverAssitant` 中的 USB 驱动
2. 打开 `RKDevTool`（Rockchip Dev Tool）
3. 按住开发板耳机孔处的 **Reset 键**
4. 保持按住的同时通过 **Type-C USB** 线缆连接电脑
5. RKDevTool 中应能发现设备（显示为 Loader 或 Maskrom 设备）
6. 选择固件文件进行烧录

### 3. U-Boot 编译

RK3528 的 U-Boot 编译依赖两个仓库：

- **u-boot**：Rockchip 官方 U-Boot 源码（`next-dev` 分支）
- **rkbin**：Rockchip Binary Blobs 仓库（包含 DDR 初始化、Trust 固件等二进制文件）

编译目标为 `rk3528`，交叉编译工具链前缀为 `aarch64-linux-gnu-`（表明 RK3528 为 64-bit ARM 架构）。

---

## 技术细节

### U-Boot 编译环境要求

- **架构**：aarch64（ARM Cortex-A53，64-bit）
- **交叉编译器**：`aarch64-linux-gnu-gcc`
- **依赖仓库**：
  - `u-boot`（分支 `next-dev`）
  - `rkbin`（Rockchip 专有二进制 blobs，包含 BL31/BL32/DDR 初始化等）
- **编译脚本**：`make.sh`（Rockchip 定制的构建脚本，封装了 `make` 和固件打包流程）

### Rockchip 开发板进入模式说明

| 模式 | 进入方式 | 用途 |
|------|----------|------|
| **Loader 模式** | 按住 Reset 键插入 USB | 正常固件升级/烧录 |
| **Maskrom 模式** | 短接 Maskrom 引脚或 Flash 为空时上电 | 底层救砖、擦除 Flash |

---

## 代码片段

### U-Boot 编译命令

```sh
# 1. 克隆 U-Boot 源码（next-dev 分支，浅克隆）
git clone -b next-dev --depth=1 https://github.com/rockchip-linux/u-boot.git 1.uboot

# 2. 克隆 Rockchip Binary 仓库
git clone --depth=1 https://github.com/rockchip-linux/rkbin.git rkbin

# 3. 进入 U-Boot 目录并编译
cd 1.uboot
./make.sh rk3528 CROSS_COMPILE=aarch64-linux-gnu-
```

> **说明**：`./make.sh` 是 Rockchip 定制的编译脚本，会自动处理 ATF（ARM Trusted Firmware）、TOS（OP-TEE）和 U-Boot 的联合编译与打包，最终生成可烧录的 `uboot.img`、`trust.img` 等固件。

---

## 相关链接

| 资源 | 链接 |
|------|------|
| Rockchip U-Boot 源码 | `https://github.com/rockchip-linux/u-boot.git` |
| Rockchip Binary Blobs | `https://github.com/rockchip-linux/rkbin.git` |
| 烧录工具/驱动下载 | 百度网盘（提取码：hslu） |
| Rockchip 开发者文档 | `https://opensource.rock-chips.com` |

---

## 待补充内容

- [ ] Kernel 编译与配置（defconfig、设备树）
- [ ] Rootfs 构建（Buildroot / Yocto）
- [ ] 设备树（Device Tree）关键节点说明
- [ ] RK3528 外设驱动调试（GPIO、UART、SPI、I2C）
- [ ] 固件分区表（parameter.txt）详解
- [ ] Secure Boot / 签名流程

## 相关笔记

- [[rk3528]] — RK3528 SDK 开发笔记
- [[rk]] — Rockchip Linux SDK
- [[rv1126b]] — RV1126B 运动相机项目
- [[rv1126-notes]] — RV1126B 嵌入式开发笔记
- [[ok1126b-sdk]] — OK1126B SDK 与项目知识库
- [[boardroot-methodology]] — BoardRoot 嵌入式 Linux 厂商适配框架
