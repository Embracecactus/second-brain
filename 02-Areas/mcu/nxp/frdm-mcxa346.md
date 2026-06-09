---
tags:
  - nxp
  - frdm
  - mcxa346
category: mcu/nxp
created: 2026-06-09
board_pn: "170-95710 REV A"
schematic_pn: "SCH-95710_A"
pcb_pn: "LAY-95710_A"
---

# FRDM-MCXA346 开发板设计文件知识库

## 项目概述

FRDM-MCXA346 是 NXP Semiconductors 推出的基于 **MCXA346** 微控制器的 Freedom 开发板。MCXA346 属于 NXP MCX A 系列 MCU，搭载 **Arm Cortex-M33** 内核，主频可达 **180 MHz**，集成高达 **1 MB Flash** 和 **256 KB SRAM**，面向工业物联网、智能家居、电机控制、传感器中枢及电池供电设备等边缘应用场景。

### 设计文件清单

| 文件类型 | 文件名 | 说明 |
|---------|--------|------|
| 原理图 | `SCH-95710_A.DSN` / `.opj` | Cadence Allegro 原理图设计 |
| PCB | `LAY-95710_A.brd` | Cadence Allegro PCB 布局 |
| BOM | `SCH-95710_A.BOM` | 物料清单 (May 23, 2025) |
| Gerber | `GRB-95710_A.zip` | Gerber 制造文件包 |
| PDF | `SPF-95710_A.pdf` / `FAB-95710_A.pdf` / `DNP-95710_A.pdf` | 原理图/制造/DNP PDF |
| 测试点 | `IPCD356.txt` | IPC-D-356A 网表测试文件 |

### NXP 团队联系信息

| 角色 | 姓名 | 邮箱 |
|------|------|------|
| CAD Engineer | Aison Zhou | aison.zhou@nxp.com |
| CAD Manager | Ionut Manolescu | ionut.manolescu@nxp.com |
| Manufacturing Program Manager | Damon Hu | damon.hu@nxp.com |
| Product Engineer | Steven Ding | steven.ding_1@nxp.com |
| Design Engineer | Kate Fan | kate.fan@nxp.com |
| DFM Engineer | Rosalia Gonzalez | HWCOEPCBSolutions@nxp.com |

---

## 关键知识点

### MCU 核心参数 (MCXA346)

- **内核**: Arm Cortex-M33，支持 TrustZone 安全扩展
- **主频**: 最高 180 MHz
- **Flash**: 最高 1 MB
- **SRAM**: 最高 256 KB
- **低功耗**: 专为超低功耗边缘计算设计

### PCB 叠层结构

```
L1_TOP      — 1/2 OZ Copper (CONDUCTOR)
L2_GND      — 1 OZ Copper   (PLANE, GND 参考平面)
L3_POWER    — 1 OZ Copper   (PLANE, 电源平面)
L4_BOTTOM   — 1/2 OZ Copper (CONDUCTOR)
```

- **层数**: 4 层板
- **板厚**: 62.81 mil (约 1.6 mm)
- **设计规则状态**: UP TO DATE
- **单位**: mils
- **小数精度**: 3 位
- **IPC 标准**: IPC-D-356A
- **Gerber 生成日期**: Thu May 22 16:50:27 2025

### 电源域架构

| 电源网络 | 说明 |
|----------|------|
| `P5V0` | 5V 主电源 (USB/外部输入) |
| `P5V_USB` | USB 5V 电源 |
| `P5V_MCU_LINK` | MCU-Link 调试器 5V |
| `P5V_HDR_IN` | 排针 5V 输入 |
| `VDD_MCU` | MCU 核心/IO 电源 |
| `VDD_MCU_ANA` | MCU 模拟电源 |
| `VDD_BOARD` | 板级电源 |
| `VDD_MCU_LINK` | MCU-Link 调试器电源 |
| `P3V3` | 3.3V 电源轨 |
| `P5-9V_VIN` | 5-9V 宽范围输入 |
| `AGND` | 模拟地 |
| `GND` | 数字地 |

### 调试接口 (MCU-Link)

板载 **MCU-Link** 调试探针 (U2)，关键信号：

| 信号 | 说明 |
|------|------|
| `DBGIF_TMS_SWDIO` | SWD 数据线 |
| `DBGIF_TCK_SWCLK` | SWD 时钟线 |
| `DBGIF_TDI` | JTAG TDI |
| `DBGIF_TDO_SWO` | JTAG TDO / SWO 跟踪 |
| `DBGIF_RESET` | MCU 复位控制 |
| `DBGIF_DETECT` | 调试器检测 |
| `DBGIF_ISP0_CTRL` | ISP 模式控制 |
| `DBGIF_CAN_S` / `CAN_TXD` / `CAN_RXD` | CAN 总线桥接 |
| `MCU_LNK_TX` / `MCU_LNK_RX` | MCU-Link UART 通信 |
| `MCULINK_USB_DM` / `MCULINK_USB_DP` | MCU-Link USB 接口 |
| `USB_ACTIVE` / `VCOM_ACTIVE` | USB/VCOM 状态 LED |

