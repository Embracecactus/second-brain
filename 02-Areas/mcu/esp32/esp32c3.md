---
tags:
  - mcu
  - esp32
  - esp32c3
  - ble-mesh
  - iot
  - embedded
  - ws2812
  - imu
  - brake-detection
  - tflite
category: mcu/esp32
created: 2026-06-09
status: active
aliases:
  - ESP32-C3 智能尾灯项目
---

# ESP32-C3 智能尾灯项目

## 项目概述

基于 ESP32-C3 的 BLE Mesh 智能自行车尾灯系统，集成 IMU 刹车检测、WS2812 RGB LED 灯效控制、BLE 无线配置和电池电量管理。项目包含两个子工程：`esp32-c3`（主固件，ESP-IDF 框架）和 `esp-skainet`（乐鑫语音识别 SDK，备用参考）。

## 技术栈

- **MCU**: ESP32-C3（RISC-V 单核，160MHz）
- **Framework**: ESP-IDF v5.1 / v6.0
- **RTOS**: FreeRTOS（含 trace facility 和 runtime stats）
- **蓝牙协议栈**: NimBLE（BLE-only 模式）
- **BLE Mesh**: ESP BLE Mesh（Node 角色，Generic Server）
- **LED 驱动**: WS2812（30 颗 LED，GPIO10，RMT 通道 0）
- **IMU 传感器**: QMI8658C（I2C，地址 0x6B，加速度计 + 陀螺仪）
- **ML 推理**: ESP-TFLite-Micro（TensorFlow Lite Micro）
- **存储**: NVS Flash（持久化用户设置）
- **电源管理**: ADC 电池电压检测（GPIO2，分压比 1.63）
- **OTA**: 双分区 OTA 更新（自定义分区表，1500K x 2 app 分区）
- **通信**: MQTT 5.0（已预留代码）、BLE Server（已实现）
- **构建系统**: CMake + ESP-IDF Component Manager
- **语言**: C / C++（.cc 文件用于 C++ 组件）

## 架构与关键设计决策

### 模块化 app 组件架构

项目将功能拆分为独立的 app 组件，通过 `EXTRA_COMPONENT_DIRS` 注入构建系统：

```
app/
├── app_ws2812.c       # WS2812 LED 灯效引擎
├── app_ble_mesh.c     # BLE Mesh 网络栈
├── app_ble_server.c   # BLE GATT Server（NimBLE）
├── app_battery.c      # ADC 电池电量检测
├── app_qmi8658.cc     # IMU 传感器驱动（C++）
├── app_setting.c      # NVS 持久化设置管理
├── app_button.c       # 按键处理
├── app_lowpower.c     # 低功耗管理
└── modle.cc           # 模型相关（TFLite 推理）
```

### 双固件版本：ESP-IDF 与 Arduino

项目同时存在两套实现：

1. **esp32-c3/main/esp32-c3.c** — ESP-IDF 原生实现，使用 FreeRTOS API、NimBLE、NVS 等
2. **esp32-c3/main/text.c** — Arduino 框架实现（含 `#include <Arduino.h>`），功能完整但代码更紧凑

Arduino 版本（text.c）包含了完整的业务逻辑原型：BLE 配置、IMU 刹车检测、LED 动画状态机。

### 刹车检测算法

采用**双通道移动平均滤波**检测刹车减速事件：

- **快速通道**（FAST_SIZE=4）：捕捉短期加速度变化
- **慢速通道**（SLOW_SIZE=40）：维护长期基线
- **判定条件**：delta = fastAvg - slowAvg，当 delta <= -0.6g 且连续 3 次满足时触发
- **防抖锁定**：触发后 1000ms 内不再重复触发
- **刹车灯效果**：红色闪烁 6 次（100ms 亮 / 100ms 灭）

### BLE Mesh + BLE Server 双模

主固件同时支持：
- **BLE Mesh**：作为 Mesh Node 加入照明网络，支持 Generic Server 模型
- **BLE Server**：独立 GATT 服务，用于手机 App 配置灯效参数

### 低功耗策略

通过 Kconfig 可配置：
- Modem Sleep 模式（BLE 控制器休眠）
- 动态频率调节（最低 10MHz，最高 160MHz）
- Light Sleep 模式（自动/手动进入）
- MAC 和基带断电（`ESP_PHY_MAC_BB_PD`）

## 关键代码片段

### 系统启动流程

```c
void app_main(void) {
    vTaskDelay(pdMS_TO_TICKS(5000));  // 等待电源稳定
    appSettingInit();                  // NVS 初始化
    appSettingLoad();                  // 加载持久化设置
    appSetCurrentMode(0);
    app_ws2812_init();                 // LED 初始化
    adc_init();                        // 电池 ADC 初始化
    app_qmi8658_init();               // IMU 初始化
    app_ble_mesh_init();              // BLE Mesh 初始化
    app_button();                      // 按键任务启动
    // 主循环：打印 FreeRTOS 任务状态（调试用）
}
```

### FreeRTOS 任务监控模式

主循环中实现了任务状态打印，用于调试堆栈使用情况：

```c
ArraySize = uxTaskGetNumberOfTasks();
StatusArray = pvPortMalloc(ArraySize * sizeof(TaskStatus_t));
uxTaskGetSystemState(StatusArray, ArraySize, &TotalRunTime);
// 打印每个任务的名称、优先级、堆栈高水位标记
```

### BLE 连接回调中的数据同步

```c
void onConnect(BLEServer* pServer) {
    uint8_t data[5] = { currentMode, brightness, speed, colorId, battery };
    pCharacteristic->setValue(data, 5);
    pCharacteristic->notify();
}
```

