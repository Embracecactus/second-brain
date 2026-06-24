---
tags:
  - mcu
  - project-idea
  - iot
  - wireless
  - lorawan
  - matter
  - thread
  - esp32
  - ch32x035
  - w801
category: mcu/project-ideas
created: 2026-06-24
status: active
aliases:
  - IoT无线项目创意
---

# 方向 2: IoT / 无线通信

> LoRa、Matter、Thread、私有 2.4G、低功耗传感器网络的非俗套玩法

---

## 💡 项目 1: CH32X035 可编程 USB-PD PPS 电源

**脑洞:** CH32X035 自带 USB-PD 3.0 控制器和 Type-C 物理层。用它做一个**可编程 USB-PD 电源**: 用 MCU 控制 PD 协商, 从笔记本 PD 充电器中诱骗出可调电压(5V/9V/12V/15V/20V + PPS 3.3-21V 可调)。可以接屏幕、编码器做调压旋钮, 甚至通过蓝牙远程调压。

**为什么别人没做:** 市面上 USB-PD 诱骗都用专用芯片(CH224K/IP2716)做固定电压, 没有人用自带 PD 的 MCU 做可编程的。因为 CH32X035 这种集成 PD + OPA + ADC + 触摸按键的单芯片 MCU 是第一款且在 2024 年刚出, 很多人还不知道。

**核验稀缺度:** ⭐⭐⭐ — 集成 PD 的 MCU 全国产独此一份

**预算:** CH32X035 ¥1.75 + PCB + 阻容 + OLED ≈ **¥22**

**芯片推荐:** [[wch|CH32X035]] (唯一选择, 因为集成 USB-PD PHY)

**关键参考:**
- [CH32X035 数据手册 (USB+PD MCU)](https://www.wch.cn/downloads/CH32X035DS0_PDF.html)
- [[wch]] — 沁恒 RISC-V 开发环境

---

## 💡 项目 2: 自建 ESP32-C6 Thread Border Router + BYO 传感器网

**脑洞:** 用 ¥16 的 C6 做 **Thread 边界路由器**(Thread Border Router), 自己写 OpenThread 固件 + Matter 协议对接 Home Assistant。市面上 Thread BR 只有 Apple TV/HomePod/Google Nest Hub, 自建的极少。再用 ¥19 的 **ESP32-H2** 做 Thread 传感器节点(温度/湿度/门磁), 形成一个完全自建的 Thread 网络。

**核验:** 社区有 ESP Thread BR 教程(Home Assistant 论坛), 但做成完整闭环 BYO 传感器网络的开源项目还非常少。ESPHome 2025.6 开始支持 OpenThread, 让这比以前容易很多。

**核验稀缺度:** ⭐⭐

**预算:** C6 ¥16 + H2 ¥19 + 传感器 ¥20 ≈ **¥55**

**芯片推荐:** [[esp32-c6-p4-h2|ESP32-C6]] (Thread BR) + [[esp32-c6-p4-h2|ESP32-H2]] (Thread 节点)

---

## 💡 项目 3: W801 ¥6 极致低价 WiFi+BT 传感器网关

**脑洞:** 用仅 ¥5.4 的 W801 做一个**¥10 级传感器网关**——采集多个蓝牙传感器的数据(温湿度/气压/AQI), 通过 WiFi 上传 MQTT/HTTP。比同功能的 ESP32 方案便宜一半。

**核验:** W801 性价比确实碾压同类, 但社区资源不足、烧录器需专用、文档基本是中文。这就是"没人玩但你 ¥5 就能碰"的选择——因为工具链劝退了大多数人。

**核验稀缺度:** ⭐⭐ — 技术性稀缺(不是没人想玩, 是玩得动的不多)

**预算:** W801 ¥5.4 + 传感器 ¥20 + 打板 ≈ **¥30**

**芯片推荐:** W801 (联盛德, QFN-56, 240MHz XT804 + DSP)

---

## 相关链接
- [[MOC-嵌入式项目创意]] — 全部创意目录
- [[new-chips-2024-2026]] — 芯片市场扫描
- [[esp32-c6-p4-h2]] — ESP32 新芯片(含 C6/H2)
- [[wch]] — 沁恒 RISC-V
