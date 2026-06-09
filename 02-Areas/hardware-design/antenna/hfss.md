---
tags: [hfss, antenna, wifi, simulation, electromagnetic, fea]
category: hardware-design/antenna
created: 2026-06-09
updated: 2026-06-09
status: active
---

# Ansys HFSS 天线仿真项目

## 项目/工具概述

Ansys HFSS (High Frequency Structure Simulator) 是业界领先的三维全波电磁场仿真软件，基于有限元法 (Finite Element Method, FEM) 求解 Maxwell 方程组，广泛应用于天线设计、射频/微波电路、信号完整性等领域。本工作区包含 **5 个 HFSS 子项目**，覆盖从基础偶极子天线到复杂有限阵列天线的设计与仿真，均使用 **Ansys ElectronicsDesktop 2025 R2** 版本，采用 HFSS Terminal Network 求解器进行 S 参数和远场方向图分析。

## 技术栈 / 关键特性

| 项目 | 说明 |
|------|------|
| **仿真工具** | Ansys ElectronicsDesktop 2025 R2 (HFSS) |
| **求解器** | HFSS Terminal Network / DrivenTerminal |
| **网格方法** | Auto (TAU Volume Mesh)，SliderMeshSettings=5 |
| **自适应收敛** | MaximumPasses=6, PercentRefinement=30, Delta=0.02 |
| **端口类型** | Lumped Port (Modal)，Impedance=50 ohm |
| **边界条件** | Perfect E, Radiation, Finite Conductivity (copper) |
| **基板材料** | FR4_epoxy (er=4.4, tan_d=0.02), copper (sigma=5.8e7 S/m) |
| **工作频率** | 1.4 GHz / 2.4 GHz / 2.45 GHz / 5.8 GHz |

## 架构与设计

```
hfss/
├── antaenna/              # 基础贴片天线 (使用 Ansys 3D Component 库)
│   ├── Project1.aedt
│   └── Project1.aedtresults/
├── Dipole/                # 偶极子天线仿真
│   ├── Project2.aedt
│   └── Project2.aedtresults/
├── esp32/                 # ESP32 WiFi 天线 (最完整的参数化设计)
│   ├── Project1.aedt
│   └── Project1.aedtresults/
├── WIFI_2P45_5P8G/        # 双频 WiFi 天线 (2.45GHz + 5.8GHz)
│   └── WIFI_2P45_5P8G/
│       ├── Project1.aedt
│       └── Project1.aedtresults/
└── Finite_Array_MultiUnitCell_Installed.aedt  # 4x4 有限阵列 + 天线罩
```

## 核心知识点

### 1. ESP32 天线项目 (esp32)

最完整的参数化微带天线设计，针对 ESP32 模组的 2.4 GHz WiFi 频段：

- **设计变量**：`w1=18mm`, `l1=25.5mm`, `h1=1.6mm` (FR4 基板厚度)
- **求解频率**：2.4 GHz，Frequency Sweep 覆盖目标带宽
- **基板**：FR4_epoxy, permittivity=4.4, loss_tangent=0.02
- **端口**：Lumped Port (Modal)，50 ohm 阻抗
- **边界**：Perfect E (PEC 地平面), Radiation (空气腔)
- **优化**：已执行参数化扫描 (opti0_0.profile)，h1 从 0.4mm 扫描到 1.6mm 共 10 步，l1=25.5mm, w1=18mm 保持不变，每步约 22 秒完成

### 2. 偶极子天线项目 (Dipole)

基础偶极子天线仿真，用于验证仿真流程和边界设置：

- **几何**：使用 Ansys 内置 3D Component 库 (`<syslib>/3DComponents/HFSS/Antennas/Dipole/Dipole_Antenna_DM.a3dcomp`)
- **偶极子参数**：dipole_length, port_gap (通过 3D Component 参数化)
- **求解频率**：1.4 GHz
- **端口**：Lumped Port (Modal)，50 ohm
- **边界条件**：Radiation 边界 (模拟自由空间)
- **材料**：PEC (perfect electric conductor, conductivity=1e30)
- **用途**：作为天线仿真的 baseline 验证