## 构建与运行

### Docker 环境（推荐）

```bash
# 拉取 ESP-IDF Docker 镜像
docker pull docker.1ms.run/espressif/idf:release-v5.1

# 启动容器
docker run -it -v $PWD:/project -u $UID -e HOME=/tmp \
  docker.1ms.run/espressif/idf:latest

# 设置目标芯片
cd esp32-c3
idf.py set-target esp32c3

# 编译烧录
idf.py flash monitor

# 菜单配置
idf.py menuconfig
```

### 分区表配置

```
nvs,      data, nvs,     ,  0x4000,
otadata,  data, ota,     ,  0x2000,
phy_init, data, phy,     ,  0x1000,
ota_0,    app,  ota_0,   ,  1500K,
ota_1,    app,  ota_1,   ,  1500K,
```

### 关键 sdkconfig 配置

| 配置项 | 值 | 说明 |
|---|---|---|
| `CONFIG_IDF_TARGET` | esp32c3 | 目标芯片 |
| `CONFIG_BT_NIMBLE_ENABLED` | y | 使用 NimBLE 协议栈 |
| `CONFIG_BLE_MESH_NODE` | y | Mesh Node 角色 |
| `CONFIG_BT_NIMBLE_MAX_CONNECTIONS` | 2 | 最大 BLE 连接数 |
| `CONFIG_ESP_HTTPS_OTA_ALLOW_HTTP` | y | 允许 HTTP OTA（开发用） |
| `CONFIG_COMPILER_OPTIMIZATION_SIZE` | y | 编译优化体积 |
| `CONFIG_APP_PROJECT_VER` | v1.0.3 | 固件版本 |
| `CONFIG_FREERTOS_GENERATE_RUN_TIME_STATS` | y | 运行时统计 |

## ML/深度学习流水线（tools/）

项目包含 IMU 数据的机器学习训练和部署流程：

```
训练环境（Python venv）:
  torch → ONNX → onnx2tf → TensorFlow Lite → ESP-TFLite-Micro

关键文件:
  tools/tensflow_train_imu.py     # TensorFlow 训练脚本
  tools/convert.py                # 模型格式转换
  tools/imu_model_quant.tflite    # 量化后的 TFLite 模型
  tools/model_quantized_int8.tflite  # INT8 量化模型
  tools/model.cc                  # C++ 模型数据（嵌入式部署用）
```

## 依赖组件

通过 `idf_component.yml` 管理：

| 组件 | 版本 | 用途 |
|---|---|---|
| espressif/led_strip | ^3.0.1 | WS2812 驱动 |
| waveshare/qmi8658 | ^1.0.1 | IMU 传感器驱动 |
| espressif/button | ^4.1.4 | 按键消抖库 |
| espressif/esp-tflite-micro | ^1.3.4 | TFLite 推理引擎 |

## 子项目：esp-skainet

ESP-Skainet 是乐鑫的智能语音助手 SDK，支持：
- **WakeNet**：唤醒词检测（"Hi ESP"、"你好小智" 等）
- **MultiNet**：离线语音命令识别（最多 200 个中英文命令词）
- **AFE**：声学前端（AEC、VAD、BSS、NS）

推荐在 ESP32-S3 上运行（支持 AI 指令和八线 SPI PSRAM），当前项目中作为参考库引入。

## 关键经验与注意事项

1. **启动延迟**：`app_main` 开头的 5 秒延迟是为了等待电源稳定，硬件设计相关
2. **C/C++ 混编**：IMU 驱动和 TFLite 使用 .cc 文件，需要 `extern "C"` 包裹头文件声明
3. **BLE 功率控制**：通过 `esp_ble_tx_power_set` 设置广播和扫描功率为 0dBm
4. **NVS 持久化**：灯效模式、亮度、速度、颜色等用户设置通过 NVS 存储，断电保持
5. **日志优化**：生产固件禁用了大部分日志（`CONFIG_LOG_DEFAULT_LEVEL_NONE`），减小体积
6. **MINIMAL_BUILD 警告**：注释中提到启用 `MINIMAL_BUILD` 会导致 `CONFIG_ESP_HTTPS_OTA_ALLOW_HTTP` 不生效
7. **OTA 安全**：当前配置跳过了 TLS 证书验证（`CONFIG_ESP_TLS_SKIP_SERVER_CERT_VERIFY`），仅适用于开发阶段

## 相关链接

- [[ESP-IDF]] — Espressif IoT 开发框架
- [[BLE Mesh]] — 蓝牙 Mesh 网络协议
- [[QMI8658]] — 6 轴 IMU 传感器
- [[WS2812]] — 可编程 RGB LED 灯带
- [[TensorFlow Lite Micro]] — 嵌入式 ML 推理引擎
- [[FreeRTOS]] — 实时操作系统
- [ESP-IDF 官方文档](https://docs.espressif.com/projects/esp-idf/)
- [ESP-Skainet GitHub](https://github.com/espressif/esp-skainet)
- [ESP BLE Mesh 文档](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/api-guides/ble-mesh/)

## 相关笔记

- [[esp32-box-lite]] — ESP32-S3-BOX-Lite 开发板
- [[esp32s3-nfc]] — ESP32-S3 + ST25DV NFC 开发
- [[brithday]] — ESP32 生日项目合集
- [[esp-idf-v5-guide]] — ESP-IDF v5 开发指南
- [[arduino]] — ESP32 墨水屏驱动项目
- [[redbook]] — 小红书 ESP32 MQTT 教程
