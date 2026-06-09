---
tags:
  - epaper
  - eink
  - image
  - 取模工具
  - GoodDisplay
  - 嵌入式
  - EPD
category: tools
created: 2026-06-09
updated: 2026-06-09
status: active
version: V4.2
author: 大连佳显电子 (Good Display)
source: 本地软件包 + 二进制逆向分析
---

# ImageToEpd 电子纸取模软件 (V4.2)

## 项目/工具概述

ImageToEpd 是大连佳显电子 (Good Display) 官方开发的专用电子纸 (EPD) 图像取模工具，用于将常见图片格式 (BMP/PNG/JPG) 转换为可直接嵌入嵌入式固件的 C 语言数组数据。软件将驱动 IC 参数、分辨率、灰度映射等复杂配置内置，用户只需选择电子纸型号即可一键完成取模操作，大幅降低了 EPD 开发中图像处理的技术门槛。V4.2 版本在三色图自动分离、多灰度等级支持和串口联调功能上做了显著增强，是 Good Display 全系列电子纸产品的配套上位机工具。

## 技术栈 / 关键特性

| 维度 | 说明 |
|------|------|
| **运行平台** | Windows (PE32 GUI 应用) |
| **开发框架** | .NET Framework 2.0 (Mono/.Net assembly) |
| **核心依赖** | `System.Drawing` (Bitmap 处理)、`System.IO.Ports` (串口通信)、`mscorlib 2.0` |
| **输出格式** | C 语言数组 (`const unsigned char`)，.c 文件 / .txt 文件 |
| **输入格式** | BMP、PNG、JPG 等常见图片格式 |
| **项目内部名** | `serial tool` (含 `serial_tool.Form1` / `serial_tool.Form2` 资源) |
| **PDB 路径** | `\Epaper demo-30--V4.2\serial tool\obj\Debug\ImageToEpd v4.2.pdb` |

### 核心功能一览

- **一键取模**: 选择 EPD 型号后自动生成对应数组，无需手动配置分辨率、驱动 IC 参数
- **三色图自动分离**: 支持黑白红 (BWR) 三色图直接导入，自动生成黑色和红色两个独立数组 (`GetPictureData` + `GetPictureDataRed`)
- **多灰度等级**: 支持 2 级 (1bpp BW)、4 级 (2bpp)、16 级 (4bpp)、256 级 (8bpp) 灰度转换
- **双驱动 IC 架构**: 同时支持 UC 系列 (`display_model_UC`) 和 SSD 系列 (`display_model_SSD`) 驱动 IC
- **图像处理**: 内置旋转 (RotateFlip)、镜像 (`button_mirror` / `button_mirror2`)、反色 (`button_reverse`)、180 度翻转 (`button_180`)、256 色转 BW (`button_256toBW`)、256 色转 16 级灰度 (`button_256to16Gray`)
- **串口联调**: 内置 SerialPort 通信模块，可直接通过串口将图像数据发送到 EPD 模块进行预览验证
- **多种导出**: 支持保存为 .c 文件 (`button_SaveFile`) 和 .txt 文件 (`button_saveTxt`)

## 架构与设计

### 数据处理流程

```
图片文件 (BMP/PNG/JPG)
    │
    ├─ 选择 EPD 型号 (display_model_UC / display_model_SSD)
    │
    ├─ 图像预处理
    │   ├─ PicToBW: 二值化转换
    │   ├─ RotateFlip: 旋转/翻转
    │   ├─ Mirror: 镜像
    │   ├─ Reverse: 反色
    │   └─ Gray 灰度处理 (2/4/16/256 级)
    │
    ├─ 取模核心
    │   ├─ GetPictureData(): 黑色通道数据提取
    │   ├─ GetPictureDataRed(): 红色通道数据提取
    │   ├─ GetPictureData_SSD(): SSD 系列专用取模
    │   ├─ GetPictureDataRed_SSD(): SSD 系列红色通道
    │   ├─ GetPictureData_4Gray(): 4 级灰度取模
    │   ├─ GetPictureData_16Gray(): 16 级灰度取模
    │   └─ GetPictureData_256Gray(): 256 级灰度取模
    │
    └─ 输出
        ├─ .c 文件 (C 数组代码，直接嵌入固件)
        ├─ .txt 文件 (纯数据文本)
        └─ 串口发送 (SerialPort → EPD 模块实时预览)
```

### 关键函数映射 (从二进制提取)

