---
tags:
  - hardware
  - config
  - reference
  - jailhouse
  - hypervisor
  - allwinner
  - h3
  - embedded
category: hardware-design
created: 2026-06-09
updated: 2026-06-09
status: active
---

# Jailhouse H3 嵌入式虚拟化配置指南

## 项目/工具概述

Jailhouse 是由 Siemens 开发的轻量级 Linux-based partitioning hypervisor，能够在 ARM 平台上实现硬件级别的分区隔离。本项目基于 Allwinner H3（四核 Cortex-A7, 512MB RAM）平台部署 Jailhouse，实现 Linux Root Cell 与裸机/RTOS Inmate Cell 的硬件级隔离共存。通过 IVSHMEM（Inter-VM Shared Memory）机制实现 Cell 间的高速数据通信，适用于实时控制与通用计算混合部署的嵌入式场景。

源码仓库：`https://github.com/siemens/jailhouse.git`

## 技术栈 / 关键特性

- **SoC 平台**：Allwinner H3 — 四核 ARM Cortex-A7, ARMv7 架构
- **Hypervisor**：Jailhouse partitioning hypervisor（非抢占式，极低开销）
- **GIC**：ARM GIC-400 (GICv2)，支持虚拟中断注入
- **共享内存**：IVSHMEM PCI 设备（`shmem_peers=2`，1 Root + 1 Guest）
- **通信协议**：IVSHMEM Protocol — undefined（自定义协议）
- **串口调试**：8250-compatible UART（MMIO 地址 `0x01c28000`）
- **内核版本**：Linux 5.10（需打 Jailhouse 补丁）
- **交叉编译工具链**：`arm-none-linux-gnueabihf-`
- **PSCI**：ARM Power State Coordination Interface（用于 CPU 热插拔管理）

## 架构与设计

### 系统分区模型

```
┌─────────────────────────────────────────────────┐
│              Hardware (Allwinner H3)             │
│  CPU0-3  |  GIC-400  |  UART  |  SRAM  |  DRAM  │
├─────────────────────────────────────────────────┤
│              Jailhouse Hypervisor                │
│         (占用 5MB @ 0x5F900000)                  │
├──────────────────────┬──────────────────────────┤
│    Root Cell         │     Inmate Cell           │
│    "H3-Linux"        │     "Jailhouse cell on H3"│
│    CPU0,1,3          │     CPU2                  │
│    Linux 5.10        │     裸机/RTOS             │
│    500MB RAM         │     隔离内存区域          │
├──────────────────────┴──────────────────────────┤
│         IVSHMEM Shared Memory Region             │
│         @ 0x4F6F0000 (数据通道)                   │
│         @ 0x4F700000 (网络通道)                   │
└─────────────────────────────────────────────────┘
```

### 内存布局

| 区域 | 起始地址 | 大小 | 用途 |
|------|----------|------|------|
| Linux 可用 RAM | `0x40000000` | 500MB | Root Cell (mem=500M) |
| IVSHMEM 数据区 | `0x4F6F0000` | ~64KB | Cell 间共享数据通道 |
| IVSHMEM 网络区 | `0x4F700000` | JAILHOUSE_SHMEM_NET_REGIONS | 虚拟网络通信 |
| Hypervisor 区域 | `0x5F900000` | 5MB | Jailhouse 运行时 + PSCI |

> **重要**：内核 bootargs 中 `mem=500M` 必须小于 Hypervisor 起始地址（505MB），否则会覆盖 Hypervisor 内存区域。

## 核心知识点

### 1. Root Cell 系统配置 (`h3-system.c`)

Root Cell 是 Jailhouse 启动后的第一个 Cell，运行 Linux 系统，拥有对硬件的初始控制权。配置文件定义了以下关键组件：

- **CPU 掩码**：`0xf` 表示 4 个 CPU 全部归 Root Cell（后续通过 Jailhouse 创建 Inmate Cell 时再划分 CPU2）
- **内存区域**：共 18 个 `mem_regions`，包含 IVSHMEM 共享内存、SRAM、外设寄存器映射
- **IRQCHIP**：GIC 基地址 `0x01c81000`，pin_base=32，全 pin bitmap 开放
- **PCI 设备**：2 个 IVSHMEM PCI 设备，BDF 0:0:0，`shmem_dev_id=0`（Root Cell 标识）
- **vPCI IRQ base**：108 — 为虚拟 PCI 设备预留的中断号起始值，避免与硬件设备中断冲突

