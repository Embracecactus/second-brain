---
tags:
  - esp32s3
  - nfc
  - st25dv
  - i2c
  - esp-idf
  - embedded
category: mcu/esp32
created: 2026-06-09
source: /mnt/c/Users/lijian/Downloads/esp32s3-st25dv-2/
---

# ESP32-S3 与 ST25DV NFC Tag 开发笔记

## 项目概述

本项目基于 **ESP32-S3** 微控制器，通过 **I2C** 总线驱动 **ST25DV04K** NFC/RFID 标签芯片，实现 NFC 数据读写功能。项目使用 **ESP-IDF v6.0.0** 框架，采用 C 语言开发，集成了 ST 官方 BSP 驱动和第三方开源库两种方案。

### 项目目录结构

```
esp32s3-st25dv-master/
├── CMakeLists.txt              # 顶层 CMake 配置
├── sdkconfig                   # ESP-IDF 项目配置
├── dependencies.lock           # 组件依赖锁文件
├── main/
│   ├── esp32-s3.c              # 主程序入口 app_main()
│   └── idf_component.yml       # main 组件依赖声明
├── components/
│   └── st25dv/                 # 自定义 ST25DV 驱动组件
│       ├── st25dv.c            # I2C 初始化及读写实现
│       ├── st25dv_reg.c        # 寄存器操作函数（已注释）
│       ├── nfctag_drv.c        # NFC Tag 驱动抽象层
│       └── include/
│           ├── st25dv.h
│           ├── st25dv_reg.h    # ST25DV 全部寄存器定义
│           └── nfctag_drv.h    # 驱动抽象接口定义
└── managed_components/         # IDF Component Manager 管理的组件
    ├── rjrp44__st25dv/         # 第三方 ST25DV 库 (v0.1.3)
    ├── espp__cli/              # CLI 组件
    └── espp__format/           # fmt 格式化库
```

---

## 关键知识点

### 1. ST25DV04K 芯片特性

ST25DV 是 STMicroelectronics 推出的 **双接口 NFC/RFID 标签芯片**，内含 EEPROM 存储器，同时支持 I2C 和 RF（NFC Forum Type 5）两种接口访问。

| 参数 | 值 |
|------|------|
| 型号 | ST25DV04K (4Kbit = 512 Bytes EEPROM) |
| 高端型号 | ST25DV64K (64Kbit) |
| ICref (04K) | `0x24` |
| ICref (64K) | `0x26` |
| RF 协议 | NFC Forum Type 5 (ISO 15693) |
| I2C 最大速率 | 1MHz |
| Mailbox 大小 | 256 Bytes |
| EEPROM 写超时 | 320ms (min: (256/4)*5) |
| 最大单次写入 | 256 Bytes |

### 2. ST25DV 双地址 I2C 架构

ST25DV 的核心设计特点是在同一 I2C 总线上使用 **两个不同的设备地址** 访问不同的寄存器空间：

| 地址类型 | 7-bit 地址 | 用途 |
|----------|-----------|------|
| Data (User) | `0x53` | 访问动态寄存器 (地址 >= `0x2000`) |
| System | `0x57` | 访问系统配置寄存器 (地址 < `0x2000`) |

> **关键判断逻辑**: 地址位 `0x2000` (`ST25DV_IS_DYNAMIC_REGISTER`) 用于区分路由到哪个 I2C 设备句柄。

### 3. 寄存器空间划分

**系统寄存器** (通过 System 地址 `0x57` 访问):