### 3. 基础贴片天线项目 (antaenna)

使用 Ansys 3D Component 的偶极子天线变体：

- **组件来源**：`<syslib>/3DComponents/HFSS/Antennas/Dipole/Dipole_Antenna_DM.a3dcomp`
- **特点**：UsesAdvancedFeatures=true，支持高级仿真功能
- **端口**：Lumped Port (Modal)
- **求解器**：HFSS Terminal Network

### 4. 双频 WiFi 天线项目 (WIFI_2P45_5P8G)

针对 2.45 GHz 和 5.8 GHz 双频段 WiFi 应用的天线设计，是最复杂的单天线项目：

- **设计变量**（13 个参数化变量）：
  - 天线尺寸：`X1=1mm`, `X2=3.8mm`, `X3=5.6mm`, `Y1=8mm`, `Y2=2mm`, `Y3=3mm`
  - 馈电位置：`XX1=2mm`, `YY1=5mm`
  - 微带线宽：`W=0.5mm`
  - 基板尺寸：`subX=30mm`, `subY=40mm`, `subH=1.6mm`
  - 波长参考：`lamda=122mm` (对应 2.45 GHz)
- **求解类型**：DrivenTerminal, MultiFrequency
- **自适应频率**：2.45 GHz + 5.8 GHz 双频自适应
- **频率扫描**：1.8 GHz ~ 6.5 GHz, 401 个采样点, Interpolating Sweep
- **材料**：FR4_epoxy 基板, copper 导体 (sigma=58 MS/m)
- **边界条件**：
  - Finite Conductivity (copper) x2 — 贴片和地平面
  - Radiation — 空气腔外表面
  - Lumped Port + Terminal (50 ohm) — 馈电端口
- **附件**：包含 Excel 工作表和 Visio 绘图 (设计文档)

### 5. 有限阵列天线项目 (Finite_Array_MultiUnitCell_Installed)

来源于 Ansys 教程 (`D:/Array_Lunch&Learn/`)，演示有限阵列 + 天线罩的完整仿真流程：

- **阵列配置**：4x4 patch 阵列，16 个端口 (numberofports=16)
  - 单元索引：Array[3,3] ~ Array[6,6] (共 16 个)
  - 单元间距：~5.08mm (0.2 inch) 基于 pin 坐标
- **多 Design 结构**：
  - `Unit_Cell_Design` — 单元胞仿真 (周期边界)
  - `00_Array2` / `Array` — 完整 4x4 阵列
  - `Installed` — 阵列 + 天线罩安装仿真
- **材料**：GOLD 导体 (用于阵列 patch), PML 吸收边界 (各向异性)
- **PML 设置**：
  - `PMLGroup1_Z_1`：各向异性 permittivity/permeability (Z 方向吸收)
  - `PMLGroup1_XZ_1`：XZ 平面吸收层
- **求解结果文件**：`.asol` (Adaptive Solution), `.dmesh` (mesh 数据), `.g3derr` (误差数据)

## 关键代码/配置片段

### ESP32 参数化变量定义

```
VariableProp('w1', 'UD', '', '18mm')
VariableProp('l1', 'UD', '', '25.5mm')
VariableProp('h1', 'UD', '', '1.6mm')
VariableOrders[3: 'w1', 'l1', 'h1']
```

### WiFi 双频天线设计变量

```
VariableProp('X1', 'UD', '', '1mm')
VariableProp('X2', 'UD', '', '3.8mm')
VariableProp('X3', 'UD', '', '5.6mm')
VariableProp('Y1', 'UD', '', '8mm')
VariableProp('Y2', 'UD', '', '2mm')
VariableProp('Y3', 'UD', '', '3mm')
VariableProp('XX1', 'UD', '', '2mm')
VariableProp('YY1', 'UD', '', '5mm')
VariableProp('W', 'UD', '', '0.5mm')
VariableProp('subX', 'UD', '', '30mm')
VariableProp('subY', 'UD', '', '40mm')
VariableProp('subH', 'UD', '', '1.6mm')
VariableProp('lamda', 'UD', '', '122mm')
```

