---
tags: [hardware, config, reference]
category: hardware-design
created: 2026-06-09
---
已生成 Obsidian 笔记文件：`/home/lijian/project/Obsidian/Jailhouse-H3-嵌入式虚拟化配置指南.md`

**提取的知识要点：**

1. **项目本质**：基于 Allwinner H3 (四核 Cortex-A7) 平台部署 Jailhouse 分区 Hypervisor，实现 Linux 与裸机/RTOS 的硬件级隔离共存。

2. **三个核心配置文件**：
   - `h3-system.c` — Root Cell 系统配置，定义了 18 个内存区域、1 个 GIC、2 个 IVSHMEM PCI 设备
   - `inmate-h3-system.dts` — Inmate Cell DTS，绑定 CPU2，24MHz 时钟，UART 调试串口
   - Linux 5.10 内核补丁 — 新增 `HVC_RESET_VECTORS`/`HVC_STUB_HCALL_NR`，导出 `__hyp_set_vectors` 等符号

3. **关键内存布局**：
   - Linux 限制 500MB (`mem=500M`)，Hypervisor 从 `0x5F900000` (505MB处) 占用 5MB
   - IVSHMEM 共享内存在 `0x4F6F0000`，网络共享在 `0x4F700000`

4. **GIC 配置**：GICv2 (GIC-400)，GICD `0x01C81000`，maintenance IRQ = 25，vPCI IRQ base = 108

5. **IVSHMEM 通信**：`shmem_peers=2` (1 Root + 1 Guest)，Root `shmem_dev_id=0`，Guest 须为 1

## 相关笔记

- [[h3]] — Allwinner H3 系统构建全栈笔记
- [[h618]] — H618 TV Box 定制 Linux 系统
- [[pcb]] — PCB 电源管理与电池充电电路设计
- [[power]] — 模块化可拆卸摄像头系统（电源设计）