| 函数名 | 功能 |
|--------|------|
| `GetPictureData()` | 黑色通道取模 (通用) |
| `GetPictureDataRed()` | 红色通道取模 (通用) |
| `GetPictureData_SSD()` | SSD 系列驱动 IC 黑色通道取模 |
| `GetPictureDataRed_SSD()` | SSD 系列驱动 IC 红色通道取模 |
| `GetPictureData_4Gray()` | 4 级灰度取模 |
| `GetPictureData_16Gray()` | 16 级灰度取模 |
| `GetPictureData_256Gray()` | 256 级灰度取模 |
| `GetRed()` | 红色通道像素提取 |
| `PicToBW()` | 图片二值化转换 |
| `Gray2_set()` / `Gray4_set()` / `Gray16_set()` / `Gray256_set()` | 灰度阈值设置 |
| `Setting_IC_mode` | 驱动 IC 模式切换 (UC/SSD) |
| `SerialDataAnalysis()` | 串口数据分析 |
| `Picture_data_send()` | 图像数据串口发送 |
| `DataToAll()` | 数据批量处理 |

### 灰度模式说明

| 模式 | 位深度 | 颜色数 | 适用场景 |
|------|--------|--------|----------|
| BW (黑白) | 1 bpp | 2 | 标准黑白 EPD，文字/图标显示 |
| 4 Gray | 2 bpp | 4 | 简单灰度图，低功耗场景 |
| 16 Gray | 4 bpp | 16 | 中等灰度，照片级显示 |
| 256 Gray | 8 bpp | 256 | 高灰度，全彩模拟 |

## 核心知识点

### 1. 什么是取模 (Image-to-Array Conversion)

取模是将图片像素数据转换为特定格式二进制数组的过程。电子纸显示屏无法直接显示图片文件，需要将像素数据编码为 MCU 可识别的数组格式，嵌入固件后通过 SPI/I2C 接口驱动 EPD 显示目标图像。

```
图片文件 → 取模软件 → C 数组数据 → 编译进固件 → SPI/I2C → EPD 显示
```

### 2. UC vs SSD 驱动 IC 架构

从二进制分析可见，软件内部维护两套独立的取模管线：

- **UC 系列** (`display_model_UC`): 使用 `GetPictureData()` / `GetPictureDataRed()` 取模
- **SSD 系列** (`display_model_SSD`): 使用 `GetPictureData_SSD()` / `GetPictureDataRed_SSD()` 取模

两者的区别在于数据打包格式和位序不同，由 `Setting_IC_mode` 属性控制切换。选择错误的 IC 模式会导致显示乱码。

### 3. 三色图处理机制

`PictureDataRed_Flag` 标志位用于判断是否需要分离红色通道。当导入三色图 (BWR) 时，软件自动：
1. 提取黑色通道数据 → `GetPictureData()` 生成黑色数组
2. 提取红色通道数据 → `GetPictureDataRed()` 生成红色数组
3. 两个数组分别写入 .c 文件的不同段

### 4. 串口联调功能

软件内置完整的串口通信模块 (`System.IO.Ports.SerialPort`)，支持：
- 自动检测串口 (`Serial_port_detection`)
- 配置波特率 (`set_BaudRate`)
- 数据发送 (`Button_send` / `Picture_data_send`)
- 数据接收分析 (`sp_DataReceived` / `SerialDataAnalysis`)
- 连接状态指示 (`Data_Connect_OK`)

这使得开发者可以在取模后直接通过串口将数据发送到 EPD 模块进行实时预览验证，无需编译固件。

### 5. .NET Framework 2.0 应用特征

- 运行时版本: `v2.0.50727`
- 使用 `System.Drawing.Bitmap` 进行图像处理
- 使用 `PictureBox` 控件显示预览
- 支持 `RotateFlipType` 进行图像变换
- 资源文件: `serial_tool.Form1.resources` / `serial_tool.Form2.resources` (多窗体)

## 关键代码/配置片段

### 输出 C 数组格式示例

```c
// 黑色通道数据 (1bpp, 例如 200x200 分辨率)
const unsigned char Image_Black[] = {
    0xFF, 0xFF, 0xFF, 0xFF,  // 每字节 8 像素
    0x80, 0x00, 0x00, 0x01,
    // ... 共 200*200/8 = 5000 字节
};

// 红色通道数据 (三色 EPD 专用)
const unsigned char Image_Red[] = {
    0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00,
    // ... 同样 5000 字节
};
```

### 灰度数据格式 (4Gray 示例)

```c
// 4 级灰度 (2bpp), 每字节 4 像素
// 00=白, 01=浅灰, 10=深灰, 11=黑
const unsigned char Image_4Gray[] = {
    0xE4, 0xE4, 0xE4, 0xE4,  // 4 像素 = 1 字节
    // ... 共 200*200/4 = 10000 字节
};
```

