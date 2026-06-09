---
title: esp32-box-lite
category: mcu/esp32
tags:
  - esp32
  - esp32s3
  - esp-box-lite
  - iot
  - embedded
  - idf
  - freertos
created: 2026-06-09
status: in-progress
---

# esp32-box-lite

## 项目概述

基于 ESP-IDF 框架为乐鑫 ESP32-S3-BOX-Lite 开发板构建的嵌入式项目。当前处于初始 Hello World 阶段，通过 Docker 容器化开发环境实现跨平台构建，目标是利用该开发板的 HMI 能力（LCD 显示、音频输入输出、触摸按键等）进行 IoT 应用开发。

## 技术栈

| 类别 | 技术 |
|------|------|
| 芯片 | ESP32-S3（Xtensa LX7 双核，240MHz） |
| 框架 | ESP-IDF v5.1（sdkconfig 中标记为 v6.0.0） |
| RTOS | FreeRTOS（双核，100Hz tick） |
| 构建系统 | CMake + idf.py |
| 开发环境 | Docker（`espressif/idf:release-v5.1`） |
| 语言 | C |
| BSP | esp-box v0.5.0 组件包 |
| 安全 | mbedTLS（TLS 1.2、RSA/ECDHE） |
| 网络 | WiFi（WPA3-SAE）、LWIP、MQTT |

## 架构与关键设计

### 项目结构

```
esp32-box-lite/
├── esp-box-0.5.0.tar.gz          # ESP-BOX BSP 组件包
└── esp32s3-box-lite/              # 主工程目录
    ├── CMakeLists.txt             # 顶层 CMake（project 声明）
    ├── Dockerfile                 # Docker 开发环境定义
    ├── sdkconfig                  # ESP-IDF 配置文件
    ├── sdkconfig.old
    ├── README.md
    ├── main/
    │   ├── CMakeLists.txt         # 组件注册
    │   └── esp32s3-box-lite.c     # 主入口 app_main()
    └── build/                     # 构建产物
```

### 开发模式

项目采用 Docker 容器化开发，避免本地安装 ESP-IDF 工具链。通过挂载项目目录到容器内，使用 `idf.py` 命令进行编译和烧录。

### BSP 组件管理

通过 `idf_component.yml` 清单文件管理依赖，支持从乐鑫组件注册表（`components.espressif.com`）拉取第三方组件。BSP 组件通过 `EXTRA_COMPONENT_DIRS` 引入，配置 `CONFIG_BSP_BOARD_ESP32_S3_BOX_Lite=y` 选择目标开发板。

## 核心知识点

### ESP32-S3-BOX-Lite 硬件特性

- **CPU**: Xtensa LX7 双核，最高 240MHz，默认 160MHz
- **Flash**: 当前配置 2MB（DIO 模式，80MHz）
- **外设支持**: I2S（音频）、SPI、I2C、LCD（RGB/I80）、USB OTG、Touch Sensor
- **WiFi**: 802.11 b/g/n，支持 WPA3-SAE
- **蓝牙**: BLE 5.0 + BLE Mesh
- **显示**: 支持 RGB LCD 和 I80 并口 LCD，16-bit 数据宽度
- **分区表**: 单应用分区（`partitions_singleapp.csv`），偏移 0x8000

### idf.py 核心命令

| 命令 | 用途 |
|------|------|
| `idf.py create-project <name>` | 创建新工程 |
| `idf.py set-target esp32s3` | 设置目标芯片 |
| `idf.py menuconfig` | 图形化配置菜单 |
| `idf.py build` | 编译项目 |
| `idf.py flash` | 烧录固件 |
| `idf.py monitor` | 串口监控（115200 baud） |
| `idf.py create-manifest` | 创建组件清单文件 |
| `idf.py add-dependency` | 添加组件依赖 |
| `idf.py reconfigure` | 重新配置（更新依赖后） |

### Kconfig 自定义选项

项目通过 `components/` 目录下的 `Kconfig.projbuild` 添加板级选择菜单，支持 ESP32-S3-BOX 和 ESP32-S3-BOX-Lite 两个变体，BOX-Lite 不支持触摸按键功能。

## 重要代码片段

### 主程序入口（Hello World）

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

void app_main(void)
{
    while (1)
    {
        vTaskDelay(1000);
        printf("Hello World!\n");
    }
}
```

当前为最简模板，在 FreeRTOS 任务循环中每秒打印一次 Hello World。

### 顶层 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.16)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(esp32s3-box-lite)
```

标准 ESP-IDF 项目 CMake 模板，通过环境变量 `IDF_PATH` 引入 ESP-IDF 构建系统。

## 构建/运行方法

### Docker 环境启动

```bash
docker pull espressif/idf:release-v5.1
docker run --rm -v $PWD:/project -w /project -u $UID -e HOME=/tmp -it espressif/idf:release-v5.1
```

### 创建新工程流程

```bash
idf.py create-project my-project
rm -rf build
idf.py set-target esp32s3
idf.py menuconfig
# 在 sdkconfig 中设置: CONFIG_BSP_BOARD_ESP32_S3_BOX_Lite=y
# 修改分区为自定义分区表
# 修改 Flash 大小为 16MB
idf.py build
idf.py flash monitor
```

### 移植 BSP 组件

将 `esp-box-0.5.0/components` 目录复制到新工程根目录，在顶层 `CMakeLists.txt` 中添加：

```cmake
set(EXTRA_COMPONENT_DIRS ../components)
```

### 烧录工具

使用乐鑫官方 [Flash 下载工具](https://docs.espressif.com/projects/esp-test-tools/zh_CN/latest/esp32/production_stage/tools/flash_download_tool.html) 根据分区信息进行离线烧录。

## 相关笔记链接

- [[esp32-idf]]
- [[esp32s3-box-lite-bsp]]
- [[freertos-task-management]]
- [[idf-component-manager]]
- [[esp32-kconfig]]
- [[mcu/esp32/esp32-wifi]]
- [[docker-embedded-development]]

## 相关笔记

- [[esp-idf-v5-guide]] — ESP-IDF v5 开发指南
- [[esp32c3]] — ESP32-C3 智能尾灯项目
- [[brithday]] — ESP32 生日项目合集
- [[arduino]] — ESP32 墨水屏驱动项目
- [[docker-alicloud]] — Docker 镜像推送到阿里云 ACR
