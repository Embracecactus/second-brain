---
tags:
  - mcu
  - esp32
  - esp32-s3
  - lvgl
  - nfc
  - stm32
  - iot
  - ota
  - mqtt
  - usb
  - wearable
category: mcu/esp32
created: 2026-06-09
status: active
project_path: /home/lijian/project/brithday
---

# brithday - ESP32 生日项目合集

## 项目概述

brithday 是一个以 ESP32-S3 为核心的嵌入式项目合集，主要围绕智能手表、NFC 卡片、USB 大容量存储设备等硬件产品进行开发。项目包含 LVGL GUI 驱动的圆形手表界面、MQTT 5.0 物联网通信、OTA 远程升级、TinyUSB MSC 设备、NFC 读写等多个子项目，同时涵盖了 STM32 平台的 L-ink 电子墨水屏 NFC 智能卡片方案。

## 技术栈

| 技术 | 说明 |
|------|------|
| ESP-IDF | v6.0.0，Espressif 官方 IoT 开发框架 |
| ESP32-S3 | 主控芯片，双核 Xtensa LX7，支持 USB-OTG |
| LVGL 8 | 嵌入式 GUI 库，360x360 圆形屏，RGB888 32bit 色深 |
| GC9C01 | 圆形 TFT LCD 驱动 IC，QSPI 4线接口，50MHz 时钟 |
| CST820S | I2C 电容触摸屏控制器，5点触控 |
| MQTT 5.0 | 物联网通信协议，支持共享订阅、消息属性等高级特性 |
| TinyUSB | USB 设备协议栈，实现 MSC 大容量存储 |
| CherryUSB | 国产 USB 协议栈，支持 CDC/HID/MSC |
| OTA | HTTP Over-The-Air 远程固件升级 |
| SPIFFS / FatFS | 嵌入式文件系统 |
| FreeRTOS | 实时操作系统，任务调度 |
| STM32L051 | L-ink NFC 卡片的低功耗 MCU |
| ST25DV | NFC/RFID 双接口 IC，ISO 15693 协议 |
| CMake | ESP-IDF 构建系统 |
| Docker | espressif/idf 镜像用于开发环境 |

## 架构与关键设计

### 子项目结构

```
brithday/
├── esp32-s3_watch/       # 核心：ESP32-S3 智能手表项目
│   ├── main/             # 主程序入口
│   ├── components/       # 组件化设计
│   │   ├── gc9c01/       # LCD 显示驱动 (SPI, 360x360)
│   │   ├── cst820s/      # 触摸屏驱动 (I2C)
│   │   ├── lvgl/         # LVGL GUI 库
│   │   ├── ui/           # SquareLine Studio 生成的 UI
│   │   ├── wifi/         # WiFi STA 连接
│   │   ├── mqtt5_lianxi/ # MQTT 5.0 客户端
│   │   ├── native_ota_simple/ # OTA 升级
│   │   ├── sdcard/       # SD 卡驱动
│   │   ├── cherryusb/    # CherryUSB 协议栈
│   │   └── spiffs_vfs/   # SPIFFS 文件系统
│   └── SQ_PROJ/          # SquareLine Studio UI 工程
├── ESP32S3_LVGL8_RGB888/ # LVGL8 + RGB888 基础示例
├── tusb_msc/             # TinyUSB MSC 大容量存储示例
├── esp32s3/              # ESP32-S3 + ST25DV NFC 项目
├── L-ink_Card/           # STM32L051 电子墨水屏 NFC 卡片
├── X-CUBE-NFC4/          # ST NFC 中间件库
├── g0-nfc4/              # STM32G0 NFC 项目
└── my-project/           # NFC4 APP 工程
```

### 智能手表核心架构

应用启动流程：
1. **NVS Flash 初始化** - 非易失性存储
2. **LCD 初始化** - GC9C01 SPI 驱动，360x360 圆形屏
3. **触摸屏初始化** - CST820S I2C 驱动
4. **LVGL 初始化** - GUI 框架 + PNG 解码器 + 显示/输入驱动注册
5. **创建 FreeRTOS 任务** - LVGL 刷新任务 + UI 创建任务
6. **WiFi 连接** - STA 模式接入路由器
7. **MQTT 5.0 启动** - 连接物联网平台