### USB 接口

- **Full-Speed USB (USB0)**: 通过 J10 连接器，信号 `USB0_DP` / `USB0_DM`，含 VBUS 检测 (`USB0_VBUS_DET`)
- **MCU-Link USB**: 通过 J15 连接器，含 VBUS 供电
- 两个 USB 端口均配置 ESD 保护 (D2, D3, D13, D14) 和串联电阻

### GPIO / 外设引脚分配

MCXA346 引脚通过多个连接器 (J1, J2, J3, J5, J6, J8, J9) 引出：

**Port 0 (P0.x)**:
- `P0_0` ~ `P0_7`: 包含 ISP/UART 信号、外部中断、按钮 (SW3 连接 `P0_6`)
- `P0_12` ~ `P0_27`: 通过排针 J19 和 LED 矩阵 (DS1) 连接

**Port 1 (P1.x)**:
- `P1_0` ~ `P1_19`: 大量 GPIO 引出至连接器
- `P1_7` / `P1_29`: 用户按钮 SW2 / SW1
- `P1_2` ~ `P1_5`: 通过连接器引出
- `P1_8` / `P1_9`: 带串联电阻的 SPI/I2C 信号

**Port 2 (P2.x)**:
- `P2_0` ~ `P2_26`: 模拟/数字复用引脚
- `P2_12`: OPAMP0 输入 / USB VBUS 检测复用
- `P2_15` ~ `P2_20`: ADC 通道，含滤波电路 (C23-C30, R27-R40)

**Port 3 (P3.x)**:
- `P3_0` ~ `P3_31`: 大量 GPIO，含 SPI/UART/I2C 外设功能
- `P3_28` / `P3_27`: CAN 总线收发器连接

**Port 4 (P4.x)**:
- `P4_0` ~ `P4_7`: GPIO 引出

### 时钟系统

| 器件 | 型号/参数 | 说明 |
|------|-----------|------|
| Y1 | 230-78690 | 主晶振，通过 C20/C21 负载电容、R12/R13 匹配电阻 |
| Y2 | 230-78680 | RTC/副晶振，通过 C57/C58 负载电容 |

### 用户交互

| 元件 | 功能 |
|------|------|
| SW1 | 用户按钮 (连接 `P1_29`)，含去抖电容 C38 |
| SW2 | 用户按钮 (连接 `P1_7`)，含去抖电容 C40 |
| SW3 | ISP/复位按钮 (连接 `P0_6`) |
| DS1 | 16 段 LED 矩阵/指示灯 (P0_12~P0_27) |
| D5 | 状态 LED |
| D9, D10 | USB/VCOM 活动指示 LED |
| D11, D12 | BOOT_MODE / 其他状态 LED |

### 跳线配置

| 跳线 | 信号 | 说明 |
|------|------|------|
| JP1 | `VDD_MCU` | MCU 电源选择 |
| JP2 | `VDD_BOARD` | 板级电源选择 |
| JP3 | `VDD_MCU_ANA` | 模拟电源选择 |
| JP4 | `BOOT_MODE` | 启动模式配置 |
| JP5 | `HW_VER_6` | 硬件版本位 |
| JP6 | `HW_VER_7` | 硬件版本位 |
| JP7 | `HW_VER_9` | 硬件版本位 |
| SJ1~SJ6 | 各类信号 | 可焊接跳线，用于信号路由 |

### 硬件版本识别

板载多位硬件版本标识 (`HW_VER_2` ~ `HW_VER_9`)，通过电阻上拉/下拉 + 可选跳线实现版本编码，供软件读取。

### 电流检测

支持 3 相电机电流检测 (A/B/C 相)：
- `RSHUNT_CURA_P/N`, `RSHUNT_CURB_P/N`, `RSHUNT_CURC_P/N`
- 通过 J4 连接器引出，配合 SJ4/SJ5/SJ6 跳线选择

### 模拟前端 (OPAMP)

MCXA346 内置运放 (OPAMP0)，引脚通过 `P2_12/OPAMP0_INP` 引出，配合外部缓冲 (MC1_BUF_P) 实现模拟信号调理。

---

## 技术细节

### PCB 设计参数

从 IPCD356 文件提取的关键信息：

- **Etch Layers**: 4 层
- **Board Thickness**: 62.81 mil (1.595 mm)
- **Drawing Extents**: 约 545mm x 425mm (含制造边框)
- **设计工具**: Cadence Allegro
- **Output 格式**: IPC-D-356A

### Via 类型

