---
tags: [moc, embedded-linux, career, dji, camera]
created: 2026-06-09
updated: 2026-06-23
target: 大疆相机驱动/嵌入式开发工程师
strategy: BSP基础 → 相机进阶 两段式路线
---

# 嵌入式Linux BSP → 相机驱动 学习路线

> **目标岗位**：大疆相机驱动/嵌入式开发
> **目标平台**：RV1126B (Cortex-A53 ×4, ISP V35, VPSS V20, 2×MIPI DPHY, 4×CSI-2)
> **方法**：先打通用 BSP 基础 → 再深入相机管线

---

## 路线总览

```
┌──────────────── 第一段：BSP 基础 ────────────────┐
│                                                   │
│  阶段一  Bootloader + 启动流程                     │
│  阶段二  设备模型 + 设备树                         │
│  阶段三  中断处理 + 并发                           │
│  阶段四  外设驱动 I2C/SPI/UART                     │
│  阶段五  电源管理                                  │
│  阶段六  Capstone BSP 驱动                         │
│                                                   │
│  → 掌握通用 BSP 开发能力, 满足 JD 基本要求          │
│                                                   │
├──────────── 第二段：相机驱动进阶 ────────────────┤
│                                                   │
│  阶段七  MIPI CSI 协议 + Camera Sensor 驱动        │
│  阶段八  V4L2 框架 + ISP Pipeline + Media Ctrl     │
│  阶段九  ISP 3A 算法 + RKAIQ 集成                  │
│  阶段十  MPP 硬件编解码管线                        │
│  阶段十一 RGA 2D加速 + 图像后处理                  │
│  阶段十二 相机 Capstone — 完整相机管线项目          │
│                                                   │
│  → 掌握相机驱动全栈, 对标大疆相机团队               │
│                                                   │
└───────────────────────────────────────────────────┘
```

---

## JD 要求 ↔ 阶段映射

| JD 要求 | 第一段 | 第二段 |
|---------|--------|--------|
| 嵌入式Linux系统架构 | ✅ 一~六 | — |
| Bootloader/Kernel移植 | ✅ 一 | — |
| C语言 + 调试 | ✅ 全程 | ✅ 全程 |
| UART/I2C/SPI协议 | ✅ 四 | ✅ 七 (Sensor I2C) |
| USB协议 | ✅ 四 | — |
| Linux设备模型 | ✅ 二 | ✅ 八 (Media Entity) |
| 中断处理 | ✅ 三 | ✅ 七 (VSYNC/帧中断) |
| 电源管理 | ✅ 五 | ✅ 九 (Sensor PM) |
| **V4L2 框架** | — | ✅ 八 |
| **MIPI CSI 协议** | — | ✅ 七 |
| **ISP Pipeline** | — | ✅ 八~九 |
| **硬件编解码** | — | ✅ 十 |
| **图像处理** | — | ✅ 十~十一 |
| 驱动设计编码测试 | ✅ 六 | ✅ 十二 |
| 技术文档 | ✅ 全程 | ✅ 全程 |

---

## 第一段：BSP 基础 (阶段一~六)

> 通用 BSP 能力，满足 JD 基本要求。这部分与大疆通用 BSP 岗位完全一致。

### 阶段一：Bootloader + 系统启动流程
> **文档**：[[bsp-boot-flow]]
> **大疆相机关联**：A/B 双分区 OTA (相机固件升级)、bootargs 传递
- Boot chain：ROM → DDR → SPL → ATF → U-Boot → Kernel
- FIT 镜像、分区表、bootargs、启动时间优化

### 阶段二：Linux 设备模型 + 设备树
> **文档**：[[bsp-device-model-dtb]]
> **大疆相机关联**：相机管线全靠设备树链接 (sensor→DPHY→CSI→CIF→ISP→VPSS)
- Bus/Device/Driver、DTS 语法、platform driver、字符设备

### 阶段三：中断处理 + 并发
> **文档**：[[bsp-interrupt-concurrency]]
> **大疆相机关联**：Sensor VSYNC 中断、ISP 帧完成中断、DMA 中断
- GIC-400、request_threaded_irq、workqueue、spinlock/mutex

### 阶段四：外设驱动 — I2C + SPI + UART
> **文档**：[[bsp-peripheral-drivers]]
> **大疆相机关联**：**Camera Sensor 通过 I2C 配置** (寄存器读写)、SPI 用于部分传感器
- I2C/SPI/UART 子系统、i2c_smbus_read_byte_data、Ftrace 追踪

### 阶段五：电源管理
> **文档**：[[bsp-power-management]]
> **大疆相机关联**：Sensor PM (standby/powerdown)、ISP 时钟门控、温控降频
- Clock (CCF)、Regulator (RK801 PMIC)、Runtime PM、PM Domain

