---
tags:
  - ok1126b
  - rockchip
  - sdk
category: embedded-linux/rockchip
created: 2026-06-09
---

# Embedded Linux 项目知识库

## 项目概述

本文档汇总自 `C:\Users\lijian\Documents\` 目录下的所有技术项目知识，涵盖嵌入式 Linux 开发、PCB 设计、UI 开发和物联网硬件开发等多个方向。主要涉及以下核心项目：

| 项目 | 芯片/平台 | 工具链 | 说明 |
|------|-----------|--------|------|
| imx6ull-document | NXP i.MX6ULL | Buildroot/Yocto/Qt | 正点原子嵌入式 Linux 开发板全套文档 |
| LCEDA-Pro PCB | Rockchip RV1126B | 立创EDA Pro | OK1126B 核心板 PCB 设计工程 |
| LVGL UI | 通用 MCU | SquareLine Studio + LVGL 8.3.11 | 嵌入式 GUI 界面开发 |
| Arduino | ESP32 (M5StickCPlus) | Arduino IDE | 物联网终端原型开发 |
| 热成像外壳 | -- | 3D STEP/STL | 热成像设备结构设计 |

## 关键知识点

### 1. I.MX6ULL 嵌入式 Linux 开发（正点原子）

#### 开发板资源体系

正点原子 I.MX6ULL 开发板提供完整的嵌入式 Linux 学习路径，按难度递进：

1. **Linux 之 Ubuntu 入门篇** -- 虚拟机环境搭建、Linux 基础操作
2. **Linux 之 ARM 裸机篇** -- 寄存器级编程、中断、时钟树
3. **Linux 之系统移植和文件系统构建篇** -- U-Boot 移植、Kernel 移植、RootFS 构建
4. **Linux 之驱动开发篇** -- 字符设备驱动、platform 驱动、设备树

#### 核心开发指南文档

| 文档 | 版本 | 用途 |
|------|------|------|
| I.MX6U 嵌入式 Linux 驱动开发指南 | V1.5.2 | 驱动开发全流程，含环境搭建、裸机、U-Boot/Kernel 移植 |
| I.MX6U 嵌入式 Linux C 应用编程指南 | V1.1 | 用户态 C 编程，文件IO、多线程、网络编程 |
| I.MX6U 嵌入式 Qt 开发指南 | V1.0.2 | Qt 应用程序开发 |
| I.MX6U 用户快速体验 | V1.8 | 硬件验证、系统烧写、资源测试 |
| Buildroot 用户手册中文版 | V1.0 | Buildroot 构建系统使用 |

#### 文件系统方案对比

| 方案 | 适用场景 | 注意事项 |
|------|----------|----------|
| Buildroot | 定制化产品，轻量级 | 推荐方案，灵活性高 |
| BusyBox | 最小化系统 | 适合资源受限场景 |
| Yocto | 企业级定制 | 需 120G+ 磁盘、10G+ 内存，编译周期长（一周+），需访问国外站点 |
| Debian | 体验/学习 | 仅用于功能体验，一般产品不使用 |

#### Qt 开发工作流

- **交叉编译环境搭建**：参考出厂系统 Qt 交叉编译环境搭建文档，在出厂系统上直接运行用户 Qt 应用
- **Qt 移植**：支持 Qt 4.8.4 和 Qt 5.12.9 两个版本移植
- **注意**：移植 Qt 不是必须的，搭建交叉编译环境后编译应用程序即可在出厂系统运行

#### 网络调试环境

- **TFTP**：用于内核调试，快速加载 kernel image 到开发板
- **NFS**：挂载网络文件系统，方便开发调试
- 支持电脑直连开发板和局域网两种连接方式

### 2. RV1126B / OK1126B PCB 设计（立创EDA Pro）

#### 项目工程

LCEDA-Pro 目录下包含多个 RV1126B 相关的 PCB 设计工程：

| 工程名称 | 说明 |
|----------|------|
| `MYZR-RV1126B-MB221-REVA` | 明远智睿 RV1126B 核心板底板设计 |
| `PRO-RV1126B-B-V11` | RV1126B 专业版 PCB 设计 |
| `RV11126B` | RV1126B 主板工程 |
| `OK1126Bx-S-V1.1` | 飞凌 OK1126B 核心板适配设计 |
| `电源管理模块` | 电源模块独立设计 |

#### RV1126B 芯片特性

Rockchip RV1126B 是一款面向 AI 视觉应用的 SoC：
- **CPU**: 四核 ARM Cortex-A7
- **NPU**: 2.0 TOPS 神经网络加速
- **ISP**: 14MP 图像信号处理器
- **视频编码**: 4K H.265/H.264
- **典型应用**: 智能安防、人脸识别、行车记录仪

### 3. LVGL 嵌入式 GUI 开发

#### 项目配置

- **SquareLine Studio 版本**: 1.5.1
- **LVGL 版本**: 8.3.11
- **项目名称**: SquareLine_Project
- **颜色深度**: 32-bit（`LV_COLOR_DEPTH != 32` 会触发编译错误）
- **字节序**: `LV_COLOR_16_SWAP = 0`（非交换模式）

#### 项目结构

```
LVGL/
├── CMakeLists.txt          # CMake 构建配置
├── filelist.txt            # 源文件列表
├── ui.c                    # UI 初始化入口
├── ui.h                    # UI 头文件
├── ui_events.h             # 事件回调声明
├── ui_helpers.c            # UI 辅助函数实现
├── ui_helpers.h            # UI 辅助函数头文件
├── screens/
│   └── ui_Screen1.c        # Screen1 屏幕初始化
├── components/
│   └── ui_comp_hook.c      # 组件 hook
├── fonts/                  # 字体资源
└── UI/                     # UI 子模块
```

#### Screen1 界面组成

- `lv_obj_t * ui_Screen1` -- 主屏幕对象
- `lv_obj_t * ui_ImgButton1` -- 图片按钮，居中对齐
- `lv_obj_t * ui_Switch1` -- 开关控件，50x25 像素，位于左上角区域（坐标 -293, -176）

#### UI Helper 关键 API

`ui_helpers.c` 提供了 SquareLine Studio 生成的标准辅助函数：

- **属性设置**: `_ui_bar_set_property`, `_ui_basic_set_property`, `_ui_label_set_property` 等
- **屏幕切换**: `_ui_screen_change` -- 支持动画淡入淡出
- **控件操作**: `_ui_flag_modify` / `_ui_state_modify` -- 标志位和状态的增/删/切换
- **动画系统**: `_ui_anim_callback_set_x/y/width/height/opacity` -- 基于 `ui_anim_user_data_t` 结构体的动画回调
- **文本绑定**: `_ui_arc_set_text_value`, `_ui_slider_set_text_value` -- 将控件值格式化为文本

### 4. Arduino / M5StickCPlus 开发

#### 代码片段：WiFi 扫描结果显示

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

#### 依赖库

- **Adafruit_GFX_Library** -- 通用图形库，支持 SPI TFT、GrayOLED
- **Adafruit_BusIO** -- I2C/SPI 总线抽象层
- 支持 CMake 和 Arduino IDE 两种构建方式

### 5. 热成像设备结构设计

热成像外壳目录包含 3D 模型文件：
- `热成像外壳.STEP` / `热成像外壳2 新.STEP` -- 外壳主体 STEP 格式
- `按钮.stl` -- 按钮部件 STL 格式
- `开关.STEP` -- 开关部件 STEP 格式

## 技术细节

### I.MX6ULL 常见开发问题

1. **系统烧写失败**：参考用户快速体验文档中的烧写章节
2. **Qt 交叉编译报错**：确保使用正确的交叉编译工具链（arm-linux-gnueabihf-gcc）
3. **Yocto 构建卡顿**：确保虚拟机分配 10G+ 内存，120G+ 剩余磁盘空间
4. **OpenCV 移植**：仅提供移植方法，使用需自行查阅资料
5. **开机 Logo 修改**：参考修改开机进度条参考手册

### LVGL 开发注意事项

- SquareLine Studio 生成的代码不要手动修改，重新生成会覆盖
- `LV_COLOR_DEPTH` 必须为 32-bit，否则编译报错
- 屏幕切换使用 `_ui_screen_change` 实现懒加载（首次切换时调用 `target_init`）
- 动画系统通过 `ui_anim_user_data_t` 结构体传递目标对象和图像集

### PCB 设计工作流（立创EDA Pro）

- 使用立创EDA Pro 进行原理图和 PCB Layout
- 在线工程通过 `online-projects-backup` 目录自动备份
- 本地工程存储在 `projects/` 目录下，格式为 `.eprj2`
- 元器件库中包含 `rv1126.elib` 自定义库

## 代码片段

### LVGL CMake 构建配置

```cmake
SET(SOURCES screens/ui_Screen1.c
    ui.c
    components/ui_comp_hook.c
    ui_helpers.c)

