---
tags: [cnc, carving, gcode, five-axis, three.js, webgl, visualization]
category: cnc-cam
created: 2026-06-09
updated: 2026-06-09
status: active
project_path: /home/lijian/project/diaoke
---

# diaoke - CNC 五轴 G 代码雕刻预览器

## 项目/工具概述

一个基于浏览器的 CNC 五轴 G 代码（NGC/TAP 格式）三维轨迹可视化工具，用于预览和分析"西游四人"等雕刻工件的加工路径。项目以纯前端 HTML + JavaScript 实现，配合 Three.js (r128) 渲染引擎进行点云/轨迹线/表面网格可视化，同时提供 Node.js 命令行工具用于批量解析、诊断和统计分析 G 代码数据。G 代码源自 LinuxCNC 5X_TT 后处理器输出，包含五轴联动（XYZ + AC 旋转轴）加工路径。

## 技术栈 / 关键特性

| 类别 | 技术 |
|------|------|
| 前端渲染 | Three.js r128 + WebGL，OrbitControls 交互式三维查看 |
| G 代码解析 | 自研 JS 正则解析器，支持五轴 X/Y/Z/A/C 坐标提取 |
| 后端服务 | Python 3 `http.server`，自定义 CORS 头支持本地文件加载 |
| 数据格式 | NGC、TAP（LinuxCNC 五轴后处理输出） |
| 加工类型 | 五轴联动加工、3+2 轴固定角度加工、三轴加工 |
| CAD/CAM 来源 | STP 模型经 CAM 后处理生成（.prt 文件） |

关键特性：
- 多种显示模式：轨迹线、点云、表面网格、组合显示
- 加工动画回放：刀具沿路径渐进式运动，带速度控制滑块
- 五轴刀具方向实时显示（A/C 轴四元数旋转）
- 高度着色（蓝-红渐变）+ 切削/快速移动颜色区分
- 大文件优化：采样抽稀、增量解析、避免栈溢出
- 支持拖拽文件加载和预设文件加载两种模式
- 预设视角切换：顶视图、前视图、侧视图、等轴视图

## 架构与设计

### 项目结构

```
diaoke/
├── 西游四人_精stp_v2.ngc              # 精加工 G 代码 (7.3MB, 184960 行)
├── 西游四人-粗-20260305.ngc           # 粗加工 G 代码 (1.4MB, 61406 行)
├── 西游四人-粗-20260305-1.ngc         # 粗加工变体 (554K, 25264 行)
├── 西游四人_精2026305stp_去除底部.tap  # 精加工去底版 (2.3MB, 56561 行)
├── 增强版五轴预览器.html              # 主力预览器 (902行) - 粗加工优化
├── 自定义G代码轨迹预览器.html        # 功能最全版 (891行)
├── 表面显示版预览器.html              # 表面网格渲染 (716行)
├── 唐僧五轴查看器.html                # 精加工专用 (340行)
├── 唐僧查看器_修复版.html             # 修复版 (328行)
├── 简单版唐僧查看器.html              # 简化版 (408行)
├── 终极简化版.html                    # 三轴入门版 (213行)
├── 无文件依赖版.html                  # 内置测试数据 (246行)
├── test_five_axis.html / test.html    # 测试页面
├── test_parser.js                     # 五轴解析器批量测试 (273行)
├── test_specific.js                   # 特定文件解析测试 (148行)
├── diagnose.js                        # 文件诊断工具 (91行)
├── start_server.py                    # CORS HTTP 服务器 (38行)
└── .claude/settings.local.json        # Claude Code 配置
```

### 查看器版本演进

| 版本 | 文件 | 特点 |
|------|------|------|
| 终极简化版 | `终极简化版.html` | 最简实现，三轴 XYZ 点云，采样 50000 点 |
| 无文件依赖版 | `无文件依赖版.html` | 内置半球测试数据，点云/表面切换 |
| 唐僧系列 | `唐僧五轴查看器.html` 等 | 针对精加工文件的专用查看器 |
| 表面显示版 | `表面显示版预览器.html` | 四种显示模式（轨迹/点云/表面/组合）+ 动画 |
| 增强版 | `增强版五轴预览器.html` | 粗加工优化：加粗轨迹、大刀具、增强光照、视角控制 |
| 自定义版 | `自定义G代码轨迹预览器.html` | 功能最全的轨迹预览器 |

## 核心知识点

### 1. G 代码五轴解析器设计

解析器采用增量状态机模式，逐行维护坐标状态：

```javascript
// 五轴坐标提取正则 - 支持 X/Y/Z/A/C/I/J/K
const coordRegex = /([XYZACIJ])(-?\d*\.?\d+)/gi;

// G 指令类型识别
const rapidCommands = ['G0', 'G00'];    // 快速移动
const cutCommands = ['G1', 'G01'];      // 切削进给
const moveCommands = [...rapidCommands, ...cutCommands, 'G2', 'G3', 'G02', 'G03'];
```

解析流程: 逐行读取 -> 剥离注释（`;` 后内容）-> 识别 G 指令类型 -> 正则提取增量坐标 -> 增量去重 -> 输出点序列 `{x, y, z, a, c, type}`。

### 2. 五轴加工类型自动检测

通过分析 A 轴和 C 轴变化范围自动判断加工模式：

```javascript
if (hasA || hasC) {
    if (aRange > 1 || cRange > 1) {
        axisType = '五轴联动加工';     // A/C 轴变化 > 1 度
    } else {
        axisType = '3+2轴固定角度加工'; // A/C 轴有值但变化 <= 1 度
    }
}
```