### 2. Inmate Cell DTS (`inmate-h3-system.dts`)

Inmate Cell 运行裸机程序或 RTOS，通过 Device Tree 描述硬件资源：

- **CPU 绑定**：仅使用 CPU2（`reg = <2>`），通过 PSCI 热插拔管理
- **时钟**：固定 24MHz 振荡器（`osc24M`）
- **调试串口**：`snps,dw-apb-uart` @ `0x01c28000`，SPI IRQ 0，reg-shift=2，reg-io-width=4
- **中断控制器**：引用 GIC（`interrupt-parent = <&gic>`）
- **PSCI 兼容**：`arm,psci-0.2`，method=smc

### 3. IVSHMEM 通信机制

IVSHMEM 是 Jailhouse 实现 Cell 间通信的核心机制：

- **共享内存区域**：前 5 个 `mem_regions` 定义了 IVSHMEM 数据通道
  - `0x4F6F0000` (4KB)：只读，用于配置/状态区
  - `0x4F6F1000` (36KB)：读写，主数据区
  - `0x4F6FA000` (8KB)：读写，扩展数据区
  - `0x4F6FC000` / `0x4F6FE000` (各 8KB)：只读，状态/控制区
- **Peer 配置**：`shmem_peers=2`（1 Root Cell + 1 Inmate Cell）
- **设备 ID**：Root Cell 的 `shmem_dev_id=0`，Guest Cell 必须设为 `shmem_dev_id=1`
- **协议类型**：`JAILHOUSE_SHMEM_PROTO_UNDEFINED`（自定义协议）

### 4. GIC (Generic Interrupt Controller) 配置

| 参数 | 值 | 说明 |
|------|-----|------|
| GIC 版本 | GICv2 (GIC-400) | ARM Cortex-A7 标配 |
| GICD 基地址 | `0x01C81000` | Distributor |
| GICC 基地址 | `0x01C82000` | CPU Interface |
| GICH 基地址 | `0x01C84000` | Hypervisor Interface |
| GICV 基地址 | `0x01C86000` | Virtual CPU Interface |
| Maintenance IRQ | 25 | Hypervisor 维护中断 |
| vPCI IRQ base | 108 | 虚拟 PCI 设备中断起始 |

### 5. Linux 5.10 内核补丁

Jailhouse 需要对 Linux 内核打补丁以支持 Hypervisor 交互，补丁涉及以下文件：

| 文件 | 修改内容 |
|------|----------|
| `arch/arm/include/asm/virt.h` | 新增 `HVC_RESET_VECTORS=2`、`HVC_STUB_HCALL_NR=3`，声明 `__hyp_set_vectors`/`__hyp_reset_vectors` |
| `arch/arm/kernel/armksyms.c` | 导出 `__boot_cpu_mode` 符号（`EXPORT_SYMBOL_GPL`） |
| `arch/arm/kernel/hyp-stub.S` | 新增 `__hyp_reset_vectors` 入口，移除 `#ifdef ZIMAGE` 限制使 HVC_SET_VECTORS 在内核模式下也可用，导出 `__hyp_stub_vectors` |
| `arch/arm64/kernel/hyp-stub.S` | 导出 `__hyp_stub_vectors` 符号 |
| `arch/x86/kernel/apic/apic.c` | 导出 `lapic_timer_period` 符号 |
| `mm/ioremap.c` | 导出 `ioremap_page_range` 符号 |
| `mm/vmalloc.c` | 导出 `__get_vm_area_caller` 符号 |

### 6. 外设寄存器映射（Root Cell 保留）

