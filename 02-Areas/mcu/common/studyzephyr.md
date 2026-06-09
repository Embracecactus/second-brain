---
tags:
  - zephyr
  - rtos
  - stm32
  - embedded
  - blinky
  - nucleo
category: mcu/common
created: 2026-06-09
status: seed
source: /home/lijian/project/studyzephyr
---

# studyzephyr - Zephyr RTOS 学习项目

## 项目概述

这是一个 Zephyr RTOS 的学习项目，记录了从零搭建 Zephyr 开发环境到创建第一个 Blinky 应用的完整过程，目标硬件平台为 STM32 Nucleo-G0B1RE 开发板。

## 技术栈

- **RTOS**: Zephyr RTOS (使用 west 构建系统)
- **SDK**: Zephyr SDK 0.17.2
- **构建工具**: CMake (>= 3.20.0) + Ninja
- **目标硬件**: STM32 Nucleo-G0B1RE (`nucleo_g0b1re`)
- **模拟器**: QEMU (`qemu_cortex_m3`)
- **编程语言**: C
- **Python**: 3.12 (venv 环境，用于 west 及依赖管理)
- **包管理**: west (Zephyr 元工具)

## 架构与关键设计

### Zephyr 项目结构

```
blinkled/
  ├── CMakeLists.txt        # CMake 构建配置，指定目标板
  ├── prj.conf              # 项目 Kconfig 配置（当前为空）
  ├── blinkled.overlay      # Device Tree Overlay（当前为空）
  └── src/
      └── main.c            # 应用入口
```

### 构建系统

Zephyr 使用 **west** 作为元工具，底层调用 CMake + Ninja。`find_package(Zephyr ...)` 加载 Zephyr 的 CMake 模块，`target_sources(app ...)` 将用户代码注册到 Zephyr 的 `app` target 中。

### Device Tree Overlay

`blinkled.overlay` 用于覆盖或扩展硬件的 Device Tree 配置（当前为空，使用板级默认配置）。

## 核心知识点

1. **west 工具链**: Zephyr 的统一构建/烧录/调试入口，`west init` 初始化工作区，`west update` 拉取所有依赖模块，`west build` 编译，`west build -t run` 在 QEMU 中运行。
2. **Zephyr CMake 包注册**: `west zephyr-export` 将 Zephyr 包路径写入 `~/.cmake/packages/Zephyr`，使 CMake 能自动发现 Zephyr。
3. **Kconfig 配置**: `prj.conf` 是项目级配置文件，用于启用/禁用内核特性（当前为空，使用默认配置）。
4. **Board 指定**: 可在 `CMakeLists.txt` 中通过 `set(BOARD ...)` 硬编码，也可通过 `west build -b <board>` 命令行指定。
5. **QEMU 模拟**: 使用 `qemu_cortex_m3` 板可在无硬件时进行开发和调试。

## 重要代码片段

### main.c - Blinky 应用

```c
#include<stdio.h>
#include<zephyr/kernel.h>

int main(void)
{
    while(1)
    {
        printf("Hello World! %s\n", CONFIG_BOARD);
        k_sleep(K_MSEC(1000));
    }
    return 0;
}
```

- `CONFIG_BOARD` 是 Zephyr 构建系统自动定义的 Kconfig 宏，对应当前目标板名称。
- `k_sleep(K_MSEC(1000))` 使用 Zephyr 内核 API 实现 1 秒延时（非阻塞式，让出 CPU）。

### CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20.0)
set(BOARD "nucleo_g0b1re")

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(blinkled)

target_sources(app PRIVATE src/main.c)
```

## 构建/运行方法

```bash
# 激活 Python 虚拟环境
source ~/project/zephyr-g0b1re/zephyrproject/.venv/bin/activate

# 编译（真实硬件）
west build -p always -b nucleo_g0b1re blinkled

# 编译并在 QEMU 中运行
west build -p always -b qemu_cortex_m3 blinkled
west build -t run
```

### 环境搭建关键步骤

1. 安装系统依赖: `sudo apt install git cmake ninja-build gperf ccache device-tree-compiler ...`
2. 创建 Python venv 并安装 west: `pip install west`
3. 初始化 Zephyr 工作区: `west init && west update`
4. 导出 CMake 包: `west zephyr-export`
5. 安装 Zephyr SDK 0.17.2: `west sdk install` 或手动解压 `./setup.sh`
6. 安装 udev 规则（烧录需要）

## 相关笔记链接

- [[Zephyr RTOS]]
- [[STM32 Nucleo 开发板]]
- [[CMake 构建系统]]
- [[嵌入式开发环境搭建]]
- [[Device Tree]]

## 相关笔记

- [[zephyr]] — Zephyr RTOS 项目笔记
- [[ncs]] — nRF Connect SDK (NCS)
- [[nrf-project]] — Nordic Zephyr 应用集合
- [[zephyr-nxp-notes]] — NXP Zephyr 开发笔记
- [[stm32]] — STM32G0B1 Makefile 工程（同为 STM32 平台）
