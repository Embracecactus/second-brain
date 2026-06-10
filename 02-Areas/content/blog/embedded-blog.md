---
title: "embedded-blog - 嵌入式技术博客"
tags:
  - blog
  - hugo
  - embedded
  - STM32
  - FreeRTOS
  - RV1126
  - WSL2
  - 硬件设计
  - RTOS
  - 嵌入式Linux
category: content/blog
created: 2026-06-09
status: active
project_path: /mnt/c/Users/lijian/workspace/codebuudy/embedded-blog
---

# embedded-blog - 嵌入式技术博客

## 项目概述

一个基于 Hugo 静态站点生成器构建的嵌入式技术博客，使用 hugo-theme-stack 主题，专注于嵌入式硬件设计、嵌入式软件开发、开发工具链和项目实战四个方向的技术分享。博客内容采用中文撰写，面向嵌入式系统开发者。

## 技术栈

### 站点构建

- **静态站点生成器**: Hugo（TOML 配置格式）
- **主题**: [hugo-theme-stack](https://github.com/CaiJimmy/hugo-theme-stack)（通过 git submodule 管理）
- **语言**: 中文 (zh-cn)
- **搜索引擎**: Fuse.js（客户端全文搜索）
- **部署目标**: Azure 云服务器
- **许可证**: CC BY-NC-SA 4.0

### 博客内容涉及的技术领域

- **MCU 平台**: STM32（GPIO、HAL 库）、ESP32
- **SoC 平台**: RV1126（Rockchip，4xCortex-A7 + RISC-V + NPU 2.0TOPS）、RK3568
- **FPGA**: Xilinx Zynq、Lattice iCE40
- **RTOS**: FreeRTOS（任务调度、优先级继承）、RT-Thread、Zephyr
- **嵌入式 Linux**: Buildroot、Yocto、设备驱动开发
- **编程语言**: C、C++、Rust、Python
- **PCB 设计**: KiCad、Altium Designer
- **开发工具**: VSCode、CLion、CMake、JTAG、GDB、Logic Analyzer

## 项目结构


embedded-blog/
├── hugo.toml                      # Hugo 主配置文件
├── .gitmodules                    # git submodule 定义
├── archetypes/
│   └── default.md                 # 文章模板（含 frontmatter 默认值）
├── content/
│   ├── about/index.md             # 关于页面
│   ├── links/index.md             # 友链页面
│   ├── search/index.md            # 搜索页面
│   ├── categories/
│   │   ├── dev-tools/_index.md    # 开发工具分类
│   │   ├── embedded-hardware/_index.md
│   │   ├── embedded-software/_index.md
│   │   └── projects/_index.md
│   └── post/
│       ├── dev-tools/
│       │   └── wsl2-embedded-dev-setup.md
│       ├── embedded-hardware/
│       │   └── stm32-gpio-guide.md
│       ├── embedded-software/
│       │   └── freertos-scheduling-deep-dive.md
│       └── projects/
│           └── rv1126-vision-module-hardware.md
├── static/
│   └── img/avatar.svg             # 侧边栏头像
└── themes/
    └── hugo-theme-stack/          # 主题（git submodule）
```

## 架构与设计决策

### Hugo 配置要点

- **Markdown 渲染**: 启用 Goldmark CJK 换行支持（eastAsianLineBreaks），允许原始 HTML（unsafe = true），代码高亮使用 Dracula 主题并带行号
- **搜索方案**: 选择 Fuse.js 客户端搜索而非 Algolia 等外部服务，contentLength 限制 4000 字符，匹配阈值 0.3
- **输出格式**: 首页同时输出 HTML、RSS、JSON（供搜索索引使用）
- **暗色主题**: 默认启用 dark color scheme，支持手动切换
- **目录结构**: 内容分类使用四个维度——嵌入式硬件、嵌入式软件、开发工具、项目实战
- **系列文章**: 通过 `series` taxonomy 支持系列连载（如 "STM32 入门系列"、"RTOS 深度解析"、"RV1126 视觉模块设计"）

### 文章模板设计

archetypes/default.md 定义了统一的文章模板，包含：

```yaml
categories:
  - embedded-software    # 默认分类
series:
  -                      # 可选系列
toc: true                # 自动生成目录
comments: false           # 默认关闭评论
license: cc-by-nc-sa     # 默认许可证
```

正文结构约定为"概述 + 正文"两段式。

### 内容创作模式

每篇文章遵循一致的写作模式：

1. **概述** — 1-2 段说明文章目标和背景
2. **理论基础** — 硬件结构、数据结构、算法原理
3. **代码实现** — 寄存器级和库级双轨对比（如 STM32 GPIO 文章同时展示寄存器操作和 HAL 库用法）
4. **性能数据** — 实测数据表格（如 FreeRTOS 任务切换时间）
5. **实践建议** — 可操作的优化技巧
6. **总结** — 核心要点回顾

## 关键内容摘要

### 已发布的文章

| 文章 | 分类 | 系列 | 核心内容 |
|------|------|------|----------|
| WSL2 + VSCode 搭建嵌入式 Linux 开发环境 | dev-tools | - | ARM 交叉编译工具链、CMake toolchain 配置、GDB 远程调试、自动化部署脚本 |
| STM32 GPIO 配置详解 | embedded-hardware | STM32 入门系列 | GPIO 寄存器（MODER/OTYPER/OSPEEDR/PUPDR/BSRR）逐位配置 vs HAL 库封装，外部中断配置 |
| FreeRTOS 任务调度机制深度解析 | embedded-software | RTOS 深度解析 | 四状态任务模型、优先级抢占调度、时间片轮转、优先级继承、vTaskDelayUntil 精确周期 |
| 基于 RV1126 的智能视觉模块设计（一） | projects | RV1126 视觉模块设计 | SOM 核心板 + 底板架构、RK809 PMIC 电源树、6 层 PCB 叠层、阻抗控制 |

## 关键代码片段

### CMake 交叉编译工具链文件

```cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR arm)
set(CMAKE_C_COMPILER arm-linux-gnueabihf-gcc)
set(CMAKE_CXX_COMPILER arm-linux-gnueabihf-g++)
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -march=armv7-a -mfpu=neon-vfpv4 -mfloat-abi=hard")
```

### STM32 GPIO BSRR 原子操作

```c
// 置位（高电平）— 原子操作，无需读-改-写
GPIOA->BSRR = (1U << 5);
// 复位（低电平）
GPIOA->BSRR = (1U << (5 + 16));
```

### FreeRTOS 精确周期任务

```c
TickType_t xLastWakeTime = xTaskGetTickCount();
for (;;) {
    doWork();
    vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(10));  // 精确 10ms 周期
}
```

## 构建与运行

```bash
# 克隆项目（含 submodule）
git clone --recurse-submodules <repo-url>
cd embedded-blog

