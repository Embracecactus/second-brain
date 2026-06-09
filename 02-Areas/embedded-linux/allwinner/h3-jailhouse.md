---
tags:
  - hardware
  - config
  - reference
category: hardware-design
created: 2026-06-09
---

# Jailhouse H3 嵌入式虚拟化配置指南

## 概述

本文档记录了在 Allwinner H3 (四核 Cortex-A7) 平台上部署 Jailhouse 分区 Hypervisor 的完整配置流程。Jailhouse 是由 Siemens 开发的轻量级 Linux-based 分区 Hypervisor，适用于嵌入式实时系统的硬件资源隔离。核心目标是将 H3 的 CPU 和内存进行物理划分，使 Root Cell (Linux) 与 Inmate Cell (裸机/RTOS) 安全共存。

项目涉及三个关键配置文件：
- `config/arm/h3-system.c` — Root Cell 系统级硬件资源声明
- `configs/arm/dts/inmate-h3-system.dts` — Inmate Cell 的 Device Tree 描述
- Linux 5.10 内核补丁 — 提供 HYP 层支持

## 关键知识点

### 1. Jailhouse 分区虚拟化原理

Jailhouse 采用物理分区模型，将硬件资源 (CPU、内存、外设) 静态划分给不同的 Cell。Root Cell 运行 Linux 并负责系统管理，Inmate Cell 运行实时任务或裸机程序。各 Cell 通过 IVSHMEM (Inter-VM Shared Memory) 进行通信。

### 2. 内存布局规划

H3 总物理内存 512MB，起始地址 `0x40000000`：

| 区域 | 起始地址 | 大小 | 用途 |
|------|---------|------|------|
| Linux 内核 | `0x40000000` | 500MB | Root Cell 可用内存 |
| Hypervisor | `0x5F900000` | 5MB | Jailhouse 运行时 (PSCI) |
| IVSHMEM 共享区 | `0x4F6F0000` | ~68KB | Cell 间通信 |
| IVSHMEM 网络区 | `0x4F700000` | 可变 | 虚拟网络 |

关键计算：505MB 起始地址 = `0x40000000 + 505 * 1024 * 1024 = 0x5F900000`。Linux 内核启动参数需限制 `mem=500M` 以避免与 Hypervisor 内存冲突。

### 3. GIC 中断控制器配置

H3 使用 ARM GICv2 (GIC-400)，各寄存器基地址：

| 组件 | 基地址 | 说明 |
|------|--------|------|
| GICD | `0x01C81000` | Distributor |
| GICC | `0x01C82000` | CPU Interface |
| GICH | `0x01C84000` | Hypervisor Interface |
| GICV | `0x01C86000` | Virtual CPU Interface |

- Maintenance IRQ: 25
- vPCI IRQ base: 108
- IRQ pin_bitmap 全部置 1 (`0xffffffff x4`)，表示 Root Cell 拥有所有 128 个中断

### 4. IVSHMEM 设备间通信

Root Cell 配置了两个 IVSHMEM PCI 设备：
- BDF (Bus:Device.Function): `0:0.0`
- `shmem_peers = 2`：1 个 Root + 1 个 Guest
- `shmem_dev_id = 0`：Root Cell 设备 ID
- 共享内存区域从 `mem_regions` 索引 0 开始，前 5 个条目为 IVSHMEM 通道

Guest Cell 的 IVSHMEM 配置要求：
- `shmem_dev_id` 必须与 Root 不同 (Guest = 1)
- `shmem_peers` 必须与 Root 一致
- 需要在 Guest 的 `mem_regions` 中声明对应的共享内存映射

### 5. Linux 5.10 内核补丁

补丁对 ARM32 Hypervisor stub 进行了关键修改：

