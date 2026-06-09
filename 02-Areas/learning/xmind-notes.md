---
tags:
  - xmind
  - mindmap
  - fpga
  - linux
category: learning
created: 2026-06-09
---

# Documents 知识库概览

## 概述

`C:\Users\lijian\Documents\` 目录是一个综合性嵌入式开发学习资料库，涵盖了 FPGA、嵌入式 Linux（I.MX6ULL）、Arduino、LVGL GUI、PCB 设计、信号分析等多个技术领域。该目录包含 XMind 思维导图笔记、PDF 技术手册、Arduino 项目、LVGL UI 工程、PCB 设计文件以及网络抓包数据等。

## 关键知识点

### 1. XMind 思维导图笔记（学习笔记）

| 文件名 | 主题 |
|--------|------|
| `FPGA点灯.xmind` | FPGA 入门 — LED 点灯实验 |
| `Linux系统.xmind` | Linux 系统知识体系 |
| `cs 发射器功能.xmind` | CS（Counter-Strike）发射器功能分析 |
| `funpack3-5.xmind` | funpack 相关知识（第3-5章） |
| `编译生成可执行文件.xmind` | 编译流程 — 从源码到可执行文件 |
| `网页控制led和rgb灯.xmind` | Web 控制 LED/RGB 灯（IoT 应用） |
| `预处理.xmind` | C/C++ 预处理器知识 |

### 2. I.MX6ULL 嵌入式 Linux 开发（正点原子）

`imx6ull-document/` 目录包含正点原子 I.MX6ULL 开发板的完整文档体系：

- **驱动开发**：`I.MX6U嵌入式Linux驱动开发指南V1.5.2.pdf`
- **C 应用编程**：`I.MX6U嵌入式Linux C应用编程指南V1.1.pdf`
- **Qt 开发**：`I.MX6U嵌入式Qt开发指南V1.0.2.pdf`
- **系统移植**：Buildroot、Yocto、Debian 文件系统构建
- **Qt 移植**：Qt4.8.4 和 Qt5.12.9 移植指南
- **OpenCV 移植**：`I.MX6U 移植OpenCV V1.3.pdf`
- **网络调试**：TFTP & NFS 网络环境搭建
- **代码规范**：`嵌入式Linux C代码规范化V1.0.pdf`

### 3. LVGL GUI 项目

`LVGL/` 目录是一个使用 **SquareLine Studio 1.5.1** 生成的 LVGL 8.3.11 UI 工程：

- 包含 Screen1 页面（ImgButton + Switch 控件）
- 使用 CMake 构建系统
- 配置要求：`LV_COLOR_DEPTH=32bit`, `LV_COLOR_16_SWAP=0`

### 4. Arduino / M5Stack 项目

- **M5StickCPlus** 项目：`sketch_jan14b.ino` — WiFi AP 扫描结果显示
- **Arduino 库**：60+ 个库，涵盖传感器、通信、显示等领域

### 5. PCB 设计工具

- **LCEDA Pro（立创 EDA）**：在线模式 PCB 设计，项目目录配置在 `projects/`
- **KiCad 10.0**：开源 PCB 设计工具

### 6. 其他工具与资源

- **COMSOL**：多物理场仿真软件（Batch 目录）
- **MATLAB**：数学计算与仿真
- **Qt Design Studio**：Qt UI 设计工具
- **MaixPyIDE**：Maix（K210）MicroPython IDE
- **Source Insight 4.0**：代码阅读与分析工具
- **Ansoft**：电磁仿真软件

### 7. 硬件原理图与文档

- `SCH_ESP32-S3-BOX-Lite_MB_V1.1_20211221.pdf` — ESP32-S3-BOX-Lite 开发板原理图
- `OrangePi_PCPlus_H3_用户手册_v3.2.pdf` — OrangePi PC Plus (H3) 用户手册
- `热成像外壳/` — 热成像仪外壳设计文件
- `热成像外壳.zip` — 外壳设计压缩包

### 8. 网络抓包数据

| 文件 | 可能用途 |
|------|----------|
| `box.pcapng` | 设备通信抓包 |
| `myboard.pcapng` | 自制板卡通信抓包 |
| `qinhengusb.pcapng` | 沁恒（WCH）USB 设备通信抓包 |

## 技术细节

### LVGL 项目架构

```
LVGL/
├── CMakeLists.txt          # CMake 构建配置
├── ui.c / ui.h             # 主 UI 入口（初始化、主题设置）
├── ui_helpers.c / .h       # UI 辅助函数（属性设置、动画、屏幕切换）
├── ui_events.h             # 事件回调声明
├── screens/
│   └── ui_Screen1.c        # Screen1 页面实现
├── components/
│   └── ui_comp_hook.c      # 组件钩子
├── fonts/                  # 字体资源
└── UI/                     # UI 资源文件
```

### Arduino M5StickCPlus 代码

```cpp
#include <M5StickCPlus.h>

