---
title: ImageToEpd 电子纸取模软件
aliases: [ImageToEpd, 电子纸取模, EPD图像转换]
tags: [嵌入式, 电子纸, EPD, 取模工具, GoodDisplay, 图像处理]
created: 2026-06-09
source: Good Display 官方文档 + 本地软件包
version: V4.2
author: 大连佳显电子 (Good Display)
---

# ImageToEpd 电子纸取模软件

## 概述

ImageToEpd 是大连佳显电子 (Good Display) 开发的专用电子纸 (EPD) 图像取模软件，用于将常见图片格式转换为可在电子墨水屏上显示的 C 语言数组数据。软件将复杂的参数设置集成到内部，用户只需选择电子纸类别即可一键完成取模操作，大幅降低了电子纸开发中图像处理的门槛。

## 核心要点

- **一键取模**: 选择电子纸型号后自动生成对应数组，无需手动配置参数
- **三色图自动分离**: 支持黑白红三色图直接导入，自动分离黑色和红色部分分别生成数组
- **输出格式**: 生成标准 C 数组代码 (.c 文件)，可直接嵌入嵌入式项目
- **开发背景**: 由 Good Display 针对自家电子纸产品线量身定制
- **当前版本**: V4.2 (2023年10月发布)
- **系统限制**: 仅支持 Windows 系统 (PE32 .NET 应用程序)

## 详细内容

### 1. 什么是取模

取模 (Image-to-Array Conversion) 即将图片转换为数据数组的过程。电子纸显示屏无法直接显示图片文件，需要将像素数据转换为特定格式的二进制数组，嵌入到 MCU 固件中，才能驱动 EPD 显示目标图像。

```
图片文件 → 取模软件 → C 数组数据 → 嵌入固件 → EPD 显示
```

### 2. 软件功能特性

| 特性 | 说明 |
|------|------|
| 一键取模 | 选择电子纸类别即可自动生成数组 |
| 三色图支持 | 自动分离黑白红三色图，无需手动拆分 |
| 参数内置 | 驱动 IC 参数、分辨率等已预配置 |
| 多型号支持 | 覆盖 Good Display 全系列电子纸产品 |
| 输出格式 | 生成 .c 文件，包含标准 C 数组 |

### 3. 操作流程

1. **打开软件** → 运行 `ImageToEpd v4.2.exe`
2. **选择电子纸型号** → 从下拉列表中选择对应的 EPD 类别/尺寸
3. **导入图片** → 选择要转换的图片文件
4. **执行取模** → 点击生成按钮
5. **保存数组** → 导出为 .c 文件，嵌入到项目代码中

### 4. 与其他取模工具对比

| 工具 | 平台 | 优势 | 劣势 |
|------|------|------|------|
| **ImageToEpd** | Windows | 一键取模，参数内置，三色图自动分离 | 仅 Windows，仅 Good Display 产品 |
| **Image2Lcd (应用版)** | Windows | 参数可自定义，通用性强 | 需手动配置参数，三色图需分别处理 |
| **Image2Lcd (网页版)** | 跨平台 | 跨平台，操作简单 | 需保存网页，功能相对有限 |

### 5. 软件包内容 (V4.2)

本地路径: `/mnt/c/Users/lijian/Downloads/ImageToEpd软件、说明书、操作视频 v4.2/ImageToEpd软件、说明书、操作视频/`

| 文件 | 大小 | 说明 |
|------|------|------|
| `ImageToEpd v4.2.exe` | 75 KB | 主程序 (.NET 应用) |
| `电子纸演示软件使用说明书 V4.2.pdf` | 1.8 MB | 官方使用手册 |
| `ImageToEpd 操作说明视频 v4.2.mp4` | 14.9 MB | 操作演示视频 |

### 6. 技术细节

- **文件类型**: PE32 executable (GUI) Intel 80386 Mono/.Net assembly
- **开发框架**: .NET Framework
- **输出格式**: C 语言数组，通常为 `const unsigned char` 数组
- **数据编码**: 根据电子纸类型支持 1bpp (黑白) 和多色编码
- **分辨率**: 自动匹配所选电子纸型号的标准分辨率

### 7. 适用场景

- **嵌入式开发**: 为 STM32、ESP32、Arduino 等 MCU 项目生成 EPD 显示数据
- **电子价签**: 零售电子纸价签的图像内容制作
- **电子纸标牌**: 信息展示牌的静态图像转换
- **原型开发**: 快速验证电子纸显示效果

### 8. 注意事项

- 仅支持 Windows 操作系统运行
- 生成的数组针对 Good Display 自有产品线，其他厂商 EPD 可能不兼容
- 三色图取模时，软件会自动生成黑色和红色两个独立数组
- 建议使用 PNG 或 BMP 格式图片以获得最佳转换效果

## 参考链接

- 官方取模指南: https://www.good-display.cn/news/117.html
- 官方软件页: https://www.good-display.com/companyfile/Host-Computer-Software-Image-to-EPD-318.html
- GitHub 参考: https://github.com/GoodDisplay/Forked-Software-ImageToEPD-for-E-paper-display
- Image2LCD 网页版: https://www.e-paper-display.com/Image2LCD.html
