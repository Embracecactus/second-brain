---
title: LinuxCNC CAM - 五轴数控 CAM 软件
category: app/qt
tags:
  - qt6
  - cam
  - cnc
  - opengl
  - 5-axis
  - cad-cam
  - linuxcnc
  - manufacturing
  - c-plus-plus
status: alpha
version: 1.0.0-alpha
created: 2026-06-09
updated: 2025-03-24
project_path: /mnt/c/Users/lijian/workspace/qt/linuxcnc
---

# LinuxCNC CAM - 五轴数控 CAM 软件

## 项目概述

基于 Qt 6 开发的五轴数控 CAM（计算机辅助制造）软件，支持 CAD 模型导入、多种加工策略刀路生成、实时仿真以及 LinuxCNC G 代码后处理输出。项目跨平台支持 Windows 和 Linux，核心功能已完成，约 7000 行 C++ 代码。

## 技术栈

| 技术 | 版本/说明 |
|------|-----------|
| **GUI 框架** | Qt 6.8.3 (Widgets, OpenGL, Network, Concurrent, Xml) |
| **3D 渲染** | OpenGL 4.5 Core Profile (QOpenGLWidget) |
| **编程语言** | C++17 |
| **构建系统** | CMake 3.20+ |
| **编译器** | MSVC 2022 (Windows) / GCC 11+ (Linux) |
| **网络通信** | Qt TCP Socket (LinuxCNC 远程控制) |
| **几何计算** | 自研 geometry_utils / matrix_math |
| **可选依赖** | OpenCASCADE 7.6+ (STEP/IGES 格式支持，尚未集成) |

## 架构设计

### 分层架构

项目采用清晰的 **UI / Core / Network / Utils** 四层分离架构：

```
src/
├── main.cpp                          # 程序入口，OpenGL 格式配置
├── main_window.h/cpp                 # 主窗口，停靠窗口布局
├── widgets/                          # UI 层 - Qt Widget 组件
│   ├── viewer_3d.h/cpp               # OpenGL 3D 查看器 (QOpenGLWidget)
│   ├── tool_path_widget.h/cpp        # 刀路管理面板
│   ├── tool_library_widget.h/cpp     # 刀具库管理
│   ├── simulation_widget.h/cpp       # 仿真控制面板
│   └── cad_import_widget.h/cpp       # CAD 导入界面
├── core/                             # 业务逻辑层
│   ├── types.h/cpp                   # 核心数据类型定义
│   ├── cad_reader.h/cpp              # CAD 文件解析 (STL/OBJ)
│   ├── tool_path_generator.h/cpp     # 3 轴刀路策略
│   ├── tool_path_generator_5axis.h/cpp # 5 轴刀路策略
│   ├── post_processor.h/cpp          # G 代码后处理
│   ├── tool_manager.h/cpp            # 刀具库管理
│   └── simulation_engine.h/cpp       # 仿真引擎
├── network/                          # 通信层
│   ├── linuxcnc_client.h/cpp         # TCP 客户端
│   └── linuxcnc_protocol.h/cpp       # 通信协议
└── utils/                            # 工具层
    ├── geometry_utils.h/cpp          # 几何计算
    └── matrix_math.h/cpp             # 矩阵/运动学运算
```

### 关键设计决策

1. **五轴运动学抽象**：通过 `MachineType` 枚举（TableTable / HeadHead / TableHead / TRTRT）抽象不同机床构型，后处理中通过 `applyKinematicTransform` 统一调度对应的正/逆运动学变换
2. **后处理器多态**：`PostProcessor` 为抽象基类，`PostProcessorLinuxCNC` 实现具体后处理逻辑，便于扩展 Heidenhain、Siemens 等格式
3. **OpenGL 现代管线**：使用 VAO/VBO + Shader Program 架构，支持 Basic / Toolpath / Grid 三种着色器程序
4. **异步 CAD 加载**：CAD 文件读取支持异步加载和进度反馈，避免 UI 阻塞
5. **Qt Fusion 风格**：统一跨平台外观，OpenGL 4.5 Core Profile + 4x MSAA 抗锯齿

## 核心数据类型

```cpp
// 刀具位置 - 6 轴 (X/Y/Z + A/B/C 旋转)
struct ToolPosition {
    double x, y, z;      // 线性轴
    double a, b, c;       // 旋转轴 (度)
    ToolPosition interpolatedTo(const ToolPosition& target, double t) const;
    double distanceTo(const ToolPosition& other) const;
};

// 五轴机床配置
struct MachineConfig {
    MachineType type;          // TableTable / HeadHead / TableHead
    double xTravel, yTravel, zTravel;  // 行程 (mm)
    double aAxisMin, aAxisMax;         // A 轴范围 (度)
    double bAxisMin, bAxisMax;         // B 轴范围 (度)
    Point3D rotaryCenter;              // 旋转中心
};

// 加工操作
struct Operation {
    OperationType type;  // Roughing / Finishing / FiveAxisSwarf / ...
    int toolId;
    MachiningParameters params;  // 进给、转速、切深、行距等
};
```

