---
title: diaoke - CNC 五轴 G 代码雕刻预览器
tags:
  - cnc
  - cam
  - gcode
  - five-axis
  - three.js
  - webgl
  - visualization
category: cnc-cam
created: 2026-06-09
status: active
project_path: /home/lijian/project/diaoke
---

# diaoke - CNC 五轴 G 代码雕刻预览器

## 项目概述

一个基于浏览器的 CNC 五轴 G 代码（NGC/TAP 格式）三维轨迹可视化工具，用于预览和分析"西游四人"等雕刻工件的加工路径。项目以纯前端 HTML + JavaScript 实现，配合 Three.js 渲染引擎进行点云可视化，同时提供 Node.js 命令行工具用于批量解析和诊断 G 代码数据。

## 技术栈

- **前端渲染**: Three.js (r128) + WebGL，使用 OrbitControls 实现交互式三维查看
- **G 代码解析**: 自研 JavaScript 正则解析器，支持五轴 (X/Y/Z/A/C) 坐标提取
- **后端服务**: Python 3 内置 HTTP 服务器 (http.server)，添加 CORS 头支持本地文件加载
- **数据格式**: NGC、TAP（LinuxCNC 五轴后处理输出）
- **加工类型**: 五轴联动加工、3+2 轴固定角度加工
- **CAD/CAM 来源**: STP 模型经 CAM 后处理生成（从 G 代码注释可见 Part_Name 为 .prt 文件）

## 项目结构

```
diaoke/
├── 西游四人_精stp_v2.ngc            # 精加工 G 代码（主要预览文件）
├── 西游四人-粗-20260305.ngc         # 粗加工 G 代码
├── 西游四人-粗-20260305-1.ngc       # 粗加工 G 代码（变体）
├── 西游四人_精2026305stp_去除底部.tap # 精加工（去除底部路径版本）
├── 终极简化版.html                   # 三轴简化预览器（推荐入门）
├── 增强版五轴预览器.html             # 增强版五轴预览器（含动画播放）
├── 唐僧查看器_修复版.html            # 唐僧单体查看器
├── 简单版唐僧查看器.html             # 简化版唐僧查看器
├── 唐僧五轴查看器.html               # 五轴唐僧查看器
├── 表面显示版预览器.html             # 表面渲染版预览器
├── 自定义G代码轨迹预览器.html        # 自定义 G 代码轨迹预览器
├── test_five_axis.html               # 五轴测试页面
├── test.html                         # 通用测试页面
├── 无文件依赖版.html                 # 无外部文件依赖版本
├── test_parser.js                    # 五轴 G 代码解析器（批量测试）
├── test_specific.js                  # 特定文件解析测试
├── diagnose.js                       # 文件诊断工具
├── start_server.py                   # CORS HTTP 服务器
└── .claude/settings.local.json       # Claude Code 配置
```

## 关键架构与设计决策

### 1. G 代码五轴解析器

核心解析逻辑基于正则表达式逐行提取坐标值，维护增量状态机：

```javascript
// 五轴坐标提取正则 - 支持 X/Y/Z/A/C/I/J/K
const coordRegex = /([XYZACIJ])(-?\d*\.?\d+)/gi;

// G 代码指令识别
const rapidCommands = ['G0', 'G00'];
const cutCommands = ['G1', 'G01'];
const moveCommands = [...rapidCommands, ...cutCommands, 'G2', 'G3', 'G02', 'G03'];
```

解析流程: 逐行读取 -> 剥离注释 (`;` 后内容) -> 识别 G 指令类型 -> 提取增量坐标 -> 去重 -> 输出点序列。

### 2. 五轴加工类型检测

通过分析 A 轴和 C 轴的变化范围自动判断加工模式：

```javascript
// 真正五轴联动: A 轴和 C 轴同时变化
// 3+2 轴加工: A/C 轴有值但变化范围小（<=1度）
// 三轴加工: 仅使用 XYZ
if (hasA || hasC) {
    if (aRange > 1 || cRange > 1) {
        axisType = '五轴联动加工';
    } else {
        axisType = '3+2轴固定角度加工';
    }
}
```

### 3. Three.js 点云可视化

使用 BufferGeometry + PointsMaterial 高效渲染大量轨迹点，按 Z 轴高度着色（蓝到红渐变）：

```javascript
// 高度着色算法
const t = (p.z - minZ) / (maxZ - minZ || 1);
colors[i * 3] = t;           // R: 低处红
colors[i * 3 + 1] = 0.3;     // G: 固定
colors[i * 3 + 2] = 1 - t;   // B: 高处蓝
```

### 4. 大文件处理策略

- 正则解析使用 `while(match = regex.exec(line))` 避免 `matchAll` 导致的栈溢出
- 手动循环计算 min/max 替代 `Math.min(...array)` 避免大数组展开溢出
- 逐行增量去重，避免全量数组比较
- 点云采样渲染（默认 50000 点），通过步长抽稀保证浏览器性能

## G 代码文件格式特征

从 NGC 文件头部注释可提取关键加工信息：

```
(Creation_Date: 2025-12-18 11:45)
(Cut_time: 12.93min)
(MCS_all: G54)
(Min_X=-20.41  Min_Y=-22.84  Min_Z=-6.32)
(Max_X=20.41   Max_Y=22.84   Max_Z=56.31)
(T2=R0.5  D=1.00 R=0.50)   -- 球头铣刀 R0.5mm
```

- 工件坐标系: G54
- 工件尺寸: 约 40.82mm x 45.68mm x 62.63mm
- 使用 R0.5 球头铣刀精加工
- 加工时间约 12.93 分钟

## 关键经验与洞察

1. **正则表达式全局匹配陷阱**: 使用带 `g` 标志的正则时必须手动重置 `lastIndex`，否则在循环复用正则对象时会跳过匹配
2. **大数组展开溢出**: `Math.min(...largeArray)` 在超过约 65000 个元素时会触发 Maximum call stack size exceeded，必须用循环替代
3. **五轴 G 代码增量模式**: 大多数 CAM 后处理输出为绝对坐标模式 (G90)，但部分行可能省略未变化的轴值，解析器需维护上一状态
4. **CORS 本地开发**: 浏览器直接打开 HTML 文件无法通过 `fetch()` 加载本地 NGC 文件，必须通过 HTTP 服务器提供服务
5. **点云渲染性能**: 超过 10 万个点时 Three.js Points 渲染明显卡顿，采样抽稀是必要手段

## 运行方法

### 启动本地服务器

```bash
cd /home/lijian/project/diaoke
python3 start_server.py
# 访问 http://localhost:8080/终极简化版.html
```

### 命令行解析 G 代码

```bash
cd /home/lijian/project/diaoke
node test_parser.js          # 批量测试所有文件
node test_specific.js        # 测试精加工文件
node diagnose.js             # 诊断特定文件
```

### 操作说明（浏览器预览器）

- 鼠标左键拖拽: 旋转视角
- 鼠标滚轮: 缩放
- 鼠标右键拖拽: 平移

## 相关概念

- [[G代码]] - CNC 机床通用编程语言
- [[五轴加工]] - 含 A/C 旋转轴的 CNC 加工方式
- [[Three.js]] - WebGL 三维渲染库
- [[CAM后处理]] - 将刀路转换为机床可执行代码的过程
- [[LinuxCNC]] - 开源 CNC 控制系统（本项目的 G 代码格式基于 LinuxCNC 5X 后处理）
- [[点云可视化]] - 大量三维点数据的渲染技术
