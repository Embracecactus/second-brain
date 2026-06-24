---
tags:
  - mcu
  - rp2350
  - pico2
  - raspberry-pi
  - risc-v
  - pio
category: mcu/rp2350
created: 2026-06-24
status: active
aliases:
  - RP2350
  - Pico 2
  - Pico 2 W
  - 树莓派 Pico 2
---

# RP2350 / Pi Pico 2 / Pico 2 W

> 树莓派 2024 年 8 月发布, 双核双架构 MCU, ¥26 起步

---

## 核心规格

- **双核异构:** Cortex-M33 (150MHz) + RISC-V hazard3 (150MHz) — 两个核可并行独立运行
- **SRAM:** 520KB (比 RP2040 的 264KB 翻倍)
- **Flash:** 4MB (板载)
- **独有外设:** **PIO** (可编程 I/O 状态机, 10 个独立状态机) — 可模拟 PWM/I2C/SPI/UART/DVI/WS2812 等任意数字时序
- **安全:** 安全启动 + OTP + SHA-256 加速
- **价格:** Pico 2 **¥26** (幸狐) / ¥47 (官版), Pico 2 W **¥35–47** (板载 CYW43439 WiFi4+BLE5)
- **工具链:** 官方 C/C++ SDK / MicroPython / CircuitPython / Arduino

---

## 与新芯片的差异化优势

| 特性 | RP2350 | ESP32-C6 | CH32V203 |
|---|---|---|---|
| 双架构异构 | ✅ M33+RISC-V | ❌ | ❌ |
| PIO 状态机 | ✅ 10 个 | ❌ | ❌ |
| 520KB SRAM | ✅ | ❌ (512KB) | ✅ (64KB) |
| 安全启动 | ✅ | ❌ | ❌ |
| 价格 | ¥26 | ¥8 | ¥2.65 |
| 无线 | 仅 Pico 2W 有 | ✅ 原生四模 | ❌ |

**RP2350 不可替代的场景:**
- 需要 PIO 模拟复杂数字时序(替代 CPLD/多路 PWM)
- 利用双核异构做分工(M33 跑应用、RV 跑底层 I/O)
- 对安全启动 / 固件防篡改有要求的产品原型

---

## 推荐玩的创新点

1. **PIO 驱动 8-12 路舵机** — 替代 PCA9685 或定时器, 精准度高且不占 CPU
   → [[03-robotics-motor]]
2. **异构双核 PDA** — M33 跑 UI + RISC-V 扫键盘/管理电源
   → [[04-retro-audio-toy]]
3. **PIO + DVI 视频输出** — 类似 RP2040 DVI 项目, 但 RP2350 的 SRAM 翻倍可做更复杂图形
4. **双核并行管道:** M33 从摄像头读取 → 共享 SRAM → RISC-V 处理 → PIO 输出显示

---

## 购买指南

| 产品 | 价格 | 含无线 | 推荐买谁 |
|---|---|---|---|
| Pico 2 (幸狐) | **¥26** | ❌ | 最划算, 本质同官版 |
| Pico 2 (树莓派官版) | ¥47 | ❌ | 官版, 带原厂 QC |
| Pico 2 W (微雪) | **¥35** | ✅ WiFi4+BLE5 | ¥35 含无线, 性价比 |
| RP2350-CAN (微雪) | ¥68 | ❌ | 带 CAN 接口 |

主力推荐: **幸狐 Pico 2 ¥26** 做纯 MCU 项目, 需要无线就 **Pico 2 W ¥35**

---

## 相关笔记
- [[MOC-MCU开发]] — MCU 目录
- [[MOC-嵌入式项目创意]] — 项目创意
- [[new-chips-2024-2026]] — 芯片市场扫描
- [[MOC-嵌入式项目创意]] → 03-robotics-motor / 04-retro-audio-toy
