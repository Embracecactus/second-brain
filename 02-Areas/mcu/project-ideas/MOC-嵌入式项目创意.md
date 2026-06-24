---
tags:
  - moc
  - mcu
  - project-ideas
  - tinyml
  - iot
  - robotics
  - retro
created: 2026-06-24
updated: 2026-06-24
---

# 📚 嵌入式项目创意地图

> 2024-2026 年间 "别人还没做过 / 极少人做过" 的新颖嵌入式项目, 面向资深玩家、¥100 内预算。
> 稀缺度评级: ⭐⭐⭐ = 真少见 | ⭐⭐ = 有人做但不多 | ⭐ = 已有成熟开源

## 四方向概览

| 方向 | 文件名 | 核心稀缺芯片 | 最推荐项目 |
|---|---|---|---|
| 边缘 AI / TinyML | [[01-edge-ai-tinyml]] | ESP32-C6, CH32V003 | WiFi CSI 穿墙呼吸检测 (¥12) |
| IoT / 无线通信 | [[02-iot-wireless]] | CH32X035, ESP32-C6/H2 | 自建 Thread Border Router (¥55) |
| 机器人 / 运动控制 | [[03-robotics-motor]] | CH32V203, RP2350 | CH32V203 双路 SimpleFOC (¥72) |
| 复古 / 音频 / 玩具 | [[04-retro-audio-toy]] | CH32V003, RP2350, ESP32-P4 | V003 ¥5 迷你游戏机 (¥13) |

## 全创意速查

| 项目 | 方向 | 核心芯片 | 总成本 | 稀缺度 |
|---|---|---|---|---|
| Wi-Fi CSI 穿墙人体呼吸/心跳检测 | TinyML | ESP32-C6 | ¥12 | ⭐⭐⭐ |
| CH32V003 2KB SRAM 极限 ML 分类器 | TinyML | CH32V003 | ¥5 | ⭐⭐⭐ |
| 本地语音命令→桌面机械臂控制 | TinyML | ESP32-S3 | ¥92 | ⭐⭐ |
| CH32X035 可编程 USB-PD PPS 电源 | IoT | CH32X035 | ¥22 | ⭐⭐⭐ |
| 自建 ESP32-C6 Thread Border Router | IoT | ESP32-C6+H2 | ¥55 | ⭐⭐ |
| W801 ¥6 WiFi+BT 传感器网关 | IoT | W801 | ¥30 | ⭐⭐ |
| CH32V203 双路 SimpleFOC BLDC | 机器人 | CH32V203 | ¥72 | ⭐⭐⭐ |
| ESP32-C6 Wi-Fi 手势→实物控制 | 机器人 | ESP32-C6 | ¥26 | ⭐⭐⭐ |
| RP2350 PIO 双核异构微型四足 | 机器人 | RP2350 | ¥86 | ⭐⭐ |
| CH32V003 ¥5 OLED 迷你游戏机 | 复古 | CH32V003 | ¥13 | ⭐⭐⭐ |
| RP2350 异构双核 PDA 复刻 | 复古 | RP2350 | ¥66 | ⭐⭐ |
| ESP32-P4 口袋 MIPI 合成器 | 复古 | ESP32-P4 | ¥93 | ⭐⭐ |

## 芯片推荐优先级 (按项目数)

- **ESP32-C6** — 4 个项目可用 (CSI 感知 / Thread / Matter / 手势), 性价比最高
- **CH32V003** — 2 个项目 (极限 ML / 迷你游戏机), ¥0.58 的乐趣天花板
- **CH32X035** — 1 个项目 (PD 电源), 但芯片独一无二、集成 PD 全国产独此一份
- **RP2350** — 2 个项目 (PIO 四足 / PDA), 双架构异构 + PIO 是独有优势

---

## 相关笔记
- [[MOC-MCU开发]] — MCU 开发目录
- [[new-chips-2024-2026]] — 芯片市场扫描
- [[esp32-c6-p4-h2]] — ESP32 新芯片
- [[wch]] — 沁恒 RISC-V
- [[rp2350-pico2]] — RP2350 / Pico 2