### 阶段六：Capstone — 完整 BSP 驱动项目
> **文档**：[[bsp-capstone-driver]]
> **大疆相机关联**：I2C 驱动 = Camera Sensor 驱动的基础
- I2C + IRQ + Runtime PM + sysfs + Lockdep 验证

---

## 第二段：相机驱动进阶 (阶段七~十二)

> 从通用 BSP 转向相机专项。这部分的深度直接决定能否通过大疆相机岗位面试。

### 阶段七：MIPI CSI 协议 + Camera Sensor 驱动
> **文档**：[[bsp-csi-sensor-driver]]
> **大疆相机核心**：大疆所有相机产品的起点 — Sensor 驱动

- MIPI CSI-2 协议：D-PHY、C-PHY、Lane 配置、Virtual Channel
- RV1126B 相机接口：2×DPHY + 4×CSI-2 Host + 1×DVP
- Camera Sensor 驱动结构：`v4l2_subdev_ops` (core/video/pad)
- 寄存器初始化序列、分辨率/帧率切换、曝光/增益控制
- DTS 管线配置：sensor → dphy → csi2 → cif 链路
- 实战：分析 SC450AI / IMX415 驱动，编写虚拟 sensor 驱动

**RV1126B 支持 Sensor**：SC200AI, SC4336, SC450AI, SC850SL, IMX335, IMX415, IMX464, IMX586, OV4689, OV13850, GC8034 等

### 阶段八：V4L2 框架 + ISP Pipeline + Media Controller
> **文档**：[[bsp-v4l2-isp-pipeline]]
> **大疆相机核心**：理解相机数据如何从 Sensor 流到内存

- V4L2 核心框架：video_device / vb2_queue / v4l2_subdev
- Media Controller：media_entity / media_pad / media_link
- RV1126B ISP V35 管线：Sensor → DPHY → CSI-2 → RKCIF → SDITF → RKISP → VPSS
- RKCIF 驱动：4×MIPI Stream、DMA、online-to-ISP 模式
- RKISP 驱动：capture / stats / params 三个 video 节点
- `media-ctl` + `v4l2-ctl` 配置管线
- 实战：用 media-ctl 画出 RV1126B 相机拓扑图，手动配置管线采集帧

### 阶段九：ISP 3A 算法 + RKAIQ 集成
> **文档**：[[bsp-isp-3a-rkaiq]]
> **大疆相机核心**：3A 是相机画质的核心竞争力

- ISP 处理流水线：BLC → LSC → AWB → Demosaic → NR → Sharpen → CCS → Gamma
- 3A 算法：AE (自动曝光) / AWB (自动白平衡) / AF (自动对焦)
- RKAIQ 架构：3A Server + 65+ IQ 算法模块 + IQ Tuning 文件 (JSON)
- ISP stats/params 节点：统计信息上报 + 参数下发 (每帧)
- IQ 调校文件：`iqfiles/isp35/*.json`
- 实战：运行 rkaiq_3A_server，调参观察画质变化，分析 stats 数据流

### 阶段十：MPP 硬件编解码管线
> **文档**：[[bsp-mpp-codec-pipeline]]
> **大疆相机核心**：相机录像/图传的核心 — 硬件 H.265 编码

- MPP 架构：MPI 接口 + mpp_service 内核驱动 + VEPU511 硬件
- Rockit 框架：RK_MPI_VI → VENC → VO 管线 (rkipc 使用)
- 编码流程：mpp_create → mpp_init → encode_put_frame → encode_get_packet
- H.265 vs H.264：码率、画质、延迟对比
- 实战：用 MPP 编码一帧 YUV → H.265，对比 CPU 软编性能

### 阶段十一：RGA 2D 加速 + 图像后处理
> **文档**：[[bsp-rga-postprocess]]
> **大疆相机核心**：预览缩放、OSD 叠加、EIS 防抖校正

- RGA 硬件：缩放、旋转、格式转换、裁剪、blending
- librga API：im2d (im_resize / imcopy / imcvtcolor)
- VPSS V20：视频后处理子系统 (ISP 后的缩放/格式转换)
- EIS 电子防抖：GDC 几何畸变校正
- OSD 叠加：时间戳水印、Logo
- 实战：用 RGA 做 NV12→RGB 转换 + 缩放，对比 CPU 性能

### 阶段十二：相机 Capstone — 完整相机管线项目
> **文档**：[[bsp-camera-capstone]]
> **大疆相机核心**：综合项目，面试展示用

- 从零搭建：Sensor 驱动 → ISP 配置 → 3A 启动 → MPP 编码 → 文件保存
- 完整流程：media-ctl 配管线 → V4L2 采集 → RKAIQ 3A → MPP H.265 编码 → mp4 封装
- 多路管线：主码流 (4K H.265) + 预览流 (1080p) + 缩略图 (JPEG)
- 性能优化：帧率、延迟、功耗测量
- 设计文档 + 代码 review

