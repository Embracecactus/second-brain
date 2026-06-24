---
tags:
  - mcu
  - project-idea
  - tinyml
  - esp32
  - ch32v003
  - csi
  - machine-learning
category: mcu/project-ideas
created: 2026-06-24
status: active
aliases:
  - 边缘AI项目创意
  - TinyML 非俗套玩法
---

# 方向 1: 边缘 AI / TinyML

> 廉价 MCU/带 NPU 的芯片上做本地推理, 避开 TFLite-Micro 关键词唤醒/姿势识别/振动检测等俗套

---

## 💡 项目 1: Wi-Fi CSI 人体感知 (无摄像头, 零传感器)

**脑洞:** 利用 ESP32-C6 的 WiFi CSI(信道状态信息)实现穿墙呼吸/心跳检测、人体存在、跌倒检测——完全不依赖摄像头/PIR/雷达。仅利用 WiFi 信号的反射变化, 就能感知室内人体微动。

**为什么没人做:** 学术圈论文很多(北大张大庆团队), 但落地到廉价 MCU 的开源端到端项目极少。乐鑫有 [[ESP-CSI]] 方案但大部分人不知道、不会用。2026 年开源的 RuView 把 ESP32-S3 硬件成本压到 ¥9。

**核验稀缺度:** ⭐⭐⭐ — 大学论文多但廉价 DIY 实现极少

**预算:** ESP32-C6 SuperMini **¥12** (无额外传感器)

**芯片推荐:** [[esp32-c6-p4-h2|ESP32-C6]] (WiFi6 CSI 更好) / ESP32-S3 (生态成熟)

**关键参考:**
- [乐鑫 ESP-CSI 文档](https://docs.espressif.com/projects/esp-techpedia/zh_CN/latest/esp-friends/solution-introduction/esp-csi/esp-csi-solution.html)
- [知乎 Wi-Fi CSI 教程](https://zhuanlan.zhihu.com/p/1975598662034953819)
- RuView 项目 (2026, ESP32-S3, ¥9 硬件成本)

---

## 💡 项目 2: CH32V003 上暴力 TinyML (2KB SRAM 极限分类器)

**脑洞:** 在最便宜的 32 位 MCU(¥0.58, 2KB SRAM)上跑微型 ML 分类器——不是 TFLite-Micro(跑不下), 而是手写极简决策树/二值化 SVM/Bayesian 分类器, 做单传感器(振动/温度/光)的特定事件识别。挑战在于 2KB SRAM 连个标准库函数都放不下, 必须手撸一切。

**为什么没人做:** 大部分人认为 2KB 不可能跑 ML, 但纯手工优化+资源极度受限的挑战本身就没人认真尝试过。就算有人试, 也是用 CH32V003 做灯控/电机, 没人用它跑算法。

**核验稀缺度:** ⭐⭐⭐ — 真没人做, 多数 TinyML 贴假

**预算:** CH32V003 开发板 + 传感器 ≈ **¥5** (芯片 ¥0.58 + 开发板 ¥4.29)

**芯片推荐:** [[wch|CH32V003]] (唯一目标)

---

## 💡 项目 3: ESP32-S3 低成本本地语音 → 控制台式机械臂 (无云端)

**脑洞:** S3 向量指令加速 KWS, 本地跑多分类语音命令(10-20 个词)控制一个桌面机械臂——全部本地推理、无云端依赖、¥100 内搞定。难点不在 KWS 本身(已经烂大街), 而在端到端的**连续识别 → 控制解码 → 轨迹规划**的闭环。

**核验:** ESP32-S3 做关键词唤醒已很多, 但结合机械臂控制的端到端开源项目极少。大部分语音机械臂依赖 PC/ROS/云端。

**核验稀缺度:** ⭐⭐

**预算:** ESP32-S3(¥12) + 舵机驱动板(¥30) + 微型机械臂(¥50) ≈ **¥92**

**芯片推荐:** [[esp32-c6-p4-h2|ESP32-S3]] (向量指令加速 + 摄像头接口备选)

---

## 相关链接
- [[new-chips-2024-2026]] — 芯片市场扫描
- [[MOC-嵌入式项目创意]] — 全部创意目录
- [[esp32-c6-p4-h2]] — ESP32 新芯片
- [[wch]] — 沁恒 RISC-V
