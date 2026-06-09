---
tags:
  - mcu
  - esp32
  - e-paper
  - waveshare
  - arduino
  - display
  - spi
category: mcu/esp32
created: 2026-06-09
status: active
project: epdesp32s3
---

# ESP32-S3 Waveshare 2.13" E-Paper 驱动项目

## 项目概述

基于 Arduino 框架在 ESP32-S3 上驱动 Waveshare 2.13inch e-Paper (B) V4 墨水屏的示例项目，支持黑白红三色显示，包含图形绘制、中英文文字渲染和预置图片展示功能。项目中包含两个版本：`epdesp32s3`（2.13b V4，新版）和 `esp32s3`（2.13b V3，旧版）。

## 技术栈

- **MCU**: ESP32-S3
- **开发框架**: Arduino
- **开发环境**: Arduino IDE
- **编程语言**: C/C++ (.ino, .cpp, .c, .h)
- **通信协议**: Software SPI（软件模拟 SPI，非硬件 SPI）
- **显示模块**: Waveshare 2.13inch e-Paper (B) V4（122x250 像素，黑白红三色）
- **图形库**: Waveshare GUI_Paint（自绘点/线/矩形/圆/文字）

## 架构与设计决策

### 项目结构

```
arduino/
├── epdesp32s3/                    # 主项目（2.13b V4 版本）
│   ├── epdesp32s3.ino             # Arduino 入口文件
│   ├── ImageData.c / .h           # 预置图片数据（4000 字节 x2）
│   └── src/
│       ├── DEV_Config.cpp / .h    # 硬件底层接口（GPIO、SPI）
│       ├── EPD.h                  # 墨水屏驱动头文件聚合
│       ├── GUI_Paint.cpp / .h     # 图形绘制库
│       ├── fonts.h                # 字体定义
│       ├── font8/12/16/20/24.cpp  # ASCII 字体（8-24号）
│       ├── font12CN.c / font24CN.c # 中文字体（GB2312）
│       └── utility/
│           ├── Debug.h
│           ├── EPD_2in13b_V4.cpp  # 墨水屏驱动实现
│           └── EPD_2in13b_V4.h
└── esp32s3/                       # 旧项目（2.13b V3 版本）
    └── esp32s3/                   # 结构类似，驱动为 EPD_2in13b_V3
```

### 关键设计决策

1. **软件 SPI 实现**：未使用 ESP32-S3 硬件 SPI，而是通过 `digitalWrite` 逐 bit 模拟 SPI 时序。注释中保留了硬件 SPI 的代码，推测是为了调试灵活性或引脚兼容性。
2. **双缓冲图像架构**：使用 `BlackImage` 和 `RYImage` 两个独立的图像缓冲区分别管理黑色和红色/黄色图层，最终通过 `EPD_2IN13B_V4_Display(black, red)` 一次性刷新到屏幕。
3. **位图打包格式**：图像数据按 1bit/pixel 打包，每字节存储 8 个像素。缓冲区大小计算公式：`(WIDTH % 8 == 0 ? WIDTH / 8 : WIDTH / 8 + 1) * HEIGHT`，对于 122x250 分辨率为 4000 字节。
4. **270 度旋转显示**：`Paint_NewImage` 中 Rotate 参数设为 270，实现横向显示（landscape mode），将 122x250 纵向屏转为 250x122 横向使用。
5. **内存动态分配**：使用 `malloc` 动态分配图像缓冲区，绘制完成后 `free` 释放，适合 ESP32-S3 内存受限环境。

## GPIO 引脚配置

| 信号 | 引脚 | 功能 |
|------|------|------|
| SCK  | GPIO 5  | SPI 时钟 |
| MOSI | GPIO 6  | SPI 数据输出 |
| CS   | GPIO 7  | 片选 |
| RST  | GPIO 8  | 复位 |
| DC   | GPIO 10 | 数据/命令选择 |
| BUSY | GPIO 11 | 忙信号（输入） |

## 代码片段

### 软件 SPI 字节写入

