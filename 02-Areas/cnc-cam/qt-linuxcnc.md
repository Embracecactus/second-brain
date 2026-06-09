---
tags: [cnc, 5-axis, qt, cam, opengl]
category: cnc-cam
created: 2026-06-09
---
Now I have all the information needed. Let me generate the Obsidian note.

---
tags:
  - cnc
  - cam
  - qt6
  - opengl
  - 5axis
  - linuxcnc
  - cad
  - cpp
category: cnc-cam
created: 2026-06-09
status: alpha
version: 1.0.0-alpha
---

# LinuxCNC CAM — 五轴数控 CAM 软件

## 项目概述

基于 Qt 6 开发的五轴数控 CAM（计算机辅助制造）软件，支持 CAD 模型导入、多策略刀路生成、实时加工仿真、G 代码后处理以及 LinuxCNC 远程控制，可运行于 Windows 和 Linux 平台。

## 技术栈

- **语言**: C++17
- **GUI 框架**: Qt 6.8.3 (Core, Widgets, Gui, OpenGL, OpenGLWidgets, Network, Concurrent, Xml)
- **3D 渲染**: OpenGL 4.5 (QOpenGLWidget, VBO/VAO)
- **构建系统**: CMake 3.20+，支持 Ninja / MSVC / GCC
- **编译器**: MSVC 2022 (Windows) / GCC 11+ (Linux)
- **网络通信**: Qt TCP Socket（与 LinuxCNC 通信，默认端口 5007）
- **可选依赖**: OpenCASCADE 7.6+（STEP/IGES 格式支持）
- **许可证**: MIT License

## 架构设计

项目采用四层分层架构，职责清晰：

### 用户界面层 (`src/widgets/`)
- **Viewer3D** — OpenGL 3D 视图，负责模型渲染、刀路可视化、相机控制、实时仿真显示
- **ToolPathWidget** — 刀路列表管理、参数编辑、G 代码预览
- **ToolLibraryWidget** — 刀具库管理（端铣刀、球头刀、钻头、V 型刀、面铣刀），支持 JSON 导入导出
- **SimulationWidget** — 仿真播放控制（播放/暂停/停止/步进）、进度和速度调节
- **CadImportWidget** — CAD 文件导入界面

### 业务逻辑层 (`src/core/`)
- **ToolPathGenerator** — 策略模式实现的刀路生成器，支持 3 轴（平行、Z 字形、轮廓、型腔、钻孔）和 5 轴（侧铣 Swarf、索引 Indexed、同时 Simultaneous、叶片、流道）策略
- **SimulationEngine** — 仿真引擎，逐步执行刀路、碰撞检测、状态管理
- **PostProcessor** — G 代码后处理，支持三种 5 轴运动学变换（Table-Table / Head-Head / Table-Head）
- **ToolManager** — 刀具参数管理、有效性校验
- **CadReader** — 工厂模式的 CAD 读取器（STL ASCII/Binary、OBJ、STEP 框架）

### 数据访问层 (`src/network/`)
- **LinuxCNCClient** — TCP Socket 通信客户端，支持紧急停止、原点回归、程序加载/运行、MDI、模式切换、点动控制、状态轮询
- **LinuxCNCProtocol** — JSON 格式的通信协议定义

### 工具库层 (`src/utils/`)
- **GeometryUtils** — 距离/角度/法线/包围盒计算、路径插值与平滑
- **MatrixMath** — 旋转/平移/缩放矩阵、LookAt、5 轴正向/逆向运动学、欧拉角转换

### 线程模型
- 主线程：UI 事件处理、OpenGL 渲染、信号槽连接
- 工作线程：CAD 文件加载、刀路生成、G 代码生成、碰撞检测（Qt Concurrent）
- 网络线程：LinuxCNC 通信与状态轮询

## 关键设计决策

### 1. 策略模式驱动的刀路生成
刀路生成器使用策略模式，每种加工策略继承 `ToolPathGenerator` 基类并通过工厂注册，便于扩展新策略：

```cpp
class ToolPathGenerator {
    virtual bool generate(
        const Model3D& model,
        const ToolpathParams& params,
        const Tool& tool,
        ToolPath& outputPath
    ) = 0;
};

// 扩展方式
ToolPathGeneratorFactory::registerStrategy(
    StrategyType::Custom,
    [](QObject* parent) { return new MyStrategy(parent); }
);
```

### 2. 五轴运动学实现
支持三种机床配置的正向/逆向运动学变换，以 Table-Table 为例：

```cpp
void transformTableTable(
    const ToolPosition& in, ToolPosition& out,
    const QVector3D& aCenter, const QVector3D& bCenter
) {
    QVector3D p(in.x, in.y, in.z);
    // B 轴旋转 (Y轴)
    QMatrix4x4 bRot = rotationY(in.b);
    p = bRot.map(p - bCenter) + bCenter;
    // A 轴旋转 (X轴)
    QMatrix4x4 aRot = rotationX(in.a);
    p = aRot.map(p - aCenter) + aCenter;
    out.x = p.x(); out.y = p.y(); out.z = p.z();
    out.a = in.a; out.b = in.b;
}
```