add_library(ui ${SOURCES})
```

### LVGL 屏幕切换（懒加载模式）

```c
void _ui_screen_change(lv_obj_t ** target, lv_scr_load_anim_t fademode, int spd, int delay, void (*target_init)(void))
{
    if(*target == NULL)
        target_init();
    lv_scr_load_anim(*target, fademode, spd, delay, false);
}
```

### LVGL 标志位操作（增/删/切换）

```c
void _ui_flag_modify(lv_obj_t * target, int32_t flag, int value)
{
    if(value == _UI_MODIFY_FLAG_TOGGLE) {
        if(lv_obj_has_flag(target, flag)) lv_obj_clear_flag(target, flag);
        else lv_obj_add_flag(target, flag);
    }
    else if(value == _UI_MODIFY_FLAG_ADD) lv_obj_add_flag(target, flag);
    else lv_obj_clear_flag(target, flag);
}
```

### M5StickCPlus LCD 显示

```cpp
#include <M5StickCPlus.h>

void setup() {
  M5.begin();
  M5.Lcd.setCursor(7, 20);
  M5.Lcd.println("scan done");
  M5.Lcd.setCursor(5, 60);
  M5.Lcd.printf("50 AP");
}
```

## 相关链接

### 正点原子 I.MX6ULL

- 资料下载中心: http://www.openedv.com/docs/index.html
- 开发板介绍视频: https://www.bilibili.com/video/av68994676
- Qt5 演示视频: https://www.bilibili.com/video/av61255737
- Qt5 新版超流畅桌面: https://www.bilibili.com/video/BV1M54y1677N
- 开发板资料网盘: https://pan.baidu.com/s/1inZtndgN-L3aVfoch2-sKA (提取码: m65i)
- PDF 合集资料: https://pan.baidu.com/s/1FSJY3PdFgV2WV4lT6Ps8fA (提取码: qsxb)
- 视频 PPT 笔记: https://pan.baidu.com/s/12C3yhpVuugtaHkhWjxfsSg (提取码: ho0m)

### 配套视频教程（百度网盘）

- Linux 之 Ubuntu 入门篇: https://pan.baidu.com/s/1GNdsA6lhPy15vahOYMMngw (提取码: 80vf)
- Linux 之 ARM 裸机篇: https://pan.baidu.com/s/1TjaQSuRZK0OiUCqc6S0SiQ (提取码: r27n)
- Linux 之系统移植和文件系统构建篇: https://pan.baidu.com/s/1ZlhaCTsdBlYdSAWVtQH_sw (提取码: x2z8)
- Linux 之驱动开发篇: https://pan.baidu.com/s/1JU95JHG-v7MKvkvXsMNhdw (提取码: n3ju)

### 工具与平台

- 立创EDA Pro: https://lceda.cn/pro
- SquareLine Studio (LVGL GUI 设计器): https://squareline.io
- LVGL 官方文档: https://docs.lvgl.io/8.3
- Arduino IDE: https://www.arduino.cc/en/software
- M5Stack 文档: https://docs.m5stack.com
