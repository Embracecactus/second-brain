---
tags:
  - comsol
  - simulation
  - multiphysics
category: hardware-design
created: 2026-06-09
---

# COMSOL Batch 批处理仿真笔记

## 概述

本文档记录了基于 **COMSOL Multiphysics 6.3** (Build 290) 进行的一系列批处理(Batch)仿真实验。所有仿真于 2025年10月25日 在 Windows 系统 (`DESKTOP-0B20O0K`) 上以批处理模式运行，涉及**静电-固体力学**耦合的多物理场稳态求解。仿真通过参数扫描方式，分别研究了初始力 `F0` 和最大电压 `Vmax` 对模型行为的影响。

## 关键知识点

### 仿真环境

- **软件版本**: COMSOL Multiphysics 6.3 (开发版本 290)
- **运行模式**: 批处理模式 (Batch Mode)
- **硬件平台**: AMD64 Family 26 Model 68 Stepping 0, AuthenticAMD
- **核心数**: 16 核 (部分仿真使用 5 核)
- **可用内存**: 31.86 GB
- **操作系统**: Windows (WSL2 环境下访问)

### 物理场耦合

仿真涉及两个核心物理场的耦合求解:

1. **静电 (Electrostatics)** — 求解电势 `comp1.V`，极化变量 `comp1.es.Pmg`
2. **固体力学 (Solid Mechanics)** — 求解位移场 `comp1.u`

### 求解器配置

| 仿真类别 | 求解器类型 | 自由度数 | 内部自由度 | 矩阵类型 |
|----------|-----------|---------|-----------|---------|
| F0 系列 (基础模型) | 分离式求解器 (Separated) | 1068 | 1 | 对称矩阵 |
| F0 系列 (压电模型) | 非线性求解器 (Nonlinear) | 1740 | 1345 | 非对称矩阵 |
| Vmax 系列 | 非线性求解器 (Nonlinear) | 1740 | 1345 | 非对称矩阵 |

### 网格信息 (F0_0 基础模型)

- 几何形函数: 二次巧凑边点单元 (Quadratic Serendipity Elements)
- 顶点单元数: 8
- 边单元数: 44
- 边界单元数: 64
- 体单元数: 28
- 最小单元质量: 1 (完美质量)

## 技术细节

### 参数扫描方案

仿真分为两大参数扫描组:

#### 1. F0 (初始力) 参数扫描

| 模型文件 | F0 值 | 研究编号 | 求解时间 | 总时间 | 物理内存 |
|----------|-------|---------|---------|-------|---------|
| `batchmodel_F0_0.mph` | 0 | Study 1 / Sol 1 | 44 s | 63 s | 978 MB |
| `batchmodel_F0_25.mph` | 25 | Study 1 / Sol 1 | 44 s | 63 s | ~1 GB |
| `batchmodel_F0_50.mph` | 50 | Study 1 / Sol 1 | 44 s | 64 s | ~1 GB |
| `batchmodel_F0_-25000000.mph` | -25,000,000 | Study 2 / Sol 6 | 32 s | 49 s | 1.15 GB |
| `batchmodel_F0_-50000000.mph` | -50,000,000 | Study 2 / Sol 6 | 31 s | 51 s | 1.25 GB |

- F0 = 0, 25, 50 使用分离式求解器 (Separated Solver)，每步需 6 次迭代收敛
- F0 = -25M, -50M 使用非线性求解器，涉及压电耦合 (Piezoelectric Coupling)，连续参数范围 t = 0 到 3

#### 2. Vmax (最大电压) 参数扫描

| 模型文件 | Vmax 值 | 研究编号 | 求解时间 | 总时间 | 物理内存 |
|----------|--------|---------|---------|-------|---------|
| `batchmodel_Vmax_600.mph` | 600 | Study 1 / Sol 1 | 31 s | 48 s | 1.43 GB |
| `batchmodel_Vmax_1000.mph` | 1000 | Study 1 / Sol 1 | 33 s | 49 s | 1.43 GB |
| `batchmodel_Vmax_1600.mph` | 1600 | Study 1 / Sol 1 | 35 s | 50 s | 1.30 GB |

- 全部使用非线性求解器，连续参数范围 t = 0 到 3，步长 `CMPpcontstep = 0.005`
- 每个参数步通常 1 次迭代即可收敛，末端 (t 接近 3) 需 2 次迭代

