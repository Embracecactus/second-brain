---
tags:
  - mcu
  - rtos
  - zephyr
  - nxp
  - embedded
  - rtkernel
category: mcu/common
created: 2026-06-09
status: active
project: zephyr
---

# Zephyr RTOS 项目笔记

## 项目概述

Zephyr 是一个可扩展的实时操作系统 (RTOS)，支持多种硬件架构，专为资源受限设备优化，内建安全机制。本项目记录了基于 NXP FRDM-MCXA346 开发板的 Zephyr 应用开发实践，包含 Docker 构建环境配置、LVGL 图形界面集成及摄像头 (OV5640) 外设驱动等内容。

## 技术栈

| 分类 | 技术 |
|------|------|
| RTOS 内核 | Zephyr v4.3.99 (开发版) |
| 构建系统 | CMake + West (Zephyr 元工具) |
| 硬件平台 | NXP FRDM-MCXA346 (ARM Cortex-M33) |
| 图形库 | LVGL (Light and Versatile Graphics Library) |
| UI 设计工具 | GUI-Guider (NXP 官方 LVGL 设计器) |
| 显示屏 | ST7735R 128x128 TFT (SPI 接口, RGB565) |
| 摄像头 | OV5640 (DVP 并行接口, 通过 SmartDMA) |
| 开发环境 | Docker (zephyr-ci / zephyr-build 镜像) |
| SDK | Zephyr SDK v0.17.4 |
| 配置系统 | Kconfig + Devicetree |
| 版本管理 | West manifest (多仓库管理) |

## 架构与关键设计

### 项目目录结构

```
zephyr/
├── mcu-zephyr-project/          # 主工程目录
│   ├── .west/config             # West 配置，指向 zephyr 子目录
│   ├── zephyr/                  # Zephyr 内核源码 (v4.3.99)
│   │   ├── west.yml             # West manifest，定义所有依赖模块
│   │   ├── Kconfig.zephyr       # 顶层 Kconfig 配置入口
│   │   └── CMakeLists.txt       # Zephyr 构建系统入口
│   ├── frdm-mcxa346/            # NXP MCXA346 应用工程
│   │   ├── CMakeLists.txt       # 应用构建脚本
│   │   ├── prj.conf             # Kconfig 项目配置
│   │   ├── app.overlay          # Devicetree overlay (硬件描述)
│   │   └── src/                 # 应用源码
│   │       ├── main.c
│   │       ├── app/             # 应用逻辑 (Shell, LVGL 封装)
│   │       └── lvgl_ui/         # GUI-Guider 生成的 UI 代码
│   ├── nrf-7002dk/              # Nordic nRF7002 DK 板级支持
│   └── modules/                 # 本地模块 (HAL, lib 等)
├── docker_zephyr/               # 自定义 Docker 镜像构建文件
└── zephyr-ci-1-0.tar.gz         # CI 镜像离线包
```

### West 多仓库管理

Zephyr 使用 `west` 工具管理数十个 Git 仓库。`west.yml` 定义了所有依赖模块：

- **HAL 层**: hal_nxp, hal_nordic, hal_st, hal_espressif 等 30+ 芯片厂商 HAL
- **中间件**: mbedtls, lvgl, littlefs, fatfs, openthread, cmsis-dsp 等
- **安全固件**: trusted-firmware-m (TF-M), trusted-firmware-a (TF-A), mcuboot
- **调试工具**: segger, percepio, mipi-sys-t

关键 `west` 命令:
```bash
west init .                    # 初始化 workspace
west update hal_nxp cmsis      # 仅拉取需要的模块 (节省时间)
west zephyr-export             # 导出 Zephyr CMake 包
west build -b frdm_mcxa346 samples/hello_world  # 编译
```

### Devicetree Overlay 设计 (`app.overlay`)

本项目通过 Devicetree overlay 描述硬件连接，关键节点：

- **MIPI DBI-SPI 显示**: ST7735R 通过 LPSPI1 连接，使用 DMA 传输
- **OV5640 摄像头**: 通过 I2C2 (LPI2C2) 配置，DVP 并行数据经 SmartDMA 采集
- **SmartDMA**: NXP 专有 DMA 引擎，地址 `0x4000e000`，中断号 108，用于摄像头数据搬运
- **GPIO 按键**: 用于 LVGL keypad input 导航

### LVGL 集成架构

```
main.c → app_lvgl_init() → LVGL 任务循环
                ↓
    GUI-Guider 生成的 UI 代码 (generated/)
                ↓
    自定义控件逻辑 (custom/)
                ↓
    ST7735R 显示驱动 (Devicetree → Zephyr display API)
```

## 核心知识点

### 1. Zephyr Kconfig 配置体系

Zephyr 使用 Kconfig 进行系统配置，`prj.conf` 是项目级配置文件。层级关系：

