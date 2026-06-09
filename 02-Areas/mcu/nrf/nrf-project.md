---
title: nrf-project - Nordic Semiconductor Zephyr 应用集合
tags:
  - mcu
  - nrf
  - zephyr
  - ble
  - mqtt
  - iot
  - nrf5340
  - nrf91
category: mcu/nrf
created: 2026-06-09
status: active
---

# nrf-project

## 项目概述

本项目是基于 Nordic Semiconductor nRF 系列芯片和 Zephyr RTOS 的多个示例应用集合，涵盖 BLE HID 设备、MQTT 物联网传感器、AWS IoT Core 云连接以及 nRF5340 音频流等场景，是一个完整的 nRF 嵌入式开发学习与参考项目。

## 技术栈

- **芯片平台**: Nordic Semiconductor nRF52/nRF53/nRF54/nRF91 系列
- **操作系统**: Zephyr RTOS (基于 NCS - nRF Connect SDK)
- **构建系统**: CMake + West (Zephyr 构建工具)
- **配置系统**: Kconfig
- **通信协议**: BLE (Bluetooth Low Energy), MQTT, TLS 1.2, DHCPv4, SNTP
- **云平台**: AWS IoT Core, Eclipse Mosquitto (测试)
- **安全**: Mbed TLS, HID Service (HIDS), LE Secure Connections, NFC OOB Pairing
- **开发工具**: VS Code + nRF Connect 扩展, nrfutil CLI

## 项目结构

```
nrf-project/
├── .vscode/settings.json          # VS Code nRF Connect 扩展配置
├── nrfutil.exe                    # Nordic 命令行工具
├── app/                           # Blinky LED 示例 (GPIO 基础)
├── aws_iot_mqtt/                  # AWS IoT Core MQTT 客户端
├── mqtt/                          # 通用 MQTT 客户端 (nRF91/nRF70)
├── nrf5340_audio/                 # nRF5340 音频流 (LE Audio)
├── peripheral_hids_keyboard/      # BLE HID 键盘设备
└── secure_mqtt_sensor_actuator/   # 安全 MQTT 传感器/执行器
```

## 子应用详解

### 1. app/ - Blinky (GPIO 入门)

最基础的 Zephyr 示例，演示如何通过 Devicetree 获取 GPIO 引脚并控制 LED 闪烁。

**关键模式 - Devicetree GPIO 访问:**
```c
#define LED0_NODE DT_ALIAS(led0)
static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(LED0_NODE, gpios);

int main(void) {
    gpio_is_ready_dt(&led);
    gpio_pin_configure_dt(&led, GPIO_OUTPUT_ACTIVE);
    while (1) {
        gpio_pin_toggle_dt(&led);
        k_msleep(1000);
    }
}
```

### 2. aws_iot_mqtt/ - AWS IoT Core 连接

完整的 AWS IoT MQTT 客户端，支持 TLS 1.2 认证、SNTP 时间同步、QoS 消息传递和指数退避重连。

**核心特性:**
- DHCPv4 自动获取 IP
- SNTP 时间同步 (`0.pool.ntp.org`)
- TLS 1.2 + ALPN 连接 AWS IoT Core (端口 8883)
- 证书嵌入式编译 (通过 `convert_keys.py` 脚本转换)
- 支持 AWS Device Qualification Program (DQP) 测试

**关键 Kconfig 配置:**
```
CONFIG_AWS_ENDPOINT="a31gokdeokxhl8-ats.iot.eu-west-1.amazonaws.com"
CONFIG_AWS_MQTT_PORT=8883
CONFIG_AWS_THING_NAME="zephyr_sample"
CONFIG_MQTT_KEEPALIVE=60
CONFIG_MBEDTLS_HEAP_SIZE=65536
```

### 3. mqtt/ - 通用 MQTT 客户端

面向 nRF91 Series 和 nRF70 Series 的 MQTT 示例，支持多种板级配置和 TLS overlay。

**架构特点:**
- 使用 ZBus 消息总线 (`CONFIG_ZBUS=y`)
- 使用 Zephyr 状态机框架 (`CONFIG_SMF=y`)
- 使用 MQTT Helper 库 (`CONFIG_MQTT_HELPER=y`)
- 多 overlay 配置: native_sim, nrf70, nrf91 等