### WiFi 双频自适应求解配置

```
SetupType='HfssDriven'
SolveType='MultiFrequency'
MultipleAdaptiveFreqsSetup:
  AdaptAt: Frequency='2.45GHz', Delta=0.02
  AdaptAt: Frequency='5.8GHz', Delta=0.02
MaximumPasses=6
PercentRefinement=30
Sweep: 1.8GHz ~ 6.5GHz, 401 points, Interpolating
```

### FR4 基板材料参数

```
permittivity='4.4'
dielectric_loss_tangent='0.02'
thermal_conductivity='0.294'
mass_density='1900'
youngs_modulus='11000000000'
poissons_ratio='0.28'
```

### ESP32 优化扫描记录 (opti0_0.profile)

```
Profile_0: h1=0.4mm, l1=25.5mm, w1=18mm → Finished (2025-10-23 23:15)
Profile_1: h1=1.6mm, l1=25.5mm, w1=18mm → Finished (2025-10-23 24:15)
每步 solve time ~22s, 总共 10 个 adaptive pass
```

## 使用方法 / 构建步骤

### 前置条件

1. 安装 Ansys ElectronicsDesktop 2025 R2 (或更新版本)
2. 确保 HFSS 许可证可用 (需要 HFSS 模块)

### 基本工作流

1. **打开项目**：在 ElectronicsDesktop 中打开对应 `.aedt` 文件
2. **检查变量**：在 Design Parameters 中确认变量值是否符合设计目标
3. **验证几何**：切换到 3D Modeler 视图，检查天线几何结构
4. **运行仿真**：
   - Analysis → Add Solution Setup → 确认频率和收敛参数
   - Analysis → Analyze (或右键 Setup → Analyze)
5. **查看结果**：
   - Results → Create Terminal Solution Data Report → S Parameter
   - Results → Create Far Fields Report → 远场方向图
6. **参数扫描**：Optimetrics → Add → Parametric 设置变量扫描范围

### ESP32 项目优化流程

```
1. 打开 esp32/Project1.aedt
2. Optimetrics → Parametric Setup 确认 h1 扫描范围
3. Analyze All → 等待完成 (约 4 分钟, 10 步)
4. Results → Post Process Profile 查看收敛历史
```

### 有限阵列项目流程

```
1. 打开 Finite_Array_MultiUnitCell_Installed.aedt
2. 先仿真 Unit_Cell_Design (周期边界条件)
3. 运行 Array design (16 端口全波仿真)
4. 运行 Installed design (含天线罩)
5. 对比 Array vs Installed 的 S 参数差异
```

## 常见问题与注意事项

- **PML 边界**：有限阵列项目使用各向异性 PML 吸收层，注意材料参数中负值的 loss_tangent 表示增益 (吸收电磁波)
- **网格收敛**：SliderMeshSettings=5 是中等精度，对精细结构可提高到 7-8
- **双频设计**：WIFI 项目使用 MultiFrequency 自适应，需同时在 2.45 GHz 和 5.8 GHz 收敛
- **3D Component**：Dipole 和 antaenna 项目依赖 Ansys 系统库路径 (`<syslib>/3DComponents/`)，在其他机器上打开需确认库路径一致
- **单位制**：所有 `.aedt` 文件使用 SI 单位 (米)，设计变量中使用 mm 便于阅读

## 相关笔记

- [[comsol]] — COMSOL 压电仿真项目 (多物理场仿真对比)
- [[comsol-batch]] — COMSOL 批量仿真笔记 (参数化扫描方法论)
- [[pcb]] — PCB 电源管理与电池充电电路设计 (天线馈电网络相关)
- [[fpga]] — FPGA 数字设计 (射频系统数字基带处理)
- [[hardware-config]] — Jailhouse H3 嵌入式虚拟化配置 (嵌入式系统集成)
