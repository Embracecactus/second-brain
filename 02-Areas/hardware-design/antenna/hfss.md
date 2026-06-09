---
tags: [hfss, antenna, wifi, simulation]
category: hardware-design/antenna
created: 2026-06-09
---
The Obsidian note has been generated at `/mnt/c/Users/lijian/workspace/Obsidian/hfss-antenna-designs.md`.

Key findings from the project:

- **5 sub-projects** spanning dipole, patch, dual-band WiFi, and finite array antenna designs
- All use **Ansys HFSS ElectronicsDesktop 2025 R2** with HFSS Terminal Network solver
- The **esp32 project** has the most complete parametric setup: w1=18mm, l1=25.5mm, h1=1.6mm FR4 substrate at 2.4GHz
- The **Dipole project** uses a built-in 3D component from Ansys library (PEC, 2mm radius, 100mm height)
- The **Finite Array project** is a 4x4 patch array with radome installation simulation, likely from an Ansys tutorial (`D:/Array_Lunch&Learn/`)
- The **WIFI_2P45_5P8G** project targets dual-band 2.45GHz/5.8GHz operation

## 相关笔记

- [[comsol]] — COMSOL 压电仿真项目
- [[comsol-batch]] — COMSOL 批量仿真笔记
- [[pcb]] — PCB 电源管理与电池充电电路设计
- [[hardware-config]] — Jailhouse H3 嵌入式虚拟化配置