| 寄存器 | 地址 | 说明 |
|--------|------|------|
| `GPO_REG` | `0x0000` | GPO 配置 |
| `ITTIME_REG` | `0x0001` | 中断持续时间 |
| `EH_MODE_REG` | `0x0002` | 能量采集模式 |
| `RF_MNGT_REG` | `0x0003` | RF 管理 |
| `RFA1SS_REG` - `RFA4SS_REG` | `0x0004` - `0x000A` | Area 1-4 安全配置 |
| `ENDA1_REG` - `ENDA3_REG` | `0x0005` - `0x0009` | Area 1-3 结束地址 |
| `I2CSS_REG` | `0x000B` | I2C 安全配置 |
| `LOCKCCFILE_REG` | `0x000C` | CC File 锁定 |
| `MB_MODE_REG` | `0x000D` | Mailbox 模式 |
| `MB_WDG_REG` | `0x000E` | Mailbox 看门狗 |
| `LOCKCFG_REG` | `0x000F` | 配置锁定 |
| `DSFID_REG` | `0x0012` | DSFID |
| `AFI_REG` | `0x0013` | AFI |
| `MEM_SIZE_LSB/MSB_REG` | `0x0014`-`0x0015` | 内存大小 |
| `ICREF_REG` | `0x0017` | IC 参考标识 |
| `UID_REG` | `0x0018` | UID (8 字节) |
| `ICREV_REG` | `0x0020` | IC 版本 |
| `I2CPASSWD_REG` | `0x0900` | I2C 密码 (8 字节) |

**动态寄存器** (通过 Data 地址 `0x53` 访问):

| 寄存器 | 地址 | 说明 |
|--------|------|------|
| `GPO_DYN_REG` | `0x2000` | GPO 动态状态 |
| `EH_CTRL_DYN_REG` | `0x2002` | 能量采集控制 |
| `RF_MNGT_DYN_REG` | `0x2003` | RF 管理动态 |
| `I2C_SSO_DYN_REG` | `0x2004` | I2C 安全会话状态 |
| `ITSTS_DYN_REG` | `0x2005` | 中断状态 |
| `MB_CTRL_DYN_REG` | `0x2006` | Mailbox 控制 |
| `MBLEN_DYN_REG` | `0x2007` | Mailbox 消息长度 |
| `MAILBOX_RAM_REG` | `0x2008` | Mailbox 缓冲区 |

### 4. 驱动架构设计

项目采用 **两层驱动抽象** 设计:

**底层 - DRV_NfcDTag 接口** (`nfctag_drv.h`):

```c
typedef struct {
    esp_err_t (*Init)(void);
    esp_err_t (*DeInit)(void);
    esp_err_t (*ReadReg)(uint16_t Reg, uint8_t* pData, uint16_t len);
    esp_err_t (*WriteReg)(uint16_t Reg, const uint8_t* pData, uint16_t len);
} DRV_NfcDTag;
```

**上层 - 名称查找机制** (`nfctag_drv.c`):

通过 `CONFIG_DRV_ST25DV04` 宏和 `_getSensorOpByName()` 函数，实现基于 Kconfig 配置的驱动注册与查找。使用宏 `DRV_NFC_D_TAG_OP_DEFINE` / `DRV_NFC_D_TAG_OP_DECLARE` / `DRV_NFC_D_TAG_OP_ADDRESS` 实现编译时多态。

### 5. Kconfig 配置项

在 sdkconfig 中的关键配置:

```
CONFIG_DRV_ST25DV04="st25dv04"          # NFC 驱动名称
CONFIG_I2C_SCL_PIN=40                   # I2C SCL 引脚
CONFIG_I2C_SDA_PIN=41                   # I2C SDA 引脚
CONFIG_MAX_ST25DV_MAX_I2C_FREQ_HZ=100000  # I2C 频率 100KHz
```

---

## 技术细节

### I2C 总线初始化流程

1. 配置 `i2c_master_bus_config_t` 结构体，设置 SDA/SCL 引脚、时钟源、毛刺滤波、内部上拉
2. 调用 `i2c_new_master_bus()` 创建 I2C 主机总线
3. 分别以 `0x53` (Data) 和 `0x57` (System) 地址添加两个设备句柄
4. 使用 ESP-IDF v5.x+ 的新 I2C Master API (`driver/i2c_master.h`)

### 寄存器读写路由机制

`ReadRegWrap()` 和 `WriteRegWrap()` 函数通过检查寄存器地址的 `0x2000` 位来决定使用哪个 I2C 设备句柄:

- **地址 < `0x2000`**: 使用 `st25dv_syst_handle` (System 地址 `0x57`)
- **地址 >= `0x2000`**: 使用 `st25dv_data_handle` (Data 地址 `0x53`)

### EEPROM 写入轮询

写入系统寄存器（即 EEPROM 操作）后，需要轮询等待 EEPROM 就绪:

```c
do {
    pollstatus = CUSTOM_ST25DV_I2C_IsReady(st25dv_syst_handle, 1);
} while((tick_elapsed < ST25DV_WRITE_TIMEOUT) && (pollstatus != NFCTAG_OK));
```

超时时间为 320ms，通过 `xTaskGetTickCount()` 实现 FreeRTOS tick 计时。

### NFC Tag 初始化与 IC 版本读取

主程序 `app_main()` 的工作流程:
1. 初始化 NVS Flash
2. 延时 10 秒等待系统稳定
3. 调用 `NFCTAG_Init()` 初始化 NFC 驱动
4. 循环读取 `ICREV_REG` (`0x0020`) 并打印 IC 版本号

### 16-bit 寄存器地址处理

ST25DV 使用 16-bit 寄存器地址，I2C 传输时地址需拆分为 2 字节:

```c
uint8_t reg_addr[2] = {(uint8_t)(Reg >> 8), (uint8_t)(Reg & 0xFF)};
```

写入时将地址和数据合并为单次传输:

```c
uint8_t write_buf[2 + Length];
memcpy(write_buf, &Reg, 2);
memcpy(write_buf + 2, pData, Length);
```

---

## 代码/配置片段

### 顶层 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.16)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(esp32-s3)
# set(CMAKE_CXX_STANDARD 20)  # 如需 C++20 支持，取消注释
```

### main/idf_component.yml (依赖声明)

```yaml
dependencies:
  idf:
    version: ">=4.1.0"
  # 已注释的可选依赖:
  # espp/st25dv: ">=1.0.9"
  # espp/i2c: ">=1.0.9"
  # rjrp44/st25dv: ">=0.1.3"
  # espressif/cjson: "~1.7.0"
```

### Docker 开发环境

```bash
# 拉取 ESP-IDF Docker 镜像
docker pull docker-hub.dahuatech.com/espressif/idf:latest

# 运行容器，挂载项目目录
docker run -it \
  -v $PWD:/project \
  -u $UID \
  -e HOME=/tmp \
  docker.1ms.run/espressif/idf:latest
```

### 创建项目与添加依赖

```bash
# 新建工程
idf.py create-project esp32-s3
cd esp32-s3/
idf.py set-target esp32s3

# 添加 ST25DV 组件依赖
idf.py add-dependency "espp/st25dv^1.0.9"
# 或使用 rjrp44 版本
idf.py add-dependency "rjrp44/st25dv"
```

### SDK 配置关键参数 (sdkconfig)

```
# 目标芯片
CONFIG_IDF_TARGET="esp32s3"
CONFIG_IDF_TARGET_ESP32S3=y
CONFIG_IDF_INIT_VERSION="6.0.0"

# CPU 频率
CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ_160=y
CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ=160

# Flash 配置
CONFIG_ESPTOOLPY_FLASHSIZE="2MB"
CONFIG_ESPTOOLPY_FLASHFREQ_80M=y
CONFIG_ESPTOOLPY_FLASHMODE_DIO=y

# I2C 配置
CONFIG_I2C_MASTER_ISR_HANDLER_IN_IRAM=y

# ST25DV 专用配置
CONFIG_DRV_ST25DV04="st25dv04"
CONFIG_I2C_SCL_PIN=40
CONFIG_I2C_SDA_PIN=41
CONFIG_MAX_ST25DV_MAX_I2C_FREQ_HZ=100000