### 串口配置参考

```csharp
// 软件内部串口配置 (从二进制提取)
serialPort.BaudRate = 115200;  // 常见波特率
serialPort.PortName = "COM3";  // 由 Serial_port_detection 自动检测
serialPort.RtsEnable = true;   // 启用 RTS 握手
serialPort.ReadTimeout = 1000; // 读取超时
```

## 使用方法 / 构建步骤

### 基本使用流程

1. **运行软件** → 双击 `ImageToEpd v4.2.exe` (需 Windows + .NET Framework 2.0)
2. **选择 EPD 型号** → 从下拉列表选择目标电子纸的类别/尺寸
3. **选择驱动 IC 类型** → UC 系列或 SSD 系列 (`IC_model` / `Setting_IC_mode`)
4. **导入图片** → 点击 "Open" 按钮 (`button_Pic_Open`)，选择 BMP/PNG/JPG 文件
5. **图像预处理** (可选):
   - 旋转/翻转: `button_180` / `RotateFlip`
   - 镜像: `button_mirror` / `button_mirror2`
   - 反色: `button_reverse`
   - 灰度转换: `button_256toBW` / `button_256to16Gray`
6. **执行取模** → 软件自动生成数组数据
7. **保存输出**:
   - .c 文件: `button_SaveFile` → SaveFileDialog
   - .txt 文件: `button_saveTxt` → 纯数据导出
8. **串口验证** (可选): 点击 `btnOpenCom` 打开串口 → `Button_send` 发送数据到 EPD 模块

### 灰度模式选择

| 选项 | 控件 | 说明 |
|------|------|------|
| 4 Gray | `checkBox_4Gray` | 启用 4 级灰度模式 |
| 16 Gray | `checkBox_16Gray` | 启用 16 级灰度模式 |
| 256 Gray | `checkBox_256Gray` | 启用 256 级灰度模式 |
| Partial | `checkBox_Part` | 局部刷新模式 |

### 软件包内容 (V4.2)

本地路径: `/mnt/c/Users/lijian/Downloads/ImageToEpd软件、说明书、操作视频 v4.2/ImageToEpd软件、说明书、操作视频/`

| 文件 | 大小 | 说明 |
|------|------|------|
| `ImageToEpd v4.2.exe` | 74 KB | 主程序 (.NET PE32 GUI) |
| `电子纸演示软件使用说明书 V4.2.pdf` | 1.8 MB | 官方使用手册 (PDF) |
| `ImageToEpd 操作说明视频 v4.2.mp4` | 15 MB | 操作演示视频 |

### 注意事项

- 仅支持 Windows 操作系统，需 .NET Framework 2.0 运行时
- 生成的数组针对 Good Display 自有产品线，其他厂商 EPD 需验证兼容性
- 三色图取模时自动生成黑色和红色两个独立数组
- 建议使用 PNG 或 BMP 格式图片以获得最佳转换效果
- UC 和 SSD 驱动 IC 的取模数据格式不同，选择错误会导致显示异常
- 串口联调功能可跳过编译固件步骤，直接预览 EPD 显示效果

## 与其他取模工具对比

| 工具 | 平台 | 优势 | 劣势 |
|------|------|------|------|
| **ImageToEpd** | Windows | 一键取模，参数内置，三色图自动分离，串口联调 | 仅 Windows，仅 Good Display 产品 |
| **Image2Lcd (应用版)** | Windows | 参数可自定义，通用性强 | 需手动配置参数，三色图需分别处理 |
| **Image2Lcd (网页版)** | 跨平台 | 跨平台，操作简单 | 需保存网页，功能相对有限 |

## 参考链接

- 官方取模指南: https://www.good-display.cn/news/117.html
- 官方软件页: https://www.good-display.com/companyfile/Host-Computer-Software-Image-to-EPD-318.html
- GitHub 参考: https://github.com/GoodDisplay/Forked-Software-ImageToEPD-for-E-paper-display
- Image2LCD 网页版: https://www.e-paper-display.com/Image2LCD.html

## 相关笔记

- [[arduino]] — ESP32 墨水屏驱动项目 (Waveshare e-Paper)
- [[AndroidStudio]] — AndroidStudioProjects (cherry-app 墨水屏控制)
- [[brithday]] — ESP32-S3 生日项目 (L-ink 电子墨水屏 + NFC)
- [[esp32c3]] — ESP32-C3 开发板笔记
- [[imagetoepd-epaper]] — ImageToEpd 原始笔记 (详细功能说明)
- [[stm32]] — STM32 嵌入式开发笔记