- **新增 HVC 调用号**：`HVC_RESET_VECTORS = 2`，`HVC_STUB_HCALL_NR = 3`
- **移除 ZIMAGE 条件编译**：使 `__hyp_set_vectors` 在内核运行后仍可用
- **新增 `__hyp_reset_vectors`**：允许运行时重置 HYP 向量
- **导出符号**：`__boot_cpu_mode`、`__hyp_stub_vectors`、`__hyp_set_vectors` 等，供 Jailhouse 模块使用
- **内存子系统导出**：`ioremap_page_range`、`__get_vm_area_caller` 用于 Jailhouse 地址映射

## 技术细节

### Inmate Cell 硬件资源

Inmate Cell DTS 定义了以下设备：

| 设备 | 地址 | 中断 | 说明 |
|------|------|------|------|
| UART | `0x01C28000` | SPI 0 | 115200 调试串口 |
| CPU | core 2 | — | Cortex-A7, PSCI 启用 |
| 时钟 | — | — | 24MHz 固定振荡器 |

Inmate 仅分配 1 个 CPU 核心 (core 2)，通过 PSCI `smc` 方式进行电源管理。

### 系统属性标志

- `JAILHOUSE_SYS_VIRTUAL_DEBUG_CONSOLE`：启用虚拟调试控制台
- `JAILHOUSE_CON_ACCESS_MMIO | JAILHOUSE_CON_REGDIST_4`：MMIO 访问，寄存器间距 4 字节
- `JAILHOUSE_PCI_TYPE_IVSHMEM`：PCI 设备类型为 IVSHMEM
- `JAILHOUSE_MEM_IO | JAILHOUSE_MEM_IO_32`：32 位 IO 内存区域

### 编译配置

```shell
# Linux 内核
git clone -b v5.10 https://github.com/torvalds/linux.git --depth=1
make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabihf- sunxi_defconfig
make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabihf- -j32

# Jailhouse
git clone https://github.com/siemens/jailhouse.git
make KDIR=./linux-5.10 ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabihf- Werror=0
```

## 代码/配置片段

### Linux 启动参数 (bootargs)

```shell
setenv bootargs console=ttyS0,115200 console=tty0 console=tty1 \
  root=/dev/mmcblk0p2 rootfstype=ext4 elevator=deadline rootwait \
  mem=500M vmalloc=512M
```

- `mem=500M`：限制 Linux 可用内存为 500MB
- `vmalloc=512M`：增大 vmalloc 空间，满足 Jailhouse 模块加载需求

### hyp-stub.S 关键新增

```asm
ENTRY(__hyp_reset_vectors)
    mov r0, #HVC_RESET_VECTORS
    __HVC(0)
    ret lr
ENDPROC(__hyp_reset_vectors)
```

新增 `HVC_RESET_VECTORS` 处理逻辑，允许通过 HVC 调用重置 Hypervisor 向量表。

### 导出符号列表

```c
// arch/arm/kernel/armksyms.c
EXPORT_SYMBOL_GPL(__boot_cpu_mode);

// arch/arm/kernel/hyp-stub.S
EXPORT_SYMBOL_GPL(__hyp_stub_vectors);

// arch/arm64/kernel/hyp-stub.S
EXPORT_SYMBOL_GPL(__hyp_stub_vectors);

// mm/ioremap.c
EXPORT_SYMBOL_GPL(ioremap_page_range);

// mm/vmalloc.c
EXPORT_SYMBOL_GPL(__get_vm_area_caller);

// arch/x86/kernel/apic/apic.c
EXPORT_SYMBOL_GPL(lapic_timer_period);
```

## 相关链接

- Jailhouse 官方仓库：https://github.com/siemens/jailhouse
- Linux 5.10 内核：https://github.com/torvalds/linux (v5.10 分支)
- ARM GICv2 架构规范：ARM GIC Architecture Specification
- IVSHMEM 协议：Jailhouse IVSHMEM Documentation
- Allwinner H3 Datasheet：全志科技官方文档
- Device Tree 规范：https://www.devicetree.org/