void setup() {
  M5.begin();
  M5.Lcd.setCursor(7, 20);
  M5.Lcd.println("scan done");
  M5.Lcd.setCursor(5, 60);
  M5.Lcd.printf("50 AP");
}
void loop(){}
```

### LCEDA Pro 配置

```json
{
    "type": "ONLINE",
    "onlineService": "https://pro.lceda.cn",
    "APP_PROJECT_DIR": [
        "C:\\Users\\lijian\\Documents\\LCEDA-Pro\\projects",
        "C:\\Users\\lijian\\Documents\\LCEDA-Pro\\example-projects"
    ],
    "LIB_DIR": ["C:\\Users\\lijian\\Documents\\LCEDA-Pro\\libraries"]
}
```

### Arduino 关键库列表（按领域分类）

**传感器类**：ADXL345, BME68x, VL53L0X, MAX30100, SparkFun_MAX3010x, TCS34725, GP8XXX, PAJ7620U2

**通信类**：LoRa, TinyGSM, PubSubClient, ArduinoHttpClient, ArduinoJson, mcp_can, IRremote

**显示类**：U8g2, GxEPD2, M5GFX, Adafruit_GFX_Library, TFTTerminal

**M5Stack 生态**：M5StickCPlus, M5Unified, M5Family, M5HAL, M5Unit-*

## 代码/配置片段

### LVGL CMakeLists.txt

```cmake
SET(SOURCES screens/ui_Screen1.c
    ui.c
    components/ui_comp_hook.c
    ui_helpers.c)

add_library(ui ${SOURCES})
```

### LVGL UI 初始化

```c
void ui_init(void)
{
    lv_disp_t * dispp = lv_disp_get_default();
    lv_theme_t * theme = lv_theme_default_init(dispp,
        lv_palette_main(LV_PALETTE_BLUE),
        lv_palette_main(LV_PALETTE_RED),
        false, LV_FONT_DEFAULT);
    lv_disp_set_theme(dispp, theme);
    ui_Screen1_screen_init();
    lv_disp_load_scr(ui_Screen1);
}
```

## 相关链接

- 正点原子资料下载中心：http://www.openedv.com/docs/index.html
- 正点原子 I.MX6ULL 开发板介绍视频：https://www.bilibili.com/video/av68994676
- 正点原子 Qt5 演示视频：https://www.bilibili.com/video/av61255737
- 正点原子 Qt5 超流畅桌面：https://www.bilibili.com/video/BV1M54y1677N
- 立创 EDA Pro：https://pro.lceda.cn
- LVGL 官方文档：https://docs.lvgl.io/
- SquareLine Studio：https://squareline.io/
- Arduino 官方：https://www.arduino.cc/

## 相关笔记

- [[wenan]] — 嵌入式学习笔记合集
- [[imx6ull-docs]] — IMX6ULL 开发文档
- [[fpga]] — FPGA LED 流水灯项目
- [[lvgl-project]] — LVGL SquareLine Studio UI 项目
- [[pcb]] — PCB 电源管理与电池充电电路设计
- [[comsol]] — COMSOL 压电仿真项目