## 加工策略

### 3 轴策略
- 平行精加工（Parallel Finishing）
- Z 字形粗加工（Zigzag Roughing）
- 轮廓加工（Contour）、型腔加工（Pocket）、钻孔（Drill）

### 5 轴策略
- **5 轴侧铣** (Swarf) - 刀具侧刃加工直纹面
- **5 轴索引** (Indexed) - 3+2 定位加工
- **5 轴同时** (Simultaneous) - 五轴联动
- 叶片加工（Blade）、流道加工（Port）

## LinuxCNC 集成

通过 TCP Socket（默认端口 5007）与 LinuxCNC 通信，支持：
- 紧急停止、原点回归
- 程序加载/运行
- 手动数据输入（MDI）、模式切换、点动控制
- 实时状态轮询

通信协议采用 JSON 格式：
```json
{
  "type": "command",
  "command": "run_program",
  "params": {"line": 0}
}
```

## 构建与运行

### Windows (Qt Creator 推荐)

```cmd
# 1. Qt Creator 打开 CMakeLists.txt
# 2. 选择套件: Desktop Qt 6.8.3 MSVC2022 64bit
# 3. 构建并运行
```

### Windows (命令行)

```cmd
cd C:\Users\lijian\workspace\qt\linuxcnc
mkdir build && cd build
cmake .. -G "Ninja" -DCMAKE_BUILD_TYPE=Release
ninja
Release\LinuxCNC_Cam.exe
```

### Linux

```bash
sudo apt install qt6-base-dev qt6-opengl-dev libgl1-mesa-dev cmake build-essential
cd /mnt/c/Users/lijian/workspace/qt/linuxcnc
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
./LinuxCNC_Cam
```

### 快速脚本

- `build.sh` - Linux 一键编译脚本
- `build.bat` - Windows 一键编译脚本
- `build_qtcreator.sh` - Qt Creator 专用

## 项目统计

| 类型 | 文件数 | 代码行数 |
|------|--------|----------|
| C++ 源文件 (.cpp) | 18 | ~4,500 |
| C++ 头文件 (.h) | 17 | ~2,500 |
| **总计** | **35** | **~7,000** |

## 已知限制与待办

- **STEP 格式**：需集成 OpenCASCADE，目前仅为框架
- **碰撞检测**：简化实现，需引入 BVH 加速
- **后处理器**：目前仅 LinuxCNC 格式，计划扩展 Heidenhain / Siemens
- **大规模模型**：百万级三角面性能需优化
- **测试**：单元测试和集成测试尚未编写

## 关键学习点

1. **Qt OpenGL 集成模式**：`QOpenGLWidget` + `QOpenGLFunctions` 继承模式是 Qt 中使用现代 OpenGL 的标准方式，需要在 `initializeGL()` 中初始化函数指针
2. **5 轴运动学**：不同机床构型（双转台/双摆头/摆头转台）的正逆运动学变换差异显著，需要在后处理层做统一抽象
3. **CAM 刀路策略设计**：策略模式适合管理多种加工算法，每种策略独立实现 `generate()` 接口
4. **跨平台 OpenGL**：OpenGL 4.5 Core Profile 在 Windows（MSVC）和 Linux（Mesa）上行为需注意差异，尤其是扩展和驱动兼容性
5. **CMake Qt6 集成**：`CMAKE_AUTOMOC` / `CMAKE_AUTORCC` / `CMAKE_AUTOUIC` 三件套是 Qt6 CMake 项目的标配，`find_package(Qt6 REQUIRED COMPONENTS ...)` 声明所需模块

## 相关链接

- [[Qt6 开发]]
- [[OpenGL 3D 渲染]]
- [[CNC 数控加工]]
- [[CMake 构建系统]]
- [[CAM 加工策略]]
- [LinuxCNC 官方项目](http://linuxcnc.org/)
- [Qt 6 文档](https://doc.qt.io/qt-6/)

## 相关笔记

- [[qt-linuxcnc]] — LinuxCNC CAM 五轴数控软件（同项目详细版）
- [[pocketcnc]] — PocketCNC Web 端五轴 CNC 仿真模拟器
- [[cmake-apt-setup]] — CMake APT 安装配置
- [[academic-papers]] — 学术论文（含 QTdesk 项目）