```cpp
void DEV_SPI_WriteByte(UBYTE data)
{
    digitalWrite(EPD_CS_PIN, GPIO_PIN_RESET);
    for (int i = 0; i < 8; i++)
    {
        if ((data & 0x80) == 0) digitalWrite(EPD_MOSI_PIN, GPIO_PIN_RESET);
        else                    digitalWrite(EPD_MOSI_PIN, GPIO_PIN_SET);
        data <<= 1;
        digitalWrite(EPD_SCK_PIN, GPIO_PIN_SET);
        digitalWrite(EPD_SCK_PIN, GPIO_PIN_RESET);
    }
    digitalWrite(EPD_CS_PIN, GPIO_PIN_SET);
}
```

### 图像缓冲区分配与初始化

```cpp
UWORD Imagesize = ((EPD_2IN13B_V4_WIDTH % 8 == 0)
    ? (EPD_2IN13B_V4_WIDTH / 8)
    : (EPD_2IN13B_V4_WIDTH / 8 + 1)) * EPD_2IN13B_V4_HEIGHT;
BlackImage = (UBYTE *)malloc(Imagesize);  // 4000 bytes
RYImage = (UBYTE *)malloc(Imagesize);
Paint_NewImage(BlackImage, EPD_2IN13B_V4_WIDTH, EPD_2IN13B_V4_HEIGHT, 270, WHITE);
```

### 中文文字渲染

```cpp
Paint_DrawString_CN(5, 15, "你好abc", &Font12CN, WHITE, BLACK);
```

## 构建与运行

1. 安装 Arduino IDE，添加 ESP32-S3 开发板支持
2. 将 `epdesp32s3/` 文件夹复制到 Arduino 项目目录
3. 在 Arduino IDE 中选择开发板 `ESP32S3 Dev Module`
4. 连接 Waveshare 2.13" e-Paper 模块，按 GPIO 引脚配置表接线
5. 编译上传，串口监视器波特率设为 115200 查看调试输出

**运行流程**：初始化 -> 清屏 -> 显示预置图片（2秒） -> 绘制几何图形+中英文文字（2秒） -> 清屏 -> 进入睡眠模式

## 关键学习与洞察

- **E-Paper 双色驱动原理**：黑白红墨水屏需要分别发送黑色图层和红色图层数据，两个图层独立控制，红色像素叠加在黑色图层之上
- **软件 SPI 的必要性**：Waveshare 的 Arduino 库默认使用软件 SPI，虽然效率低于硬件 SPI，但引脚分配更灵活，适合快速原型验证
- **图像数据预编译**：`ImageData.c` 中的数组是预先用工具将图片转换为 1bit 位图数据，直接编译到固件中，适合显示固定 Logo 或启动画面
- **ESP32-S3 的 `malloc` 使用**：在嵌入式环境中动态分配 8KB（两个 4000 字节缓冲区）是安全的，ESP32-S3 通常有 512KB SRAM
- **E-Paper 功耗特性**：墨水屏仅在刷新时耗电，静态显示零功耗，适合电池供电的 IoT 场景。代码中最后调用 `EPD_2IN13B_V4_Sleep()` 进入低功耗模式

## 相关概念

- [[ESP32-S3]] - 主控芯片
- [[E-Paper Display]] - 电子墨水屏技术
- [[SPI Protocol]] - 串行外设接口协议
- [[Waveshare]] - 墨水屏模块供应商
- [[Arduino Framework]] - 嵌入式开发框架
- [[GUI_Paint Library]] - Waveshare 图形绘制库
- [[Bit Image Encoding]] - 位图数据编码格式

## 相关笔记

- [[esp32-box-lite]] — ESP32-S3-BOX-Lite 开发板
- [[esp32c3]] — ESP32-C3 智能尾灯项目
- [[esp32s3-nfc]] — ESP32-S3 + ST25DV NFC 开发
- [[brithday]] — ESP32 生日项目合集
- [[esp-idf-v5-guide]] — ESP-IDF v5 开发指南
- [[AndroidStudio]] — Android 墨水屏 App（同为 ESP32 墨水屏生态）
- [[epaper-converter]] — 电子纸取模软件