### 收敛行为分析

**F0_0 基础模型** (分离式求解器):
- 每个连续参数步需要 6 次分离式迭代
- 静电场阻尼系数恒定为 0.8
- 固体力学阻尼系数恒定为 1.0
- 最终误差估计: 0.00023，残差估计: 0.043

**F0_-25M 压电模型** (非线性求解器):
- 大部分参数步仅需 1 次迭代
- 末端 (t = 2.995~3.0) 需 2 次迭代，误差估计突增至 0.0019
- 最终收敛至 SolEst = 0.51，ResEst = 1.9e-06

**Vmax_600 模型** (非线性求解器):
- 中间段收敛稳定，SolEst 维持在 1e-7 量级
- 末端 (t = 3.0) 出现跳跃: SolEst = 1.2e-06 -> 28
- 最终 ResEst = 1.5e-08

### 缩放变量 (Scaling Variables)

| 物理场 | 变量 | 缩放因子 |
|--------|------|---------|
| 极化 | `comp1.es.Pmg` | 3.4e-08 |
| 位移场 | `comp1.u` | 2.6e-11 |
| 电势 | `comp1.V` | 1 |

## 代码/配置片段

### COMSOL Batch 运行命令参考

```bash
# 典型的 COMSOL 批处理运行命令格式
comsol batch -inputfile batchmodel_Vmax_600.mph \
             -outputfile batchmodel_Vmax_600.mph \
             -batchlog batchmodel_Vmax_600.mph.log \
             -np 5
```

### 求解器日志关键字段说明

```
Iter      SolEst      ResEst     Damping    Stepsize #Res #Jac #Sol   LinErr   LinRes
```

| 字段 | 含义 |
|------|------|
| `Iter` | 迭代次数 |
| `SolEst` | 解估计误差 (Solution Estimate) |
| `ResEst` | 残差估计 (Residual Estimate) |
| `Damping` | 阻尼系数 (Damping Factor) |
| `Stepsize` | 步长大小 |
| `#Res` | 残差计算次数 |
| `#Jac` | Jacobian 矩阵组装次数 |
| `#Sol` | 线性系统求解次数 |
| `LinErr` | 线性求解误差 |
| `LinRes` | 线性求解残差 |

### 状态文件格式

```
1761396642450    # Unix 时间戳 (毫秒)
Done             # 状态: Done / Running / Failed
```

### 备份模型

- `backupbatchmodel.mph` (743 KB) — 基础模型备份，未含求解结果
- 其余模型文件约 60 MB (含结果数据) 或 ~1.3 MB (Vmax 系列)

## 文件清单

| 文件名 | 大小 | 说明 |
|--------|------|------|
| `backupbatchmodel.mph` | 743 KB | 基础模型备份 |
| `batchmodel_F0_0.mph` + 附属文件 | ~731 KB + log | F0 = 0 仿真 |
| `batchmodel_F0_25.mph` + 附属文件 | ~735 KB + log | F0 = 25 仿真 |
| `batchmodel_F0_50.mph` + 附属文件 | ~740 KB + log | F0 = 50 仿真 |
| `batchmodel_F0_-25000000.mph` + 附属文件 | ~60 MB + log | F0 = -25M 压电仿真 |
| `batchmodel_F0_-50000000.mph` + 附属文件 | ~60 MB + log | F0 = -50M 压电仿真 |
| `batchmodel_Vmax_600.mph` + 附属文件 | ~1.3 MB + log | Vmax = 600 仿真 |
| `batchmodel_Vmax_1000.mph` + 附属文件 | ~1.3 MB + log | Vmax = 1000 仿真 |
| `batchmodel_Vmax_1600.mph` + 附属文件 | ~1.3 MB + log | Vmax = 1600 仿真 |

每个 `.mph` 模型文件附带:
- `.mph.log` — 求解器日志
- `.mph.recovery` — 恢复文件 (0 字节，表示正常完成)
- `.mph.status` — 状态文件

## 相关链接

- COMSOL 官方文档: [https://www.comsol.com/documentation](https://www.comsol.com/documentation)
- COMSOL Batch Mode 指南: [https://www.comsol.com/documentation/](https://www.comsol.com/documentation/)
- 模型文件路径: `C:\Users\lijian\Documents\COMSOL\Batch\`
- WSL 访问路径: `/mnt/c/Users/lijian/Documents/COMSOL/Batch/`