```
Kconfig.zephyr (入口)
  ├── boards/Kconfig        # 板级配置
  ├── soc/Kconfig           # SoC 配置
  ├── arch/Kconfig          # 架构配置
  ├── kernel/Kconfig        # 内核配置
  ├── drivers/Kconfig       # 驱动配置
  ├── lib/Kconfig           # 库配置
  └── subsys/Kconfig        # 子系统配置
```

### 2. LVGL 在 Zephyr 中的关键配置

```kconfig
# 内存池 - 使用系统堆
CONFIG_LV_Z_MEM_POOL_SYS_HEAP=y
CONFIG_LV_Z_MEM_POOL_SIZE=81920

# 显示缓冲 - 25% 屏幕面积, RGB565
CONFIG_LV_Z_VDB_SIZE=25
CONFIG_LV_Z_BITS_PER_PIXEL=16
CONFIG_LV_COLOR_DEPTH_16=y

# 字节序交换 (匹配 ST7735R 硬件)
CONFIG_LV_COLOR_16_SWAP=y

# 异步刷新线程
CONFIG_LV_Z_FLUSH_THREAD=y
CONFIG_LV_Z_FLUSH_THREAD_PRIORITY=-1
```

### 3. Devicetree Overlay 中 SmartDMA 配置

SmartDMA 是 NXP MCX 系列的专有外设，Zephyr SDK 中默认未包含其设备树定义，需手动添加：

```dts
smartdma: smartdma@4000e000 {
    compatible = "nxp,smartdma";
    reg = <0x4000e000 0x1000>;
    interrupt-parent = <&nvic>;
    interrupts = <108 0>;  // 124 - 16 = 108 (内核中断偏移)
    program-mem = <0x4000000>;
};
```

### 4. Docker 构建环境

Zephyr 官方提供三层 Docker 镜像：

| 镜像 | 用途 |
|------|------|
| `ci-base` | CI 基础工具 (不含 SDK) |
| `ci` | CI 完整环境 (含 Zephyr SDK) |
| `zephyr-build` / `devel` | 开发环境 (含 VNC 等额外工具) |

国内构建需替换 Ubuntu APT 源和 pip 源为清华镜像。

## 重要代码片段

### 应用入口 (main.c)

```c
#include <zephyr/kernel.h>
#include "app_lvgl.h"

int main(void)
{
    app_lvgl_init();  // 初始化 LVGL 图形界面
    while (1) {
        k_sleep(K_MSEC(1000));  // 主循环休眠，LVGL 由独立线程驱动
    }
}
```

### CMakeLists.txt 应用构建

```cmake
cmake_minimum_required(VERSION 3.20.0)
set(BOARD "frdm_mcxa346")
find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(frdm_mcxa346)

FILE(GLOB_RECURSE app_sources src/main.c
                              src/lvgl_ui/generated/*.c
                              src/lvgl_ui/custom/*.c
                              src/app/*.c)
target_sources(app PRIVATE ${app_sources})
```

## 构建/运行方法

### 方式一：Docker 容器内构建

```bash
# 拉取镜像 (使用加速)
docker pull docker.1ms.run/zephyrprojectrtos/zephyr-build:v0.28.7

# 启动容器，挂载工程目录
docker run -ti -v $PWD:/workdir zephyr-ci:v1.0

# 容器内初始化与编译
cd /workdir
west init .
west update hal_nxp cmsis cmsis_6
west zephyr-export
west build -b frdm_mcxa346 frdm-mcxa346
```

### 方式二：自定义 Docker 镜像构建

```bash
cd docker_zephyr/docker-image
docker build -f Dockerfile.base --build-arg UID=$(id -u) --build-arg GID=$(id -g) -t zephyr-ci-base:v1.0 .
docker build -f Dockerfile.ci --build-arg BASE_IMAGE=zephyr-ci-base:v1.0 -t zephyr-ci:v1.0 .
```

### VSCode 连接 Docker 开发

1. 安装 VSCode 扩展: `Docker`, `Remote-Containers`, `Remote-WSL`
2. WSL2 中启动容器: `docker run -ti -v $PWD:/workdir zephyr-ci:v1.0`
3. VSCode 左下角点击远程连接，选择正在运行的 Docker 容器

## 相关笔记链接

- [[MCU 嵌入式开发]]
- [[NXP MCXA346 芯片笔记]]
- [[LVGL 图形库笔记]]
- [[Devicetree 设备树]]
- [[Docker 开发环境]]
- [[Kconfig 配置系统]]
- [[CMake 构建系统]]
- [[嵌入式 RTOS 对比]] (FreeRTOS vs Zephyr vs RT-Thread)

## 相关笔记

- [[studyzephyr]] — Zephyr RTOS 学习项目
- [[ncs]] — nRF Connect SDK (NCS)
- [[nrf-project]] — Nordic Zephyr 应用集合
- [[zephyr-nxp-notes]] — NXP Zephyr 开发笔记
- [[frdm-mcxa346]] — FRDM-MCXA346 开发板设计文件
- [[lvgl]] — LVGL T5AI 项目
- [[wenan]] — 嵌入式学习笔记合集