# FreeRTOS
CONFIG_FREERTOS_HZ=100
CONFIG_FREERTOS_NUMBER_OF_CORES=2
```

### ST25DV 状态码定义

```c
#define NFCTAG_OK      (0)
#define NFCTAG_ERROR   (-1)
#define NFCTAG_BUSY    (-2)
#define NFCTAG_TIMEOUT (-3)
#define NFCTAG_NACK    (-102)
```

### 第三方库 rjrp44/st25dv 提供的 NDEF 功能

```c
// NDEF 记录类型
#define ST25DV_TYPE5_NDEF_MESSAGE  0x03
#define ST25DV_TYPE5_TERMINATOR_TLV 0xFE

// TNF (Type Name Format) 值
#define NDEF_ST25DV_TNF_WELL_KNOWN  0x01
#define NDEF_ST25DV_TNF_MIME        0x02
#define NDEF_ST25DV_TNF_EXTERNAL    0x04

// NDEF 写入 API
esp_err_t st25dv_ndef_write_ccfile(uint64_t ccfile);
esp_err_t st25dv_ndef_write_content(st25dv_config, uint16_t *address, bool mb, bool me, std25dv_ndef_record record);
esp_err_t st25dv_ndef_write_app_launcher_record(st25dv_config, uint16_t *address, bool mb, bool me, char *app_package);
esp_err_t st25dv_ndef_write_json_record(st25dv_config, uint16_t *address, bool mb, bool me, cJSON *json_data);
```

### 第三方库 st25dv_config 结构体

```c
typedef struct {
    uint8_t user_address;    // 0x53
    uint8_t system_address;  // 0x57
} st25dv_config;
```

---

## ESP32-S3 硬件特性 (从 sdkconfig 提取)

| 特性 | 配置值 |
|------|--------|
| CPU 架构 | Xtensa 双核 (LX7) |
| CPU 频率 | 160 MHz |
| FPU | 支持 |
| Flash | 2MB, DIO 模式, 80MHz |
| I2C 接口数 | 2 个 |
| I2C FIFO | 32 字节 |
| I2C 支持 | 10-bit 地址, Slave 模式, HW 清除总线 |
| GPIO 数量 | 49 个 |
| UART | 3 个, 最大 5Mbps |
| SPI | 3 个, 支持 OPI |
| WiFi | 支持, WPA3-SAE |
| BLE | 支持, BLE 5.0 |
| USB OTG | 支持 |
| PSRAM | 支持 (未启用) |
| Deep Sleep | 支持 |
| 指令 Cache | 16KB, 8-way |
| 数据 Cache | 32KB, 8-way |
| Brownout 检测 | Level 7 |

---

## 相关链接

- **ST25DV 官方文档**: STMicroelectronics ST25DV04K / ST25DV64K Datasheet
- **rjrp44 ST25DV 库**: https://github.com/RJRP44/ST25DV-Library (MIT License, v0.1.3)
- **ESP-IDF 编程指南**: https://docs.espressif.com/projects/esp-idf/
- **ESP-IDF I2C Master API**: `driver/i2c_master.h` (v5.x+ 新 API)
- **NFC Forum Type 5 Tag**: ST25DV 符合 NFC Forum Type 5 (ISO 15693) 规范
- **espp/st25dv 组件**: ESP-IDF Component Registry 上的 espp 封装版本 (>=1.0.9)
- **fmt 库**: https://github.com/fmtlib/fmt (项目中通过 espp__format 组件引用)

---

## 备注与待办

- `st25dv_reg.c` 中的寄存器操作函数全部被注释，当前仅实现了底层 I2C 读写和 `ReadRegWrap`/`WriteRegWrap` 路由
- 项目中尝试过两种第三方 ST25DV 库 (`espp/st25dv` 和 `rjrp44/st25dv`)，当前均被注释，使用自定义 `components/st25dv` 驱动
- `main/idf_component.yml` 中 `cJSON` 依赖被注释，如需 NDEF JSON 功能需启用
- 主程序当前仅实现了循环读取 IC 版本号的 demo 功能
- C++20 支持 (`set(CMAKE_CXX_STANDARD 20)`) 在 CMakeLists.txt 中被注释，如需使用 fmt 等 C++ 库需启用
