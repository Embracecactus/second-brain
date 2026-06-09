---
title: RV1126B 小雨 IPC50 开发笔记
created: 2026-06-09
updated: 2026-06-09
status: active
tags:
  - rockchip
  - rv1126b
  - embedded-linux
  - amp
  - ipc
  - mcu
  - rtos
  - buildroot
  - myzr
soc: Rockchip RV1126B
board: MYZR IPC50
sdk_version: RV1126B_LINUX_SDK_RELEASE_V1.2.0_20260401
kernel: linux-6.1
---

# RV1126B 小雨 IPC50 开发笔记

## 项目概述

RV1126B 是瑞芯微推出的新一代高性能 AI 视觉处理器，集成四核 Cortex-A7 + RISC-V MCU，内置 2.0 TOPS NPU，适用于智能安防、边缘计算及工业视觉场景。明远智睿（MYZR）基于该 SoC 设计了 IPC50 核心板方案，采用 AMP（非对称多处理）架构实现 Linux 与 RTOS 双系统协同运行，兼顾通用计算与实时控制需求。本笔记基于 MYZR 官方 SDK（`RV1126B_LINUX_SDK_RELEASE_V1.2.0_20260401`）整理，涵盖 SDK 构建流程、MCU 子系统开发、AMP 架构配置及常见问题排查。

## 技术栈

| 类别 | 技术/工具 | 说明 |
|------|-----------|------|
| SoC | Rockchip RV1126B | 四核 Cortex-A7 @ 1.5GHz + RISC-V MCU @ 400MHz |
| NPU | 内置 NPU | 2.0 TOPS INT8 推理性能 |
| ISP | 内置 ISP | 支持双路摄像头输入，最高 8MP |
| 主系统 OS | Buildroot Linux | 基于 Linux 6.1 内核 |
| 实时子系统 | RISC-V MCU (RT-Thread) | 运行于独立核心，负责实时任务 |
| 构建系统 | Buildroot + Makefile | 一键构建 Linux 与 MCU 固件 |
| U-Boot | Rockchip U-Boot 2017.09 | 支持 AMP 多固件加载 |
| 设备树 | DTS/DTSI | 板级硬件描述 |
| 交叉编译工具链 | aarch64-linux-gnu-gcc 12.1 | 主系统编译 |
| MCU 工具链 | riscv64-unknown-elf-gcc 12.1 | MCU 子系统编译 |
| 烧录工具 | RKDevTool / upgrade_tool | Windows / Linux 烧录 |
| 调试接口 | UART / JTAG / GDB | 串口调试与在线调试 |
| 通信机制 | RPMsg / Mailbox | 核间通信 |

## 架构与设计

### 系统启动流程

```
[BootROM] → [TPL/SPL] → [U-Boot] → [Linux Kernel] → [Rootfs]
                            ↓
                      [加载 amp_mcu.its]
                            ↓
                    [启动 RISC-V MCU] → [RT-Thread]
```

### AMP 架构

| 组件 | 运行核心 | 职责 |
|------|----------|------|
| Linux | Cortex-A7 x4 | 通用计算、网络、存储、AI 推理调度 |
| RT-Thread | RISC-V MCU | 实时音视频采集、IO 控制、低延迟任务 |
| RPMsg | 共享内存 + Mailbox | 核间消息传递与数据交换 |
| NPU 驱动 | Linux 用户态 | 模型加载、推理调度 |

### 硬件组成

| 模块 | 规格 |
|------|------|
| CPU | RV1126B, Cortex-A7 x4 + RISC-V |
| RAM | 板载 DDR4（容量视配置） |
| Storage | eMMC / SPI NOR Flash / SPI NAND |
| Camera | 双路 MIPI CSI 输入 |
| Network | 10/100M 以太网、Wi-Fi（模组可选） |
| Display | MIPI DSI / CVBS 输出 |
| Audio | I2S / PDM 接口 |
| USB | USB 2.0 OTG / Host |
| GPIO/SPI/I2C/UART | 通用外设扩展 |

### SDK 文件资源树