---

## RV1126B 相机硬件资源速查

| 资源 | 数量/型号 | 驱动文件 | 说明 |
|------|----------|---------|------|
| ISP | V35 | `isp/` (rkisp.c) | 图像信号处理 |
| VPSS | V20 | `vpss/` | 视频后处理 |
| RKCIF | 1×DVP + 4×MIPI | `cif/` | 相机输入接口 |
| MIPI CSI-2 Host | 4 路 | `cif/mipi-csi2.c` | CSI-2 接收 |
| CSI-2 DPHY | 2 物理 (6 虚拟) | `phy-rockchip-csi2-dphy.c` | MIPI 物理层 |
| 编码器 | VEPU511 | `mpp_rkvenc.c` | H.264/H.265/JPEG |
| 解码器 | VDPU384A | `mpp_rkvdec.c` | H.264/H.265/JPEG |
| RGA | RGA2 | `rga2/` | 2D 图形加速 |
| NPU | 2.0 TOPS | `rknpu` | AI 推理 (可用于AI ISP) |

### 相机管线拓扑

```
[Sensor I2C] → [csi2_dphy] → [mipi_csi2] → [rkcif]
                                              ↓
                              ┌── CAPTURE: DMA → DDR (V4L2 video)
                              └── TOISP: SDITF → rkisp_vir (ISP V35)
                                                        ↓
                                                  rkisp_sditf
                                                        ↓
                                                  rkvpss_vir (VPSS)
                                                        ↓
                                          Rockit (VI → VENC → VO)
                                                        ↑
                                  RKAIQ 3A (stats/params per frame)
```

---

## 已有笔记索引

### 第一段：BSP 基础文档
- [[bsp-boot-flow]] — 阶段一
- [[bsp-spl-fit]] — SPL FIT 解析与验证 (阶段一深度)
- [[bsp-uboot-adaptation]] — U-Boot 板级适配与启动流程 (阶段一移植)
- [[bsp-uboot-secureboot]] — U-Boot 安全启动 & FIT 签名深度解析 (阶段一安全)
- [[bsp-uboot-boottime]] — U-Boot 启动速度优化 — 全链路分析与优化 (阶段一性能)
- [[bsp-uboot-rktools]] — Rockchip 工具链深度解析 (阶段一工具)
- [[bsp-uboot-env]] — U-Boot 环境变量系统深度解析 (阶段一配置)
- [[bsp-uboot-dm-deep]] — U-Boot Driver Model 深度解析 (阶段一核心)
- [[bsp-uboot-mmc]] — U-Boot MMC 子系统深度解析 (阶段一存储)
- [[bsp-uboot-usb]] — U-Boot USB 子系统深度解析 (阶段一通信)
- [[bsp-uboot-bringup]] — Board Bring-up 实战诊断指南 (阶段一调试)
- [[bsp-uboot-amp]] — AMP Boot 详解 (阶段一异构)
- [[bsp-device-model-dtb]] — 阶段二
- [[bsp-device-model-platform-bus-deep]] — Platform Bus 源码追溯 (阶段二深度)
- [[bsp-device-model-probe-deep]] — Driver Probe 全路径源码追溯 (阶段二深度)
- [[bsp-interrupt-concurrency]] — 阶段三
- [[bsp-peripheral-drivers]] — 阶段四
- [[bsp-power-management]] — 阶段五
- [[bsp-capstone-driver]] — 阶段六

### 第二段：相机进阶文档
- [[bsp-csi-sensor-driver]] — 阶段七
- [[bsp-v4l2-isp-pipeline]] — 阶段八
- [[bsp-isp-3a-rkaiq]] — 阶段九
- [[bsp-mpp-codec-pipeline]] — 阶段十
- [[bsp-rga-postprocess]] — 阶段十一
- [[bsp-camera-capstone]] — 阶段十二

### 附录 (已验证的实验数据)
- [[kernel-debug-env]] — 附录A：内核调试环境
- [[v4l2-isp-deep-dive]] — 附录B：V4L2/UVC 实验 (UVC 基础)
- [[mpp-hardware-codec]] — 附录C：MPP 编码实验 (MPP 基础)

### 大疆对标分析
- [[dji-bsp-analysis]] — 大疆嵌入式技术栈深度分析

### 平台资料
- [[rv1126b]] — RV1126B 运动相机项目
- [[rv1126-notes]] — RV1126B 嵌入式开发笔记
- [[MOC-工业相机]] — 相机与ISP知识

## 相关领域
- [[Home]] — 知识库主页
- [[MOC-MCU开发]] — 微控制器
- [[MOC-多媒体]] — FFmpeg/V4L2
- [[MOC-硬件设计]] — 电源/天线/PCB