| 外设 | 物理地址 | 大小 | 标志 |
|------|----------|------|------|
| SRAM A2 | `0x00040000` | 48KB | RW, IO |
| SRAM C | `0x01D00000` | 512KB | RW, IO |
| System Control | `0x01C00000` | 64KB | RW, IO |
| Clock | `0x01C20000` | 1KB | RW, IO, IO_32 |
| PIO (pinctrl) | `0x01C20800` | 1KB | RW, IO, IO_32 |
| Watchdog | `0x01C20CA0` | 32B | RW, IO, IO_32 |

## 关键代码/配置片段

### U-Boot 启动参数（限制 Linux 内存）

```shell
setenv bootargs console=ttyS0,115200 console=tty0 console=tty1 \
  root=/dev/mmcblk0p2 rootfstype=ext4 elevator=deadline rootwait \
  mem=500M vmalloc=512M
```

### 编译 Jailhouse

```shell
# 克隆仓库
git clone https://github.com/siemens/jailhouse.git

# 将 h3-system.c 复制到 config/arm/ 目录
# 将 inmate-h3-system.dts 复制到 configs/arm/dts/ 目录

# 编译（指定内核源码路径）
make KDIR=./linux-5.10 ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabihf- Werror=0
```

### 编译 Linux 5.10 内核（含 Jailhouse 补丁）

```shell
# 克隆内核源码
git clone -b v5.10 https://github.com/torvalds/linux.git --depth=1

# 应用 Jailhouse 补丁（参见 README.md 中的 diff 内容）
patch -p1 < jailhouse-h3.patch

# 配置并编译
make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabihf- sunxi_defconfig
make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabihf- -j32
```

### Root Cell IVSHMEM PCI 设备配置结构

```c
.pci_devices = {
    {
        .type = JAILHOUSE_PCI_TYPE_IVSHMEM,
        .bdf = 0 << 3,                    // Bus 0, Device 0, Function 0
        .bar_mask = JAILHOUSE_IVSHMEM_BAR_MASK_INTX,
        .shmem_regions_start = 0,         // mem_regions 中首个 IVSHMEM 区域索引
        .shmem_dev_id = 0,               // Root Cell = 0, Guest Cell = 1
        .shmem_peers = 2,                // 总 peer 数 = 1 Root + 1 Guest
        .shmem_protocol = JAILHOUSE_SHMEM_PROTO_UNDEFINED,
    },
},
```

## 使用方法 / 构建步骤

### 完整部署流程

1. **准备交叉编译环境**：安装 `arm-none-linux-gnueabihf-` 工具链
2. **克隆并打补丁 Linux 5.10 内核**：应用 Jailhouse hyp-stub 补丁
3. **编译内核**：使用 `sunxi_defconfig`，编译 zImage 和 dtb
4. **克隆 Jailhouse 仓库**：将 `h3-system.c` 放入 `config/arm/`，`inmate-h3-system.dts` 放入 `configs/arm/dts/`
5. **编译 Jailhouse**：指定 `KDIR` 指向编译好的内核源码树
6. **烧写系统**：将内核、dtb、rootfs 写入 SD 卡（`/dev/mmcblk0p2`）
7. **配置 U-Boot**：设置 bootargs 限制 `mem=500M`，保留 Hypervisor 内存
8. **启动 Linux**：加载 Jailhouse 内核模块
9. **激活 Hypervisor**：`jailhouse enable /path/to/h3-system.cell`
10. **创建 Inmate Cell**：`jailhouse cell create /path/to/inmate.cell`
11. **启动 Inmate**：`jailhouse cell load <cell-name> /path/to/baremetal.bin`
12. **运行 Inmate**：`jailhouse cell start <cell-name>`

### 调试串口

Inmate Cell 通过 UART0 (`0x01c28000`) 输出调试信息，波特率 115200。可通过 USB-TTL 适配器连接 PC 端串口工具查看。

## 相关笔记

- [[h3]] — Allwinner H3 系统构建全栈笔记（内核编译、U-Boot 配置、外设驱动）
- [[h618]] — H618 TV Box 定制 Linux 系统（同类 ARM 平台参考）
- [[pcb]] — PCB 电源管理与电池充电电路设计（硬件层配套设计）
- [[power]] — 模块化可拆卸摄像头系统（电源设计与嵌入式系统集成）