```
RV1126B_LINUX_SDK_RELEASE_V1.2.0_20260401/
├── buildroot/                # Buildroot 根文件系统
├── app/                      # 用户态应用（IPC 业务程序等）
├── kernel/                   # Linux 6.1 内核源码
├── u-boot/                   # U-Boot 引导加载程序
├── external/                 # 外部库与第三方组件
├── device/rockchip/          # 芯片与板级配置
│   ├── common/               # 通用脚本（mkfirmware.sh 等）
│   ├── rv1126b/              # RV1126B 芯片级配置
│   └── rv1126b-myir/         # MYZR IPC50 板级配置
├── prebuilts/                # 预编译工具链
├── tools/                    # 烧录与打包工具
├── docs/                     # 官方文档
├── amp_mcu.its               # AMP 多固件打包描述文件
├── mkfirmware.sh             # 固件打包脚本
├── build.sh                  # 主构建入口脚本
└── README.md                 # SDK 快速使用指南
```

## 核心知识点

### 1. SDK 版本与获取

MYZR 提供的 SDK 基于 Rockchip 官方 SDK 定制，当前版本为 `RV1126B_LINUX_SDK_RELEASE_V1.2_0_20260401`。SDK 通过百度网盘或 MYZR 官网下载，解压后需确保磁盘空间大于 50GB（含编译产物）。

### 2. 板级配置路径

板级定制文件位于 `device/rockchip/rv1126b-myir/`，包含：
- `BoardConfig.mk` — 主板配置（存储介质、分区表、内核 DTB 选择）
- `parameter.txt` — 分区与启动参数
- `firmware/` — 预置固件（MiniLoader、Trust 等）

### 3. U-Boot 构建系统

U-Boot 采用 Rockchip 定制版本（2017.09 基线），支持 AMP 多固件加载。关键配置项：
- `CONFIG_AMP_OPTEE_SUPPORT` — AMP + OP-TEE 联合启动
- `CONFIG_AMP_MCU_SUPPORT` — MCU 固件加载支持
- 构建命令：`./build.sh uboot` 或单独进入 `u-boot/` 目录执行 make

### 4. MCU 子系统编译

MCU 固件独立于 Linux 构建，使用 `riscv64-unknown-elf-gcc` 工具链。编译产物为 `mcu.bin`，需通过 `amp_mcu.its` 打包进 FIT 镜像。MCU 代码位于 `external/rv1126b_mcu/`（路径因 SDK 版本可能略有差异）。

### 5. PREEMPT_RT 补丁

Linux 主系统可选打 PREEMPT_RT 实时补丁，降低内核延迟。配置路径：`kernel/` 目录下执行 `make menuconfig`，选择 `Fully Preemptible Kernel (RT)`。注意：当 MCU 已承担实时任务时，RT 补丁的优先级降低。

### 6. 设备树配置

板级 DTS 文件位于 `kernel/arch/arm64/boot/dts/rockchip/`，MYZR IPC50 对应文件为 `rv1126b-myir-ipc50.dts`。关键节点：
- `reserved-memory` — MCU 与 Linux 共享内存区域划分
- `mailbox` — 核间通信 Mailbox 控制器
- `rpmsg` — RPMsg 虚拟通道配置

## 关键代码与配置

### amp_mcu.its（AMP 多固件打包描述）

```its
/dts-v1/;

/ {
	description = "FIT source file for AMP MCU";
	images {
		mcu {
			description = "mcu";
			data = /incbin/("mcu.bin");
			type = "firmware";
			arch = "riscv";
			os = "rtos";
			compression = "none";
			load = <0x38000000>;
			entry = <0x38000000>;
			hash-1 {
				algo = "sha256";
			};
		};
	};
	configurations {
		default = "conf";
		conf {
			description = "mcu";
			firmware = "mcu";
			loadables = "mcu";
			signature-1 {
				algo = "sha256,rsa2048";
				//key-name-hint = "dev";
			};
		};
	};
};
```

### 分区表（parameter.txt 节选）

```
CMDLINE: ... root=/dev/mmcblk0p5 rootfstype=ext4 ...
0x00002000@0x00002000(parameter)
0x00002000@0x00004000(Miniloader)
0x00004000@0x00006000(Trust)
0x00004000@0x0000a000(U-Boot)
0x00002000@0x0000e000(logo)
0x00004000@0x00010000(boot)
0x00002000@0x00014000(dtbo)
0x00008000@0x00016000(misc)
0x00080000@0x0001e000(rootfs)
0x00040000@0x0009e000(oem)
0x00040000@0x000de000(userdata)
```

### MCU 固件验证命令