### 3. 核心数据类型设计
`types.h` 定义了完整的 CAM 数据模型：`ToolPosition`（6 轴坐标）、`Tool`（刀具定义）、`ToolPath`（刀路+G 代码）、`Model3D`（三角网格模型）、`MachineConfig`（机床配置），所有类型使用 Qt 容器和向量类型以便与 OpenGL 集成。

### 4. OpenGL 渲染配置
启动时配置 OpenGL 4.5 Core Profile，启用 4x MSAA 抗锯齿，24 位深度缓冲，8 位模板缓冲。

## 代码统计

| 类型 | 文件数 | 代码行数（约） |
|------|--------|---------------|
| 头文件 (.h) | 23 | ~2,500 |
| 源文件 (.cpp) | 23 | ~4,500 |
| **总计** | **46** | **~7,000** |

## 构建与运行

### Linux
```bash
sudo apt install qt6-base-dev qt6-opengl-dev libgl1-mesa-dev cmake build-essential
cd /path/to/linuxcnc
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
./LinuxCNC_Cam
```

### Windows（Qt Creator 推荐）
1. 打开 Qt Creator → 文件 → 打开文件或项目 → 选择 `CMakeLists.txt`
2. 选择套件: Desktop Qt 6.8.3 MSVC2022 64bit
3. 构建并运行

### Windows（命令行）
```cmd
cd C:\Users\lijian\workspace\qt\linuxcnc
mkdir build && cd build
cmake .. -G "Ninja" -DCMAKE_BUILD_TYPE=Release
ninja
LinuxCNC_Cam.exe
```

## 工作流程

```
导入 CAD 模型 (STL/OBJ/STEP)
  → 配置刀具库
  → 创建加工操作
  → 生成刀路（选择策略和参数）
  → 仿真验证（碰撞检测）
  → 后处理生成 G 代码
  → 连接 LinuxCNC 发送程序
```

## 关键学习与洞察

1. **5 轴 CAM 的核心难点在于运动学变换** — 不同机床构型（Table-Table / Head-Head / Table-Head）需要不同的正向/逆向运动学计算，且需处理奇异点和轴限位
2. **策略模式是刀路生成的理想架构** — 不同加工策略的算法差异大但接口统一，策略模式使得新增加工方式无需修改核心框架
3. **Qt Concurrent 用于计算密集型任务** — 刀路生成和碰撞检测应避免阻塞 UI 线程
4. **OpenGL 4.5 Core Profile 是现代 3D 渲染的基线** — VBO/VAO 缓存几何数据、视锥体裁剪和 LOD 是大规模刀路可视化的必要优化
5. **STEP 格式需要 OpenCASCADE** — STL/OBJ 只包含网格数据，STEP 包含精确 B-Rep 几何，对高精度加工至关重要

## 已知限制

- STEP 格式支持仅为框架，需集成 OpenCASCADE
- 碰撞检测为简化实现，需 BVH 加速优化
- 大规模模型性能有待提升
- 后处理器目前仅支持 LinuxCNC 格式

## 项目路径

- 项目根目录: `/mnt/c/Users/lijian/workspace/qt/linuxcnc`
- 构建配置: `/mnt/c/Users/lijian/workspace/qt/linuxcnc/CMakeLists.txt`
- 源码入口: `/mnt/c/Users/lijian/workspace/qt/linuxcnc/src/main.cpp`
- 核心数据类型: `/mnt/c/Users/lijian/workspace/qt/linuxcnc/src/core/types.h`
- 架构文档: `/mnt/c/Users/lijian/workspace/qt/linuxcnc/docs/ARCHITECTURE.md`

## 相关概念

- [[CNC]] — 计算机数控基础
- [[CAM]] — 计算机辅助制造
- [[CAD]] — 计算机辅助设计
- [[OpenGL]] — 3D 图形渲染 API
- [[Qt6]] — 跨平台 GUI 框架
- [[LinuxCNC]] — 开源 CNC 控制器
- [[G-Code]] — 数控机床编程语言
- [[5-Axis Machining]] — 五轴加工技术
- [[Kinematics]] — 机器人/机床运动学
- [[Toolpath Generation]] — 刀路生成算法

## 相关笔记

- [[qt-projects]] — LinuxCNC CAM 五轴数控软件（Qt 项目集）
- [[pocketcnc]] — PocketCNC Web 端五轴 CNC 仿真模拟器
- [[3dgui]] — 终端 3D G-Code 可视化渲染器
- [[pocktcnc-win]] — PocketCNC Windows 版
- [[h618]] — H618 TV Box 定制 Linux 系统（LinuxCNC 应用）