### 分区表设计

```
nvs       : 24KB  (配置存储)
phy_init  : 4KB   (射频校准)
factory   : 4MB   (出厂固件)
ota_0     : 4MB   (OTA 分区 A)
ota_1     : 4MB   (OTA 分区 B)
otadata   : 8KB   (OTA 状态)
spiffs    : 1MB   (文件系统)
```

总 Flash 大小 16MB，支持双 OTA 分区切换。

## 核心知识点

### 1. LVGL RGB888 刷新机制

LVGL 默认使用 RGB565 格式，但 GC9C01 屏幕需要 RGB888。在 flush 回调中进行颜色格式转换：

- 使用 `LV_COLOR_GET_R/G/B` 宏提取各通道
- 分配 DMA 内存缓冲区 (`heap_caps_malloc` with `MALLOC_CAP_DMA`)
- 通过 SPI 推送 3 字节/像素数据到 LCD

### 2. GC9C01 圆形屏坐标对齐

圆形 LCD 驱动要求刷新区域的起始坐标和宽高必须为偶数，否则显示异常。通过 `rounder_cb` 回调在 LVGL 发送刷新前自动对齐区域。

### 3. MQTT 5.0 协议特性

相比 MQTT 3.1.1，MQTT 5.0 支持：
- **User Property** - 自定义键值对附加到消息
- **Payload Format Indicator** - 标识载荷编码格式
- **Message Expiry Interval** - 消息过期时间
- **Shared Subscription** - 共享订阅实现负载均衡
- **Request/Response** - 通过 Response Topic + Correlation Data

### 4. OTA 双分区升级

使用 `esp_ota_ops` API 实现固件升级：
- HTTP 下载新固件到 `ota_0` 或 `ota_1` 分区
- 通过 `esp_ota_set_boot_partition()` 切换启动分区
- SHA-256 校验确保固件完整性

### 5. USB MSC 大容量存储

TinyUSB MSC 模式将 ESP32-S3 的 SPI Flash 暴露为 USB 大容量存储设备：
- 使用 Wear Levelling 均衡 Flash 磨损
- Host PC 和嵌入式应用不能同时访问存储
- 支持 SPI Flash 和 SD MMC 两种存储介质

### 6. L-ink NFC 智能卡片方案

基于 STM32L051 + ST25DV 的硬件方案：
- ST25DV 仅支持 ISO 15693（RFID），不能模拟 ISO 14443（M1 卡）
- IC 卡模拟通过集成多颗 UID 芯片 + 拨轮切换实现
- 200x200 单色电子墨水屏用于显示定制内容
- NFC 能量采集供电

## 重要代码片段

### LVGL flush 回调（RGB888 转换）

```c
static void lvgl_flush_cb(lv_disp_drv_t *disp_drv, const lv_area_t *area, lv_color_t *color_p)
{
    uint32_t w = area->x2 - area->x1 + 1;
    uint32_t h = area->y2 - area->y1 + 1;
    uint32_t size = w * h;

    uint8_t *rgb888_buf = heap_caps_malloc(size * 3, MALLOC_CAP_DMA);
    for (uint32_t i = 0; i < size; i++) {
        lv_color_t c = color_p[i];
        rgb888_buf[i * 3]     = LV_COLOR_GET_R(c);
        rgb888_buf[i * 3 + 1] = LV_COLOR_GET_G(c);
        rgb888_buf[i * 3 + 2] = LV_COLOR_GET_B(c);
    }
    lcd_gc9c01_set_window(area->x1, area->y1, area->x2, area->y2);
    lcd_gc9c01_push_pixels(rgb888_buf, size);
    heap_caps_free(rgb888_buf);
    lv_disp_flush_ready(disp_drv);
}
```

### 触摸屏坐标映射

