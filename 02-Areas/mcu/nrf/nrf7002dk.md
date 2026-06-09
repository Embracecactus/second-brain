---
tags: [nrf5340, zephyr, ble]
category: mcu/nrf
created: 2026-06-09
---

# nRF7002DK 开发笔记

## 项目概述

本项目基于 Nordic nRF7002 DK 开发板，使用 Zephyr RTOS 进行嵌入式开发。nRF7002 DK 集成了 nRF5340 SoC（双核 Arm Cortex-M33）和 nRF7002 WiFi 6 协处理器，适用于 IoT 和无线通信应用。

### 硬件特性
- **主控芯片**: nRF5340 (双核 Cortex-M33)
  - Application core (cpuapp): 高性能应用处理器
  - Network core (cpunet): 低功耗网络处理器
- **WiFi 模块**: nRF7002 (WiFi 6)
- **开发环境**: Docker 容器化编译环境
- **调试工具**: J-Link + nRF Command Line Tools

## 关键知识点

### 1. 编译环境搭建

使用 Docker 镜像进行开发，确保环境一致性：

```bash
# 使用阿里云镜像
docker run -ti -v $PWD:/workdir \
  crpi-njzo90d6qanuirs8.cn-hangzhou.personal.cr.aliyuncs.com/lijian_docker/zephyr-ci:v1.0

# 进入容器后
cd workdir/

# 初始化 Zephyr 项目
west init .

# 拉取必要模块
west update nrf_hw_models
west update hal_nordic
west update cmsis
west update cmsis_6
west update trusted-firmware-m  # TF-M 模块，CONFIG_BUILD_WITH_TFM=y 需要
west zephyr-export
```

**项目目录结构**:
```
mcu-zephyr-project/
├── build/              # 编译输出目录
├── frdm-mcxa346/       # 其他板子 app 程序
├── modules/            # 模块和 SDK
├── nxp-zephyrREADME.md
└── zephyr/             # Zephyr 源码
```

### 2. 板级应用开发

#### Application Core (cpuapp) 编译

```bash
# 创建应用目录
mkdir -p nrf-7002dk/nrf5340-cpuapp
cd nrf-7002dk/nrf5340-cpuapp

# 应用目录结构
nrf-7002dk/
├── nrf-7002dk.overlay    # 设备树覆盖文件
├── CMakeLists.txt        # CMake 配置
├── prj.conf              # 项目配置
└── src/
    └── main.c            # 主程序

# 编译
west build -b nrf7002dk/nrf5340/cpuapp
```

#### Network Core (cpunet) 编译

```bash
# 创建应用目录
mkdir -p nrf-7002dk/nrf5340-cpunet
cd nrf-7002dk/nrf5340-cpunet

# 编译
west build -b nrf7002dk/nrf5340/cpunet
```

### 3. 下载和调试环境

#### 必需工具
1. **J-Link**: https://www.segger.com/downloads/jlink/
2. **nRF Command Line Tools**: 包含 nrfjprog 等工具
3. **nRF Connect for Desktop**: 安装 programmer 插件

#### 参考资料
- https://www.cnblogs.com/HannibalWang/p/17210880.html

#### 常用命令
```bash
# 查看 nrfjprog 版本
nrfjprog -v
```

## 技术细节

### 1. CMake 配置

```cmake
cmake_minimum_required(VERSION 3.20.0)
set(BOARD "nrf7002dk/nrf5340/cpuapp")

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})

project(nrf7002dk_nrf5340_cpuapp)

FILE(GLOB_RECURSE app_sources src/main.c)

include_directories(app PRIVATE src)

target_sources(app PRIVATE ${app_sources})
```

**关键点**:
- 指定目标板为 `nrf7002dk/nrf5340/cpuapp`
- 使用 Zephyr 构建系统
- 自动收集 src 目录下的源文件

### 2. 项目配置 (prj.conf)