# 初始化 submodule（如果已克隆但未拉取 submodule）
git submodule update --init --recursive

# 本地预览
hugo server -D          # 包含草稿文章
hugo server             # 仅已发布文章

# 构建静态文件
hugo                    # 输出到 public/ 目录

# 部署到 Azure
# 将 public/ 目录内容上传到 Azure Web App 或 Static Website
```

## 关键学习与洞察

1. **寄存器 vs HAL 库的权衡** — 寄存器操作效率最高但可移植性差，HAL 库开发效率高但有性能开销。实践策略：原型验证用 HAL，性能关键路径改用 LL 库或直接寄存器操作。

2. **RTOS 优先级反转** — FreeRTOS 的互斥量内置优先级继承协议，是解决优先级反转问题的关键机制，比二值信号量更适合保护共享资源。

3. **vTaskDelay vs vTaskDelayUntil** — 前者的实际周期 = 执行时间 + 延迟值，存在累积漂移；后者基于绝对时间，周期精确无漂移，是周期性任务的正确选择。

4. **WSL2 嵌入式开发优势** — WSL2 提供原生 Linux 内核，可直接运行交叉编译工具链和 gdbserver，避免了传统 VM 的性能损耗和文件系统兼容问题。

5. **多层 PCB 设计原则** — 高速信号（RGMII、MIPI CSI、USB 3.0）需要完整的参考地平面，6 层板采用 Signal-GND-Signal-Power-GND-Signal 叠层，关键信号走内层并控制阻抗。

6. **RV1126 电源域设计** — SoC 内部不同模块（核心逻辑、NPU、DDR）有独立电源域，需通过 PMIC（RK809）分别供电并按序上下电，电源纹波和去耦设计直接影响系统稳定性。

## 相关概念链接

- [[Hugo]] — 静态站点生成器
- [[hugo-theme-stack]] — Hugo 博客主题
- [[STM32]] — ST Microelectronics MCU 平台
- [[FreeRTOS]] — 实时操作系统
- [[RV1126]] — Rockchip 视觉处理 SoC
- [[CMake]] — 跨平台构建系统
- [[WSL2]] — Windows Subsystem for Linux 2
- [[交叉编译]] — Cross Compilation
- [[PCB设计]] — 印刷电路板设计
- [[RTOS调度]] — 实时操作系统任务调度
- [[嵌入式Linux]] — Embedded Linux 开发
- [[GPIO]] — 通用输入输出接口
- [[MIPI CSI]] — 摄像头串行接口
- [[RGMII]] — Reduced Gigabit Media Independent Interface
```

## 相关笔记

- [[web-blog]] — Hugo 个人技术博客（同为 Hugo 博客）
- [[selfMedia]] — 猪猪猪序员自媒体内容制作项目
- [[redbook]] — 小红书技术内容创作素材库
- [[rv1126b]] — RV1126B 运动相机项目（博客中有 RV1126 文章）
- [[stm32]] — STM32G0B1 Makefile 工程（博客中有 STM32 文章）
- [[studyzephyr]] — Zephyr RTOS 学习项目（博客中有 FreeRTOS 文章）