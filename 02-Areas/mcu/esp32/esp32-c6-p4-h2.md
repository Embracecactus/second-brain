---
tags:
  - mcu
  - esp32
  - esp32-c6
  - esp32-h2
  - esp32-p4
  - espressif
  - wifi6
  - thread
  - matter
category: mcu/esp32
created: 2026-06-24
status: active
aliases:
  - ESP32-C6
  - ESP32-H2
  - ESP32-P4
  - 乐鑫新芯片
---

# ESP32 新家族: C6 / H2 / P4

> 乐鑫 2024-2025 推出的新芯片, 覆盖 Thread/Matter/WiFi6/视觉新场景

---

## ESP32-C6 — WiFi6 + Thread + Zigbee 四模

- **核心:** RISC-V 32 位单核, 160MHz
- **无线:** WiFi 6 (802.11ax) + BLE 5.3 + Zigbee + Thread (802.15.4)
- **功耗:** 休眠 16μA, 适合电池供电
- **价格:** SuperMini 开发板 **¥7.7–16.5** (淘宝现货海量)
- **独有特性:** CSI(信道状态信息)可用于 Wi-Fi 人体感知(穿墙呼吸检测/手势识别)
- **工具链:** ESP-IDF / Arduino / MicroPython / ESPHome / ESP-Matter
- **可比:** 比 ESP32-S3 多了 Thread/Zigbee 和 WiFi6, 但无 S3 的向量指令和 PSRAM

**适合做:** Thread Border Router、Matter 传感器、WiFi CSI 人体感知、WiFi6 低功耗 IoT

[[01-edge-ai-tinyml]] | [[02-iot-wireless]]

---

## ESP32-H2 — 纯 802.15.4 低功耗节点

- **核心:** RISC-V 32 位单核, 96MHz
- **无线:** BLE 5.3 + Zigbee + Thread (**无 WiFi**)
- **价格:** 微雪 Zero 开发板 **¥19.19** (现货)
- **工具链:** 同 ESP-IDF, 支持 OpenThread
- **定位:** 与 C6 搭配: C6 做 Thread Border Router, H2 做低功耗 End Device
- **功耗:** 纯 802.15.4 模式下极低

**适合做:** Thread 传感器节点、Zigbee 端设备、低功耗 Mesh

[[02-iot-wireless]]

---

## ESP32-P4 — 首款 MPU 级 RISC-V(带 MIPI)

- **核心:** 双核 RISC-V HP (400MHz) + LP 核(40MHz), 性能 ≈ 入门级 MPU
- **新特性:**
  - **MIPI-CSI** (摄像头接口, 非 DVP)
  - **MIPI-DSI** (显示接口, 非 SPI/8080)
  - I2S 音频, USB 2.0 OTG, Ethernet
- **价格:** 核心板 **¥42.9** (启明云端) / 完整套件 ¥350+
- **工具链:** ESP-IDF (alpha), Arduino 逐步支持
- ⚠️ **注意:** P4 **本身无 WiFi/BT**, 需配合 ESP32-C6 模块一起使用
- **2025 年新出**, 芯片 ¥30-60 但核心板 ¥42.9 还在 ¥100 预算线内

**适合做:** MIPI 屏/LCD 驱动、摄像头本地视觉、口袋合成器、触控 HMI

[[04-retro-audio-toy]]

---

## 选型速查

| 芯片 | 淘宝价 | 无线能力 | AI 加速 | 最佳用途 |
|---|---|---|---|---|
| ESP32-C6 | ¥8–16 | WiFi6+BLE+Zigbee+Thread | — | 四模通信+CSI 感知 |
| ESP32-H2 | ¥19 | BLE+Zigbee+Thread | — | Thread 低功耗节点 |
| ESP32-P4 | ¥43 | 无(需配 C6) | 向量扩展 | MIPI 视觉/显示+音频 |
| ESP32-S3 | ¥12 | WiFi4+BLE | **有向量指令** | TinyML/KWS/视觉 |

## 相关笔记
- [[MOC-MCU开发]] — MCU 目录
- [[MOC-嵌入式项目创意]] — 项目创意
- [[new-chips-2024-2026]] — 芯片市场扫描
- [[esp32c3]] — ESP32-C3 项目
- [[arduino]] — ESP32 墨水屏驱动