### 4. peripheral_hids_keyboard/ - BLE HID 键盘

完整的 BLE HID 键盘实现，支持 NFC OOB 配对，可作为电脑外接键盘使用。

**关键特性:**
- BLE HID GATT Service 暴露
- 标准键盘 Report Map (8 字节: modifier + reserved + 6 keys)
- LE Secure Connections 配对
- NFC Out-of-Band 配对 (可选)
- 多连接支持 (最多 2 个设备)
- Caps Lock 状态 LED 同步

**HID Report 格式:**
```c
// 按键报告格式: [modifier, reserved, Key1, Key2, Key3, Key4, Key5, Key6]
// 示例: "h" 按下 = 0x00 0x00 0x0B 0x00 0x00 0x00 0x00 0x00
// Shift+"l" = 0x02 0x00 0x0F 0x00 0x00 0x00 0x00 0x00
```

**BLE 连接回调模式:**
```c
BT_CONN_CB_DEFINE(conn_callbacks) = {
    .connected = connected,
    .disconnected = disconnected,
    .security_changed = security_changed,
};
```

**NFC OOB 配对流程:**
1. NFC 天线检测到设备 -> 触发 BLE 广播
2. 手机读取 NFC Tag 中的 OOB 数据
3. 使用 LESC (LE Secure Connections) 完成配对
4. 再次触碰 NFC 天线断开连接

### 5. secure_mqtt_sensor_actuator/ - 安全 MQTT 传感器

完整的 IoT 传感器/执行器设备实现，周期性发布温度数据并响应远程 LED 控制命令。

**架构设计:**
```
Network L4 Event -> Semaphore -> MQTT Connect -> Periodic Publish (k_work_delayable)
                                              -> Subscribe & Command Handler
```

**JSON 传感器数据格式:**
```c
static const struct json_obj_descr sensor_sample_descr[] = {
    JSON_OBJ_DESCR_PRIM(struct sensor_sample, unit, JSON_TOK_STRING),
    JSON_OBJ_DESCR_PRIM(struct sensor_sample, value, JSON_TOK_NUMBER),
};
// 输出示例: {"unit":"C","value":23}
```

**MQTT 客户端 ID 生成策略:**
```c
// 板名 + 随机十六进制后缀，确保唯一性
snprintk(client_id, sizeof(client_id), CONFIG_BOARD"_%x", (uint8_t)sys_rand32_get());
```

**MQTT 连接状态机:**
```c
void app_mqtt_run(struct mqtt_client *client) {
    app_mqtt_subscribe(client);
    while (mqtt_connected) {
        rc = app_mqtt_process(client);  // poll socket + keepalive
        if (rc != 0) break;
    }
    mqtt_disconnect(client, NULL);  // graceful disconnect
}
```

**远程命令支持:**
- `led_on` - 打开板载 LED
- `led_off` - 关闭板载 LED

### 6. nrf5340_audio/ - nRF5340 音频

LE Audio 音频流应用，支持多种角色:
- `broadcast_source/` - 广播源
- `broadcast_sink/` - 广播接收
- `unicast_client/` - 单播客户端
- `unicast_server/` - 单播服务器

支持 FOTA (Firmware Over-The-Air) 升级配置。

## 架构与设计决策

### 1. Zephyr Devicetree 驱动模型
所有硬件访问通过 Devicetree 宏 (`DT_ALIAS`, `GPIO_DT_SPEC_GET`) 实现，编译时绑定硬件配置，避免硬编码引脚号。

### 2. Kconfig 模块化配置
每个子项目通过 `prj.conf` 和 overlay 文件实现配置分离:
- `prj.conf` - 基础配置
- `overlay-tls-*.conf` - TLS 相关覆盖
- `overlay-static*.conf` - 网络配置覆盖

### 3. Sysbuild 多镜像构建
nRF5340 等多核芯片使用 `Kconfig.sysbuild` 配置系统构建 (IPC Radio 等)。