进一步通过检测 A 轴和 C 轴是否同时变化来区分真正五轴联动：

```javascript
const aChanged = Math.abs(curr.a - prev.a) > 0.01;
const cChanged = Math.abs(curr.c - prev.c) > 0.01;
if (aChanged && cChanged) simultaneousChanges++; // 真正五轴联动
```

### 3. 五轴刀具方向四元数旋转

使用 A/C 轴角度实时计算刀具朝向：

```javascript
function updateToolOrientation(a, c) {
    const aRad = (a || 0) * Math.PI / 180;
    const cRad = (c || 0) * Math.PI / 180;
    const quaternion = new THREE.Quaternion();
    const aQuat = new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(1, 0, 0), aRad);
    const cQuat = new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(0, 0, 1), cRad);
    quaternion.multiply(cQuat).multiply(aQuat);
    toolCone.setRotationFromQuaternion(quaternion);
}
```

### 4. 大文件处理与性能优化

- **正则 `lastIndex` 陷阱**: 带 `g` 标志的正则在循环复用时必须手动重置 `lastIndex = 0`，否则跳过匹配
- **大数组展开溢出**: `Math.min(...largeArray)` 超过约 65000 元素时触发 Maximum call stack size exceeded，必须用手动循环替代
- **点云采样**: 超过 10 万个点时 Three.js Points 渲染明显卡顿，通过步长抽稀（默认 50000 点）保证浏览器性能
- **coordRegex 防护**: `test_specific.js` 中加入 `coordCount > 100` 安全阀，防止异常行导致无限循环
- **增量去重**: 逐点比较前一个唯一点，避免全量数组的内存开销

### 5. 表面网格生成算法

`表面显示版预览器.html` 实现了简化的分层三角化：

```javascript
// 按 Z 高度分层（每 2mm 一层）
const zLayer = Math.round(p.z / 2) * 2;
// 每层按极角排序
const sorted = layerPoints.sort((a, b) => {
    return Math.atan2(a.y, a.x) - Math.atan2(b.y, b.x);
});
// 相邻层之间创建三角形连接
indices.push(prevLayerJ, vertexOffset + j, vertexOffset + j + 1);
```

## 关键代码/配置片段

### G 代码文件头部格式（LinuxCNC 5X_TT 后处理）

```
(no_group_name   LinuxCNC_5X_TT)
(Creation_Date: 2025-12-18 11:45)
(Part_Name: 西游四人_stp.prt)
(Cut_time: 12.93min)
(MCS_all: G54)
(Min_X=-20.41  Min_Y=-22.84  Min_Z=-6.32)
(Max_X=20.41   Max_Y=22.84   Max_Z=56.31)
(T2=R0.5  H2  D=1.00 R=0.50  75.0  D-  -6.32  50.44)
```

- 工件坐标系: G54
- 工件尺寸: 约 40.82mm x 45.68mm x 62.63mm
- 使用 R0.5 球头铣刀精加工
- 加工时间约 12.93 分钟
- A 轴范围: 0~50.44 度

### CORS HTTP 服务器（start_server.py）

```python
class CORSRequestHandler(SimpleHTTPRequestHandler):
    def end_headers(self):
        self.send_header('Access-Control-Allow-Origin', '*')
        self.send_header('Access-Control-Allow-Methods', 'GET, POST, OPTIONS')
        self.send_header('Access-Control-Allow-Headers', 'Content-Type')
        super().end_headers()
```

### Three.js 点云高度着色

```javascript
const t = (p.z - minZ) / (maxZ - minZ || 1);
colors[i * 3] = t;           // R: 高处偏红
colors[i * 3 + 1] = 0.3;     // G: 固定
colors[i * 3 + 2] = 1 - t;   // B: 低处偏蓝
```

## 使用方法 / 构建步骤

### 启动本地服务器

```bash
cd /home/lijian/project/diaoke
python3 start_server.py
# 访问 http://localhost:8080/终极简化版.html
# 或 http://localhost:8080/增强版五轴预览器.html
```

### 命令行解析 G 代码

```bash
cd /home/lijian/project/diaoke
node test_parser.js          # 批量测试所有文件，输出五轴统计
node test_specific.js        # 测试精加工文件，输出详细点数和五轴数据
node diagnose.js             # 诊断特定文件，逐步排查解析问题
```

### 浏览器预览器操作

- 鼠标左键拖拽: 旋转视角
- 鼠标滚轮: 缩放
- 鼠标右键拖拽: 平移
- 底部控制栏: 播放/暂停/重置/居中 + 速度滑块
- 右侧按钮: 切换显示模式（轨迹线/点云/表面网格/组合）
- 视角按钮: 顶视图/前视图/侧视图/等轴视图
- 文件加载: 点击"选择文件"或直接拖拽 NGC/TAP/GCODE 文件到页面

### 快捷符号链接（方便开发）

```bash
ln -sf "西游四人_精stp_v2.ngc" "tangseng_finish.ngc"
ln -sf "西游四人-粗-20260305.ngc" "tangseng_rough.ngc"
```

## 相关笔记

- [[3dgui]] -- 终端 3D G-Code 可视化渲染器
- [[pocketcnc]] -- PocketCNC Web 端五轴 CNC 仿真模拟器
- [[qt-linuxcnc]] -- LinuxCNC CAM 五轴数控软件
- [[pocktcnc-win]] -- PocketCNC Windows 版
- [[diaoke-cnc-gcode-viewer]] -- 本项目的详细技术分析笔记