```c
static void touchpad_read_cb(lv_indev_drv_t *indev_drv, lv_indev_data_t *data)
{
    uint8_t n = cst820s_read_data(&touch_handle);
    if (n > 0) {
        cst820s_point_t pt = cst820s_get_point(&touch_handle, 0);
        touch_x = TFT_VER_RES - pt.x;  // 坐标旋转映射
        touch_y = TFT_HOR_RES - pt.y;
        touch_pressed = (pt.event == CST820S_EVENT_PRESS || pt.event == CST820S_EVENT_TOUCHING);
    }
    data->point.x = touch_x;
    data->point.y = touch_y;
    data->state = touch_pressed ? LV_INDEV_STATE_PRESSED : LV_INDEV_STATE_RELEASED;
}
```

### GC9C01 SPI 引脚配置

```c
// QSPI 4线模式，50MHz 时钟
#define TFT_DAT_D0  6
#define TFT_DAT_D1  7
#define TFT_DAT_D2  8
#define TFT_DAT_D3  9
#define TFT_CLK     5
#define TFT_CS      13
#define TFT_DC      12
#define TFT_RST     11
#define TFT_BLK     18  // 背光控制
```

### CST820S 触摸屏 I2C 配置

```c
#define I2C_MASTER_SCL_IO   15
#define I2C_MASTER_SDA_IO   14
#define CST820S_RST_PIN     16
#define CST820S_INT_PIN     17
#define CST820S_I2C_ADDR    0x15
#define I2C_MASTER_FREQ_HZ  100000  // 100kHz
```

## 构建/运行方法

### Docker 开发环境

```bash
# 拉取 ESP-IDF Docker 镜像
docker pull docker-hub.dahuatech.com/espressif/idf:latest

# 启动容器，挂载项目目录
docker run -it -v $PWD:/project -u $UID -e HOME=/tmp espressif/idf:latest
```

### 编译与烧录

```bash
# 设置目标芯片
idf.py set-target esp32s3

# 编译
idf.py build

# 烧录并监控串口
idf.py -p /dev/ttyUSB0 flash monitor
```

### 添加组件依赖

```bash
idf.py add-dependency "espp/st25dv^1.0.9"
idf.py create-manifest --component=my-component
```

### SDK 配置要点

- Flash 大小: 16MB
- 自定义分区表: `partitions.csv`
- 蓝牙: BLE 4.2 已启用
- USB: TinyUSB + CherryUSB 双栈
- LVGL: 32bit 色深 (ARGB8888)，Montserrat 14 字体
- MQTT: 协议版本 5.0
- OTA: 跳过 TLS 证书验证（开发阶段）

## 相关笔记链接

- [[ESP32-S3]] - ESP32-S3 芯片参考
- [[LVGL]] - LVGL GUI 框架笔记
- [[MQTT]] - MQTT 协议详解
- [[NFC-ST25DV]] - ST25DV NFC 双接口芯片
- [[L-ink-Card]] - 电子墨水屏 NFC 智能卡片
- [[TinyUSB]] - USB 设备协议栈
- [[ESP-IDF]] - ESP-IDF 开发框架
- [[FreeRTOS]] - FreeRTOS 实时操作系统
- [[OTA-升级]] - ESP32 OTA 固件升级
- [[GC9C01-LCD]] - GC9C01 圆形 TFT 驱动
- [[CST820S-触摸]] - CST820S 电容触摸控制器
- [[CherryUSB]] - CherryUSB 协议栈

## 相关笔记

- [[esp32-box-lite]] — ESP32-S3-BOX-Lite 开发板
- [[esp32c3]] — ESP32-C3 智能尾灯项目
- [[esp32s3-nfc]] — ESP32-S3 + ST25DV NFC 开发
- [[arduino]] — ESP32 墨水屏驱动项目
- [[esp-idf-v5-guide]] — ESP-IDF v5 开发指南
- [[lvgl-project]] — LVGL SquareLine Studio UI 项目
- [[AndroidStudio]] — Android App（同为 IoT 设备管理）
- [[weixinxiaoapp]] — 微信小程序 NFC 项目