### 4. MQTT 客户端分层
- `mqtt_helper` (NCS 库) - 通用 MQTT 辅助
- `mqtt_client.c` (应用层) - 业务逻辑封装
- 通过 `k_work_delayable` 实现非阻塞周期发布

### 5. BLE 安全模型
- SMP (Security Manager Protocol) 配对
- HIDS 加密读写权限 (`CONFIG_BT_HIDS_DEFAULT_PERM_RW_ENCRYPT=y`)
- 禁用自动 PHY 更新和安全请求，避免连接兼容性问题

## 关键配置说明

| 配置项 | 说明 | 典型值 |
|--------|------|--------|
| `CONFIG_MQTT_LIB_TLS` | 启用 MQTT TLS | y |
| `CONFIG_MBEDTLS_HEAP_SIZE` | TLS 堆内存 | 60000-65536 |
| `CONFIG_BT_MAX_CONN` | BLE 最大连接数 | 2 |
| `CONFIG_BT_HIDS_MAX_CLIENT_COUNT` | HID 最大客户端 | 2 |
| `CONFIG_MAIN_STACK_SIZE` | 主线程栈大小 | 2048-4096 |
| `CONFIG_POSIX_API` | 启用 POSIX socket API | y |

## 构建与运行

### 环境准备
1. 安装 nRF Connect SDK (NCS)
2. 配置 West 构建工具
3. 安装 VS Code + nRF Connect 扩展

### 构建命令示例
```bash
# Blinky 示例
west build -b nrf52840dk_nrf52840 app/

# AWS IoT MQTT
west build -b nrf9160dk_nrf9160 aws_iot_mqtt/

# BLE HID 键盘 (带 NFC)
west build -b nrf52840dk_nrf52840 peripheral_hids_keyboard/ -- -DCONFIG_NFC_OOB_PAIRING=y

# 安全 MQTT 传感器 (静态 IP + 非加密)
west build -b adi_eval_adin1110ebz secure_mqtt_sensor_actuator/ -- -DOVERLAY_CONFIG="overlay-static-insecure.conf"
```

### 烧录
```bash
west flash
```

### AWS IoT 证书准备
```bash
cd aws_iot_mqtt/src/creds/
# 将 AWS IoT Core 下载的证书放入此目录
python convert_keys.py
# 生成 ca.c, cert.c, key.c
```

## 关键学习点

1. **Devicetree 是 Zephyr 硬件抽象的核心** - 所有外设访问应通过 DT 宏，而非直接操作寄存器地址
2. **Kconfig overlay 机制** - 实现同一代码库的多配置构建，适合不同硬件和安全需求
3. **MQTT + TLS 的内存开销** - MbedTLS 堆需要 60KB+，对 RAM 有限制的芯片需注意
4. **BLE HID 的 Report Map** - 需要严格遵循 USB HID 规范，Report 格式决定设备类型识别
5. **NFC OOB 配对** - 显著改善用户体验，避免手动输入配对码
6. **k_work_delayable 模式** - Zephyr 推荐的非阻塞周期任务实现方式

## 相关概念

- [[Zephyr RTOS]]
- [[Nordic nRF Connect SDK]]
- [[BLE HID Profile]]
- [[MQTT 协议]]
- [[AWS IoT Core]]
- [[Mbed TLS]]
- [[Devicetree]]
- [[Kconfig]]
- [[LE Audio]]
- [[NFC OOB Pairing]]

## 参考资源

- [Zephyr Project Documentation](https://docs.zephyrproject.org/)
- [nRF Connect SDK Documentation](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/latest/)
- [AWS IoT Core MQTT Guide](https://docs.aws.amazon.com/iot/latest/developerguide/mqtt.html)
- [Bluetooth HID Specification](https://www.bluetooth.com/specifications/specs/hid-service-1-0/)
- [Eclipse Mosquitto](https://mosquitto.org/)

## 相关笔记

- [[ncs]] — nRF Connect SDK (NCS)
- [[zephyr]] — Zephyr RTOS 项目笔记
- [[studyzephyr]] — Zephyr RTOS 学习项目
- [[redbook]] — 小红书 ESP32 MQTT 教程（同为 IoT MQTT 场景）