```kconfig
# SPDX-License-Identifier: Apache-2.0

# 启用 MPU (Memory Protection Unit)
CONFIG_ARM_MPU=y

# 启用 TrustZone-M
CONFIG_ARM_TRUSTZONE_M=y

# 启用 GPIO
CONFIG_GPIO=y

# 启用 UART 驱动
CONFIG_SERIAL=y

# 启用控制台输出
CONFIG_CONSOLE=y
CONFIG_UART_CONSOLE=y

# 启用 RNG (Random Number Generator)
CONFIG_MBEDTLS_PSA_CRYPTO_EXTERNAL_RNG=y
CONFIG_MBEDTLS_PSA_CRYPTO_EXTERNAL_RNG_ALLOW_NON_CSPRNG=y

# ISN 需要 CS-Rand，nRF 板子上游不支持
CONFIG_NET_TCP_ISN_RFC6528=n
```

**配置说明**:
- **ARM_MPU**: 启用内存保护，增强安全性
- **ARM_TRUSTZONE_M**: 启用 TrustZone 安全扩展
- **GPIO**: 通用输入输出接口
- **SERIAL/CONSOLE**: UART 串口通信和控制台输出
- **MBEDTLS_PSA_CRYPTO_EXTERNAL_RNG**: 使用外部硬件随机数生成器
- **NET_TCP_ISN_RFC6528**: 禁用 TCP ISN（因 nRF 不支持 CS-Rand）

### 3. 设备树覆盖文件 (nrf-7002dk.overlay)

设备树覆盖文件用于自定义硬件配置。当前文件为空，表示使用默认配置。

## 代码片段

### Hello World 示例 (main.c)

```c
/*
 * Copyright (c) 2012-2014 Wind River Systems, Inc.
 *
 * SPDX-License-Identifier: Apache-2.0
 */

#include <stdio.h>

int main(void)
{
    printf("Hello World! %s\n", CONFIG_BOARD_TARGET);

    return 0;
}
```

**代码说明**:
- 使用标准 C 库的 `printf` 输出
- `CONFIG_BOARD_TARGET` 是 Zephyr 构建系统定义的宏，表示当前目标板
- 程序启动后输出 "Hello World!" 和目标板信息

## 开发流程总结

1. **环境准备**:
   - 安装 Docker
   - 拉取 Zephyr CI 镜像
   - 配置 J-Link 和 nRF 工具

2. **项目初始化**:
   - 创建项目目录结构
   - 使用 `west init` 初始化 Zephyr
   - 使用 `west update` 拉取依赖模块

3. **应用开发**:
   - 编写 CMakeLists.txt
   - 配置 prj.conf
   - 编写设备树覆盖文件（如需要）
   - 开发应用程序代码

4. **编译和下载**:
   - 使用 `west build` 编译
   - 使用 `nrfjprog` 或 nRF Connect Programmer 下载

5. **调试**:
   - 使用 J-Link 调试器
   - 查看串口输出

## 相关链接

- **Zephyr 官方文档**: https://docs.zephyrproject.org/
- **nRF7002 DK 产品页**: https://www.nordicsemi.com/Products/nRF7002-DK
- **J-Link 下载**: https://www.segger.com/downloads/jlink/
- **nRF Connect for Desktop**: https://www.nordicsemi.com/Products/Development-tools/nRF-Connect-for-Desktop
- **参考博客**: https://www.cnblogs.com/HannibalWang/p/17210880.html

## 注意事项

1. **双核架构**: nRF5340 有 cpuapp 和 cpunet 两个核心，需要分别编译和下载
2. **模块依赖**: 确保拉取所有必要的 west 模块，特别是 trusted-firmware-m
3. **设备树**: 使用 overlay 文件可以灵活配置硬件，无需修改源码
4. **安全性**: 启用 MPU 和 TrustZone 可以增强系统安全性
5. **随机数**: nRF 板子使用外部硬件 RNG，需配置相关选项

---

**创建时间**: 2026-06-09
**适用硬件**: nRF7002 DK (nRF5340 + nRF7002)
**开发环境**: Zephyr RTOS + Docker