| Via 名称 | 尺寸 (mil) | 用途 |
|----------|-----------|------|
| VIA14X8 | 40 x 40 | 标准过孔 |
| V24H12M28MX28 | 44 x 44 | 中等过孔 |
| V20H10M24MX24 | 40 x 40 | 小过孔 |
| V16H08T | 38 x 38 | 微过孔 |
| V18H08MX22 | 38 x 38 | 微过孔 |

### 关键连接器

| 编号 | 类型 | 用途 |
|------|------|------|
| J1, J3 | 210-81775 | Arduino 风格排母 (信号扩展) |
| J2, J9 | 210-81776 | Arduino 风格排母 (信号扩展) |
| J4 | 210-81774 | 电流检测/电源连接器 |
| J5, J6 | 210-81617 | 扩展连接器 |
| J8 | 210-83442 (DNP) | 高密度扩展连接器 |
| J10 | USB Type-C (211-10168) | Full-Speed USB 接口 |
| J14 | 210-80192 | MCU GPIO 连接器 |
| J15 | USB Type-C (211-10168) | MCU-Link USB 调试接口 |
| J16, J17 | 210-78429 | 扩展接口 |
| J18 | 排针 | 电源输入 |

### 主要 IC 器件

| 编号 | 器件 | 说明 |
|------|------|------|
| U1 | MCXA346 (344-10082) | 主 MCU，144 引脚封装 |
| U2 | MCU-Link (344-02543) | 板载调试探针 |
| U3 | 电源管理 (315-80827) | 电压调节器 |
| U5, U6 | 信号缓冲/电平转换 (312-83230) | USB/信号缓冲 |
| U7, U8 | USB ESD 保护 (480-78843, DNP) | USB 端口保护 |

---

## 代码/配置片段

### 关键信号命名映射 (来自 IPCD356)

```
P  NNAMEm0000  P2_12/OPAMP0_INP-MC1_BUF_P
P  NNAMEm0001  P2_12/USB0_VBUS_DET-FS_USB
P  NNAMEm0002  DBGIF_ISP0_CTRL
P  NNAMEm0003  DBGIF_TMS_SWDIO
P  NNAMEm0004  DBGIF_TCK_SWCLK
P  NNAMEm0005  MCULINK_USB_VBUS
P  NNAMEm0006  MCULINK_MCU_USB_VBUS
```

### Padstack 摘要

```
VIA14X8              TOP→BOTTOM   40x40 mil
S065-040T_TOL2_2     TOP→BOTTOM   85x85 mil
C065-040T_TOL2_2     TOP→BOTTOM   85x85 mil
O035X058-020X043T    TOP→BOTTOM   55x78 mil
R012X028S            TOP→TOP      32x48 mil  (0201 电阻)
C040SXP              TOP→TOP      60x60 mil  (0402 电容)
S040S                TOP→TOP      60x60 mil  (0402 封装)
SH_8MM_SQ_CIR_TENTED TOP→TOP     335x335 mil (安装孔)
```

### BOM 关键器件统计

- **总行项目**: 84 项
- **DNP (不贴装)**: 约 30+ 项
- **电容**: 56 个参考设计 (C1~C58)
- **电阻**: 88 个参考设计 (R1~R88)
- **LED/二极管**: 14 个 (D1~D14)
- **连接器**: 19 个 (J1~J19)
- **跳线**: 7 个 (JP1~JP7)
- **可焊接跳线**: 6 个 (SJ1~SJ6)
- **测试点**: 26 个 (TP1~TP28)

---

## 相关链接

- [NXP FRDM-MCXA346 官方页面](https://www.nxp.com/products/processors-and-microcontrollers/arm-microcontrollers/general-purpose-mcus/frdm-mcxa346)
- [NXP MCX A 系列 MCU](https://www.nxp.com/products/processors-and-microcontrollers/arm-microcontrollers/general-purpose-mcus/mcx-a-series-microcontrollers)
- [MCUXpresso IDE & SDK](https://www.nxp.com/design/design-center/software/development-software/mcuxpresso-software-and-tools-/mcuxpresso-integrated-development-environment-ide:MCUXpresso-IDE)
- [NXP MCU-Link 调试器](https://www.nxp.com/design/design-center/software/development-software/mcuxpresso-software-and-tools-/mcu-link-debug-probe:MCU-LINK)
- 设计文件本地路径: `/mnt/c/Users/lijian/Downloads/FRDM-MCXA346-DESIGN-FILES/`
- 工作区路径: `/mnt/c/Users/lijian/workspace/mcu/nxp/frdm-mcxa346/`

---

> **注意**: BOM 中标注 DNP 的器件为可选贴装，用于功能扩展或调试用途。实际量产配置需根据应用场景调整。

## 相关笔记

- [[zephyr-nxp-notes]] — NXP Zephyr 开发笔记
- [[zephyr]] — Zephyr RTOS 项目笔记
- [[pcb]] — PCB 电源管理与电池充电电路设计
- [[hardware-design]] — 硬件设计笔记索引
