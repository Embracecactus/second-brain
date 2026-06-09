---
title: PCB 电源管理与电池充电电路设计
tags:
  - pcb
  - hardware-design
  - power-management
  - battery-charging
  - usb-c
  - buck-converter
category: hardware-design/pcb
created: 2026-06-09
status: draft
project: pcb
---

# PCB 电源管理与电池充电电路设计

## 项目概述

该项目是一个基于 KiCad 的 PCB 电源管理板设计，核心功能为 USB-C 输入供电、锂离子电池充电管理（BQ27421）以及多路 DC-DC 降压转换（5V / 3.3V），同时具备 I2C 电池监控接口和 LED 指示功能，适用于嵌入式系统的电源子系统。

## 技术栈

- **EDA 工具**: KiCad（基于 `.tel` 网表格式推断）
- **核心 IC**:
  - **U2**: BQ27421（TI 电池电量计，I2C 接口，SON-12 封装）
  - **U3**: DC-DC Buck 转换器（VQFN-7 封装，用于 VBAT_5V 到 SRN 的降压路径）
  - **U4**: DC-DC Buck 控制器（TSOT-23-6 封装，驱动外部 MOSFET Q1/Q2）
  - **U5**: LDO 或 DC-DC 转换器（TSOT-23-6 封装，输出 VCC3V3_SYS）
  - **U6**: USB-C 电源路径管理 IC（ESOP-8 封装，带 VBUS 控制和 LED 驱动）
- **被动元件**: 0402/0603/0805 封装的电阻电容，SMD 功率电感
- **连接器**: USB-C（HYCW575-USBC06-680B）、12Pin 2.54mm 排针（H1/H3）、2Pin 排针（H2）
- **接口协议**: I2C（BQ27421_SCL / BQ27421_SDA）

## 架构与关键设计

### 电源拓扑

```
USB-C (VBUS_5V)
  ├──> U6 (电源路径管理) ──> VBUS_5V 供电域
  │     ├──> Q1 (MOSFET, SOT-23-3) ──> VCC5V0_SYS (5V 系统电源)
  │     └──> U6.LED 驱动 ──> LED2, LED3 (限流: R15=1kΩ, R16=1kΩ)
  │
  └──> 充电路径 ──> BQ27421 (U2) ──> BAT (电池)
        ├──> U3 (Buck) ──> L1 (1uH) ──> VBAT_5V
        └──> I2C 监控 (H1 排针引出)

BAT (电池)
  └──> Q2 (MOSFET) ──> VBAT_5V ──> U4 (Buck) ──> VCC5V0_SYS

VCC5V0_SYS
  └──> U5 (Buck/LDO) ──> L2 (3.3uH) ──> VCC3V3_SYS (3.3V 系统电源)
```

### 关键电源域

| 电源域 | 电压 | 来源 | 用途 |
|--------|------|------|------|
| `VBUS_5V` | 5V | USB-C 输入 | 主供电、充电 |
| `VBAT_5V` | 5V | 电池经 U3 Buck | 电池备电输出 |
| `VCC5V0_SYS` | 5V | Q1/Q2 MOSFET 选择 | 系统 5V 总线 |
| `VCC3V3_SYS` | 3.3V | U5 转换 | 系统 3.3V 数字电源 |
| `BAT` | 3.7~4.2V | 锂电池 | BQ27421 监控 |
| `SRN` | - | 电流检测 | 采样电阻 R6 (10mΩ) |

### 电流检测路径

- **R6** (10mΩ, 0603): 串联在 BAT 与 SRN 之间，用于 BQ27421 的电流采样
- SRN 网络连接至 U2.7、U3.5、U3.7、U6.5，实现多路电流感知
- **R7** (732kΩ) 和 **R8** (100kΩ): 构成 U3 的反馈分压网络

## 核心知识点

### 1. BQ27421 电池电量计

BQ27421 是 TI 的单节锂离子电池电量计，通过 I2C 接口（SCL/SDA）与主控通信：
- **BAT** 引脚直接连接电池正极
- **BIN** (H1.12) 为电池温度检测输入，通过 R4 (10kΩ) 上拉
- **GPOUT** 为可编程中断输出，通过 R3 (10kΩ) 上拉
- **SRN** 为电流检测负端，连接 10mΩ 采样电阻

### 2. USB-C 电源路径管理 (U6)

U6 (ESOP-8) 负责 USB-C 的电源路径切换：
- **VBUS_5V** 直接从 USB-C 的 B4/B9 引脚获取
- U6.4/U6.8 连接 VBUS，控制充电和供电路径
- 内置 LED 驱动（U6.6 -> LED3, U6.7 -> LED2），用于充电状态指示
- **LED1** 独立连接 VCC3V3_SYS 作为系统电源指示

### 3. 双路 MOSFET 电源切换

Q1 和 Q2 (SOT-23-3) 作为高端开关，由 U4 (TSOT-23-6) 控制：
- Q1: 控制 VBUS_5V -> VCC5V0_SYS 路径
- Q2: 控制 VBAT_5V -> VCC5V0_SYS 路径
- U4 的 EN 引脚通过 R5 (470kΩ) 上拉至 VCC5V0_SYS
- 实现 USB 供电与电池供电的自动切换（Power Path Management）

