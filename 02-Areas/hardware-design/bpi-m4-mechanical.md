---
title: "BPI_M4B_V10 PCB DXF 工程文件解析"
aliases:
  - BPI M4B V10 DXF
  - BPI_M4B PCB Artwork
tags:
  - PCB
  - DXF
  - BananaPi
  - EDA
  - gerber-alternative
type: engineering-note
created: 2026-06-09
source_date: 2024-05-22
board_name: BPI_M4B_V10
format: AutoCAD DXF (AC1009/R12)
unit: mm
---

# BPI_M4B_V10 PCB DXF 工程文件解析

## 概述

本笔记记录 BananaPi M4B (V10 版本) 的 PCB 设计 DXF 文件结构与内容。DXF 文件来源于嘉立创 EDA (LCEDA) 导出，采用 AutoCAD R12 (AC1009) 格式，包含顶层 (Top) 和底层 (Bottom) 的丝印层 (Silkscreen) 及焊盘层 (Pad) 信息。文件日期标记为 2024-05-22，导出时间 2024-05-23。

---

## 关键信息速览

| 项目 | 值 |
|---|---|
| 板卡名称 | BananaPi M4B |
| 版本号 | V10 |
| DXF 格式 | AutoCAD R12 (AC1009) |
| 单位制 | mm (LUNITS=2) |
| 精度 | 0.01mm (LUPREC=2) |
| 板厚 (THICKNESS) | 1.602232 mm |
| 设计范围 X | -95.60 ~ 437.80 mm |
| 设计范围 Y | -131.33 ~ 302.97 mm |
| 线型 | CONTINUOUS (实线) |

---

## 文件清单

| 文件名 | 大小 | 行数 | 说明 |
|---|---|---|---|
| `BPI_M4B_V10-20240522-TOP-20240523.dxf` | 1.4 MB | 187,554 | 顶层 (Top Layer) |
| `BPI_M4B_V10-20240522-BOT-20240523.dxf` | 590 KB | 76,138 | 底层 (Bottom Layer) |

源路径: `/mnt/c/Users/lijian/Downloads/BPI_M4B_V10-DXF/`

---

## DXF 文件结构

每个 DXF 文件由以下标准 Section 组成:

```
HEADER    -- 全局变量 (版本、范围、单位等)
TABLES    -- 表定义 (线型、图层、文字样式)
BLOCKS    -- 块定义 (可复用的焊盘图形)
ENTITIES  -- 实体数据 (实际的几何图形)
```

### 图层 (Layer) 定义

#### TOP 文件 -- 5 个图层

| 图层名 | 颜色号 | 用途 |
|---|---|---|
| `BG_DESIGN_OUTLINE` | 7 (白/黑) | PCB 板框 (Board Outline) |
| `BG_SILKSCREEN_TOP` | 11 (青) | 板级顶层丝印 |
| `PG_SILKSCREEN_TOP_OUTLINE` | 31 | 封装级顶层丝印边框 |
| `PG_SILKSCREEN_TOP` | 21 | 封装级顶层丝印 |
| `PIN_TOP` | 12 | 顶层焊盘 |

#### BOT 文件 -- 4 个图层

| 图层名 | 颜色号 | 用途 |
|---|---|---|
| `BG_DESIGN_OUTLINE` | 7 (白/黑) | PCB 板框 (Board Outline) |
| `PG_SILKSCREEN_BOTTOM_OUTLINE` | 254 | 封装级底层丝印边框 |
| `PG_SILKSCREEN_BOTTOM` | 254 | 封装级底层丝印 |
| `PIN_BOTTOM` | 85 | 底层焊盘 |

> **命名规则说明**: `BG_` 前缀 = Board Global (板级全局), `PG_` 前缀 = Package Global (封装级), `PIN_` = 焊盘 (Pad)。

---

## 实体 (Entity) 统计

### TOP 层实体分布

| 实体类型 | 数量 | 说明 |
|---|---|---|
| POLYLINE | 3,638 | 多段线 (含闭合轮廓) |
| VERTEX | 8,635 | 顶点 (POLYLINE 的子实体) |
| SEQEND | 3,638 | 序列结束标记 |
| CIRCLE | 736 | 圆形 (焊盘/过孔) |

各图层实体归属:

| 图层 | 实体数 |
|---|---|
| BG_SILKSCREEN_TOP | ~8,502 |
| PIN_TOP | ~2,847 |
| PG_SILKSCREEN_TOP_OUTLINE | ~1,213 |
| PG_SILKSCREEN_TOP | ~416 |
| BG_DESIGN_OUTLINE | ~9 |

### BOT 层实体分布