```bash
# 检查 MCU 固件是否正确加载
cat /sys/kernel/debug/rpmsg/amp/status

# 查看 AMP 子系统状态
cat /proc/amp/rv1126b

# MCU 日志输出（通过共享内存调试串口）
cat /dev/amp_log
```

## 构建步骤

1. **解压 SDK**：`tar -xzf RV1126B_LINUX_SDK_RELEASE_V1.2_0_20260401.tar.gz`，执行 `./build.sh` 首次初始化环境依赖。

2. **选择板级配置**：执行 `./build.sh lunch`，选择 `rv1126b_myir_ipc50` 对应的配置编号。

3. **构建完整固件**：执行 `./build.sh` 一键构建（含 U-Boot、Kernel、Rootfs、MCU）。也可分步构建：
   ```bash
   ./build.sh uboot       # 构建 U-Boot
   ./build.sh kernel      # 构建内核与模块
   ./build.sh rootfs      # 构建根文件系统
   ./build.sh mcu         # 构建 MCU 固件（需确认脚本支持）
   ```

4. **打包固件**：执行 `./build.sh firmware` 或 `./mkfirmware.sh`，生成 `rockdev/` 目录下的完整镜像。

5. **烧录**：使用 RKDevTool（Windows）或 `upgrade_tool`（Linux）通过 USB 进入 Loader 模式烧录，或使用 SD 卡启动。

## 故障排查

| 问题现象 | 可能原因 | 解决方案 |
|----------|----------|----------|
| MCU 固件加载失败 | `amp_mcu.its` 路径或 `load` 地址错误 | 检查 ITS 文件中 `load` 地址是否与 MCU 链接脚本一致 |
| Linux 启动卡在 Waiting for rootfs | 分区表 `parameter.txt` 中 rootfs 分区偏移/大小不匹配 | 核对 `parameter.txt` 与实际镜像大小 |
| RPMsg 通信超时 | 共享内存区域划分冲突 | 检查 DTS `reserved-memory` 中 MCU 与 Linux 区域不重叠 |
| 构建报工具链找不到 | 交叉编译工具链未安装或 PATH 未配置 | 执行 `source envsetup.sh` 或检查 `prebuilts/` 路径 |

## 开发资源

### MYZR 官方文件索引

| 文件名 | 说明 |
|--------|------|
| `MYZR-RV1126B-Linux-AMP开发手册（IPC50）V1.0.pdf` | AMP 架构与 IPC50 开发指南 |
| `MYZR-RV1126B-Linux-软件开发手册（IPC50）V1.0.pdf` | SDK 使用与软件开发流程 |
| `MYZR-RV1126B-Linux-烧录工具使用手册（IPC50）V1.0.pdf` | 固件烧录操作说明 |
| `MYZR-RV1126B-Linux-硬件手册（IPC50）V1.0.pdf` | 硬件接口与引脚定义 |
| `MYZR-RV1126B-Linux-系统开发手册（IPC50）V1.0.pdf` | 系统定制与内核配置 |

### 工具清单

| 工具 | 用途 | 获取方式 |
|------|------|----------|
| RKDevTool | Windows USB 烧录 | Rockchip 官方 |
| upgrade_tool | Linux USB 烧录 | SDK `tools/` 目录 |
| SecureCRT / minicom | 串口调试 | 官网下载 |
| aarch64-linux-gnu-gcc | Linux 主系统编译 | SDK `prebuilts/` |
| riscv64-unknown-elf-gcc | MCU 固件编译 | SDK `prebuilts/` |
| Git | 版本管理 | 系统包管理器 |

### 本地参考路径

- 板级配置：`device/rockchip/rv1126b-myir/`
- 内核设备树：`kernel/arch/arm64/boot/dts/rockchip/rv1126b-myir-ipc50.dts`
- MCU 源码：`external/rv1126b_mcu/`（路径因版本而异）
- AMP 打包配置：`amp_mcu.its`
- 构建入口：`build.sh`

## 相关笔记

- [[Rockchip RV1126 系列概述]]
- [[RV1126 AMP 架构详解]]
- [[RISC-V MCU 开发入门]]
- [[Linux 内核 PREEMPT_RT 补丁]]
- [[Buildroot 定制根文件系统]]
- [[嵌入式 Linux 设备树编写规范]]
- [[RKDevTool 烧录指南]]