### 4. Buck 转换器反馈网络

U5 的反馈分压决定输出电压：
- R10 (232kΩ) / R11 (30kΩ) 构成分压比：$V_{out} = 0.6V \times (1 + R10/R11) \approx 4.64V$
  - 注：该分压比偏高，可能需要根据实际 IC datasheet 确认参考电压
- R9 (47kΩ) 可能用于环路补偿或软启动
- C9 (22pF) 为反馈去耦电容

### 5. 外部接口定义 (H1 12Pin 排针)

| Pin | 信号 | 说明 |
|-----|------|------|
| H1.1/3/5 | VBUS_5V | USB 5V 电源输出 |
| H1.7 | SRN | 电流检测端 |
| H1.9 | BQ27421_SDA | I2C 数据线 |
| H1.10 | GPOUT | 电池计中断输出 |
| H1.11 | BQ27421_SCL | I2C 时钟线 |
| H1.12 | BIN | 电池温度检测 |

## 重要代码片段

### 网表关键连接（Netlist 片段）

```
# BQ27421 I2C 上拉至 3.3V
'BQ27421_SCL' ; H1.11 R1.2 U2.2
'BQ27421_SDA' ; H1.9 R2.2 U2.1
# R1, R2 另一端连接 VCC3V3_SYS (4.7kΩ 上拉)

# 电流采样电阻
SRN ; C1.2 H1.7 H2.2 L1.2 R6.2 U2.7 U3.5 U3.7 U6.5
BAT ; C3.2 R6.1 U2.6 U2.8

# USB-C VBUS 直连
'VBUS_5V' ; H1.1 H1.3 H1.5 Q1.3 U6.4 U6.8 USB1.B4 USB1.B9

# 5V/3.3V 系统电源分配
'VCC5V0_SYS' ; C2.2 C7.2 C8.2 H3.7 H3.9 H3.11 Q1.2 Q2.2 R5.2 R9.2 U4.6 U5.5
'VCC3V3_SYS' ; C9.2 C11.2 C12.2 H3.1 H3.3 H3.5 L2.2 R1.1 R2.1 R3.2 R4.2 ...
```

### BOM 关键器件清单

| 编号 | 封装 | 值/型号 | 用途 |
|------|------|---------|------|
| U2 | SON-12 | BQ27421 | 电池电量计 |
| U3 | VQFN-7 | DC-DC Buck | 电池到 5V 升压 |
| U4 | TSOT-23-6 | Buck Controller | MOSFET 驱动 |
| U5 | TSOT-23-6 | DC-DC | 5V 到 3.3V 降压 |
| U6 | ESOP-8 | USB-C PMIC | 电源路径管理 |
| L1 | 7.1x6.6mm | 1uH | U3 功率电感 |
| L2 | 4.4x4.2mm | 3.3uH | U5 功率电感 |
| R6 | 0603 | 10mΩ | 电流采样 |
| USB1 | SMD | USB-C 680B | USB Type-C 接口 |

## 构建/运行方法

### 设计流程

1. **原理图设计**: 使用 KiCad Schematic Editor 绘制原理图
2. **网表生成**: 导出 `.tel` 格式网表文件（如 `Netlist_Schematic1_2026-04-15.tel`）
3. **PCB Layout**: 在 KiCad PCB Editor 中进行布局布线
4. **DRC 检查**: 运行设计规则检查，确保无间距违规
5. **Gerber 导出**: 生成生产文件发往 PCB 制造商

### 注意事项

- 功率电感 L1 (1uH, 7.1x6.6mm) 和 L2 (3.3uH, 4.4x4.2mm) 需关注饱和电流规格
- 电流采样电阻 R6 (10mΩ) 的精度直接影响电池电量计算准确性
- USB-C 连接器的 CC 引脚配置需确认（此网表中未显式出现 CC 网络，可能在 U6 内部处理）
- 所有去耦电容应尽量靠近 IC 电源引脚放置

## 相关笔记

- [[hardware-design/usb-c-power-delivery]] - USB PD 协议与电源路径管理
- [[hardware-design/bq27421-battery-gauge]] - TI BQ27421 电量计配置指南
- [[hardware-design/dc-dc-buck-converter]] - Buck 转换器设计与选型
- [[hardware-design/pcb-layout-best-practices]] - PCB 布局布线最佳实践
- [[hardware-design/i2c-protocol]] - I2C 通信协议详解
- [[hardware-design/lithium-battery-charging]] - 锂电池充电管理策略

## 项目文件结构

```
/home/lijian/project/pcb/
├── .claude/
│   └── settings.local.json        # Claude 项目配置
└── Netlist_Schematic1_2026-04-15.tel  # KiCad 网表文件
```

## 相关笔记

- [[power]] — 模块化可拆卸摄像头系统（同为电源 PCB 设计）
- [[frdm-mcxa346]] — FRDM-MCXA346 开发板设计文件
- [[bpi-m4-mechanical]] — BPI-M4 PCB DXF 文件
- [[AI-EDA]] — AI-EDA 智能 EDA 工具
- [[hardware-config]] — Jailhouse H3 嵌入式虚拟化配置