| 实体类型 | 数量 | 说明 |
|---|---|---|
| POLYLINE | 1,167 | 多段线 |
| VERTEX | 4,005 | 顶点 |
| SEQEND | 1,167 | 序列结束标记 |
| CIRCLE | 101 | 圆形 |

各图层实体归属:

| 图层 | 实体数 |
|---|---|
| PIN_BOTTOM | ~2,790 |
| PG_SILKSCREEN_BOTTOM_OUTLINE | ~2,215 |
| PG_SILKSCREEN_BOTTOM | ~212 |
| BG_DESIGN_OUTLINE | ~9 |

---

## 焊盘形状 (Block Definitions)

DXF 定义了 8 种标准焊盘形状 (FIGURE block), 用于通过 INSERT 引用实例化:

| Block 名称 | 几何形状 | 说明 |
|---|---|---|
| `FIGURE_CIRCLE` | 圆形 (R=0.5mm) | 圆形焊盘/过孔 |
| `FIGURE_SQUARE` | 正方形 (1x1mm) | 方形焊盘 |
| `FIGURE_HEX` | 正六边形 | 六角焊盘 (螺母孔等) |
| `FIGURE_OCT` | 正八边形 | 八角焊盘 |
| `FIGURE_DIAMOND` | 菱形 | 菱形焊盘 (极性标记) |
| `FIGURE_CROSS` | 十字形 | 十字形焊盘 |
| `FIGURE_TRI` | 三角形 | 三角形焊盘 |
| `FIGURE_OBLONG` | 椭圆/长圆形 | 长圆形焊盘 (常见于 USB/连接器) |

所有 Block 定义的单位尺寸为 1mm 基准, 通过 INSERT 时的缩放因子 (Scale) 调整实际大小。

---

## 接口与标识信息

从顶层丝印文字 (TEXT entity, group code 1) 提取到的板卡接口/标识:

| 标识 | 含义 |
|---|---|
| `BPI_M4B` | 板卡名称标识 |
| `V10` | 版本号 |
| `USB` | USB 接口 (出现 2 处, 含 USB Host/OTG) |
| `OTG` | USB OTG 接口 |
| `HDMI` | HDMI 视频输出接口 |
| `GBE` | Gigabit Ethernet 千兆以太网 |
| `WIFI/BT` | WiFi / Bluetooth 无线模块 |
| `GPIO(40PIN)` | 40-Pin GPIO 扩展排针 |
| `GND RX TX` | UART 调试串口标记 |
| `PWR` | 电源接口/指示 |
| `RST` | 复位按钮 |
| `FEL` | FEL 模式按键 (Allwinner 芯片烧录模式) |
| `USER` | 用户按键 |
| `SYS` | 系统指示灯 |
| `LINER` | 线性稳压器标识 |
| `A`, `V`, `Y+`, `G+` | LED 指示灯/电源标识 |

---

## 技术细节

### DXF 版本兼容性

文件采用 AC1009 (AutoCAD Release 12) 格式, 这是 EDA 工具导出的最广泛兼容版本。支持实体类型:
- `POLYLINE` + `VERTEX` + `SEQEND` (闭合多段线)
- `CIRCLE` (圆形)
- `TEXT` (单行文字, 使用 STANDARD 样式, 字高 0.2mm)

未使用 `LWPOLYLINE`, `MTEXT`, `HATCH`, `SPLINE` 等高版本实体, 保证了最大兼容性。

### 坐标系统

- 原点 (INSBASE): (0, 0)
- X 轴范围: -95.60mm ~ +437.80mm (总宽约 533mm)
- Y 轴范围: -131.33mm ~ +302.97mm (总高约 434mm)
- 这些尺寸包含了设计扩展边界, 实际 PCB 尺寸远小于此

### 焊盘精度

焊盘坐标精度为 0.01mm (LUPREC=2), 典型焊盘尺寸:
- 最小间距: 0.045720mm (45.72um, 出现 11,312 次 -- 可能为 BGA 焊盘间距)
- 常见值: 0.099060mm, 0.127000mm, 0.149860mm, 0.152400mm

---

## 应用场景

1. **PCB 外壳/结构件设计** -- 导入 SolidWorks/Fusion 360 进行机械配合设计
2. **丝印层检查** -- 验证元件标注、接口标识的正确性
3. **焊盘分布分析** -- 了解 BGA/QFP 等封装的焊盘布局
4. **替代 Gerber 查看** -- 在无 Gerber Viewer 时用 AutoCAD/QCAD 快速查看 PCB 设计

---

## 相关工具

- **嘉立创 EDA** (LCEDA) -- DXF 源文件导出工具
- **QCAD / LibreCAD** -- 开源 DXF 查看/编辑工具
- **AutoCAD** -- 商业 DXF 原生支持
- **KiCad** -- 可通过插件导入 DXF 辅助 PCB 设计
