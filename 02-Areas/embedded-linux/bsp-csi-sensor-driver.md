---
tags:
  - embedded-linux
  - camera
  - mipi-csi
  - sensor-driver
  - v4l2-subdev
  - rockchip
  - dji
category: embedded-linux
created: 2026-06-22
updated: 2026-06-22
status: active
soc: Rockchip RV1126B
kernel: Linux 6.1.141
isp: V35
---

# 阶段七：MIPI CSI 协议 + Camera Sensor 驱动

> **大疆相机核心**：Camera Sensor 驱动是所有相机产品的起点。大疆每款产品都适配特定 Sensor (Sony IMX / SmartSens SC / OmniVision OV)，驱动质量直接影响画质和帧率。
>
> 本章从 MIPI CSI-2 协议出发，深入 V4L2 subdev 框架，分析 RV1126B 上的真实 Sensor 驱动，最终编写一个虚拟 Sensor 驱动。

---

## 一、MIPI CSI-2 协议

### 1.1 MIPI 联盟与 CSI-2

MIPI (Mobile Industry Processor Interface) 是移动设备接口标准。CSI-2 (Camera Serial Interface 2) 是相机传感器与 SoC 之间的串行接口标准。

```
Camera Sensor                    SoC
┌──────────┐    MIPI CSI-2     ┌──────────┐
│  Pixel   │    D-PHY (物理层)  │  CSI-2   │
│  Array   │──→──────────────→│  Receiver│
│  ISP     │    1~4 Lane       │  (Host)  │
│  I2C cfg │←─── SCL/SDA ──────│  I2C     │
└──────────┘                   └──────────┘
```

### 1.2 D-PHY 物理层

```
D-PHY 信号线:
  Clock Lane (1对差分): CLK+/CLK-
  Data Lane 0 (1对差分): D0+/D0-    (必须)
  Data Lane 1: D1+/D1-              (可选)
  Data Lane 2: D2+/D2-              (可选)
  Data Lane 3: D3+/D3-              (可选)

常见配置:
  1 Lane:  最多 ~1 Gbps (约 4K@15fps raw)
  2 Lane:  ~2 Gbps (约 1080p@60fps raw)
  4 Lane:  ~4 Gbps (约 4K@30fps raw)
```

| D-PHY 状态 | 电压 | 说明 |
|-----------|------|------|
| LP-00 (Low Power) | 1.2V | 空闲/控制 |
| LP-01 | 1.2V | 控制状态 |
| LP-10 | 1.2V | 控制状态 |
| LP-11 | 1.2V | Ready |
| HS (High Speed) | 200mV | 数据传输 (差分) |

### 1.3 CSI-2 包结构

```
CSI-2 传输包:
┌──────────┬──────────┬───────────────┬──────────┐
│ SoT      │ Header   │ Payload (Data)│ ECC+CRC  │ EoT │
│ (起始)   │ (1~4B)   │ (像素数据)     │ (校验)   │ (结束)│
└──────────┴──────────┴───────────────┴──────────┘

Header 包含:
  - Data Identifier (VC + DT)
  - Pixel Data Type (RAW8/RAW10/RAW12/RAW14, YUV422, RGB888...)
  - Word Count (payload 长度)

Virtual Channel (VC):
  - CSI-2 支持最多 4 个虚拟通道 (VC0~VC3)
  - 多 sensor 可共享同一组 Lane, 用 VC 区分
```

### 1.4 像素数据类型

| Data Type | 格式 | 每像素位数 | 用途 |
|-----------|------|-----------|------|
| 0x2A | RAW8 | 8 | 低端 sensor |
| 0x2B | RAW10 | 10 | 主流 sensor |
| 0x2C | RAW12 | 12 | 高画质 sensor |
| 0x2D | RAW14 | 14 | 专业级 sensor |
| 0x1E | YUV422 8-bit | 16 | 处理后输出 |
| 0x24 | RGB888 | 24 | 处理后输出 |
| 0x1B | Embedded Data | — | 寄存器/元数据 |

> **大疆相机**：高端产品用 RAW12/RAW14 (动态范围高)，消费级用 RAW10。

### 1.5 RV1126B MIPI 接口

```dts
/* rv1126b.dtsi — 2 物理 DPHY + 4 CSI-2 Host */

/* 物理 DPHY */
csi2_dphy0_hw: csi2-dphy0-hw@21c40000 {
    compatible = "rockchip,rv1126b-csi2-dphy-hw";
    reg = <0x21c40000 0x100>;
    status = "disabled";
};
csi2_dphy1_hw: csi2-dphy1-hw@21c50000 { ... };

/* 虚拟 DPHY (2 物理 = 6 虚拟) */
csi2_dphy0: csi2-dphy0 {    /* dphy0 全模式 4 Lane */
    compatible = "rockchip,rv1126b-csi2-dphy";
    status = "disabled";
};
csi2_dphy1: csi2-dphy1 {    /* dphy0 split 0+1 Lane */
    compatible = "rockchip,rv1126b-csi2-dphy";
    status = "disabled";
};
csi2_dphy2: csi2-dphy2 {    /* dphy0 split 2+3 Lane */
    ...
};
csi2_dphy3: csi2-dphy3 {    /* dphy1 全模式 4 Lane */
    ...
};
/* csi2_dphy4, csi2_dphy5: dphy1 split */

/* CSI-2 Host (4 路) */
mipi0_csi2_hw: mipi0-csi2-hw@21c00000 {
    compatible = "rockchip,rv1126b-mipi-csi2-hw";
    reg = <0x21c00000 0x100>;
    interrupts = <GIC_SPI 145 IRQ_TYPE_LEVEL_HIGH>,
                 <GIC_SPI 146 IRQ_TYPE_LEVEL_HIGH>;
    status = "disabled";
};
/* mipi1~3_csi2_hw 类似, 基址 0x21c10000~0x21c30000 */
```

| 配置模式 | DPHY 使用 | Lane 数 | 适用场景 |
|---------|----------|---------|---------|
| Full mode | dphy0 或 dphy1 | 4 Lane | 单路 4K sensor |
| Split mode | dphy0 → dphy1+dphy2 | 2+2 Lane | 双路 1080p sensor |

---

## 二、V4L2 Subdev 框架

### 2.1 V4L2 子设备概念

Camera Sensor 在 V4L2 框架中是一个 **subdev (子设备)**，不同于阶段二的字符设备：

```
传统字符设备 (阶段二):
  /dev/hello → file_operations (open/read/write/close)

V4L2 subdev (相机):
  /dev/v4l-subdevX → v4l2_subdev_ops (core/video/pad)
  不直接 read/write, 而是通过 ioctl 配置格式/帧率/流控
```

### 2.2 v4l2_subdev 核心结构

```c
/* V4L2 子设备操作集 */
struct v4l2_subdev_ops {
    const struct v4l2_subdev_core_ops  *core;   /* 电源/ ioctl /寄存器 */
    const struct v4l2_subdev_video_ops *video;  /* 流控 (s_stream) */
    const struct v4l2_subdev_pad_ops   *pad;    /* 格式/分辨率/帧率 */
    const struct v4l2_subdev_sensor_ops *sensor;/* 镜头/传感器 */
    ...
};

/* core_ops: 电源管理、寄存器读写 */
struct v4l2_subdev_core_ops {
    int (*s_power)(struct v4l2_subdev *sd, int on);  /* 上电/下电 */
    int (*ioctl)(struct v4l2_subdev *sd, ...);        /* 自定义 ioctl */
    int (*g_register)(...);  /* 读寄存器 (调试) */
    int (*s_register)(...);  /* 写寄存器 (调试) */
};

/* video_ops: 流控 */
struct v4l2_subdev_video_ops {
    int (*s_stream)(struct v4l2_subdev *sd, int enable); /* 启动/停止采集 */
};

/* pad_ops: 格式协商 (多端口) */
struct v4l2_subdev_pad_ops {
    int (*enum_mbus_code)(...);     /* 枚举支持的像素格式 */
    int (*enum_frame_size)(...);    /* 枚举支持的分辨率 */
    int (*enum_frame_interval)(...);/* 枚举支持的帧率 */
    int (*get_fmt)(...);            /* 获取当前格式 */
    int (*set_fmt)(...);            /* 设置格式 */
    int (*get_frame_interval)(...); /* 获取帧率 */
    int (*set_frame_interval)(...); /* 设置帧率 */
};
```

### 2.3 Sensor 驱动注册流程

```c
/* 1. 定义 of_match_table */
static const struct of_device_id sc450ai_of_match[] = {
    { .compatible = "smartsens,sc450ai" },
    { /* sentinel */ }
};

/* 2. 定义 i2c_driver (Sensor 是 I2C 设备) */
static struct i2c_driver sc450ai_i2c_driver = {
    .driver = {
        .name = "sc450ai",
        .of_match_table = of_match_ptr(sc450ai_of_match),
    },
    .probe  = sc450ai_probe,
    .remove = sc450ai_remove,
};

/* 3. probe 中初始化 V4L2 subdev */
static int sc450ai_probe(struct i2c_client *client, ...)
{
    /* 分配私有数据 */
    struct sc450ai *sensor = devm_kzalloc(...);

    /* 初始化 v4l2_subdev */
    v4l2_i2c_subdev_init(&sensor->sd, client, &sc450ai_subdev_ops);
    sensor->sd.flags |= V4L2_SUBDEV_FL_HAS_DEVNODE;

    /* 初始化 media entity */
    sensor->pad.flags = MEDIA_PAD_FL_SOURCE;
    media_entity_pads_init(&sensor->sd.entity, 1, &sensor->pad);
    sensor->sd.entity.function = MEDIA_ENT_F_CAM_SENSOR;

    /* 注册 subdev */
    v4l2_async_register_subdev(&sensor->sd);
    return 0;
}
```

> **关键理解**：Sensor 驱动 = I2C 驱动 (阶段四) + V4L2 subdev (本章新学)。I2C 用于读写寄存器，V4L2 subdev 用于格式/流控管理。

---

## 三、Camera Sensor 驱动深度分析

### 3.1 Sensor 驱动结构 (以 SC450AI 为例)

```
kernel-6.1/drivers/media/i2c/sc450ai.c

驱动结构:
  ├── I2C 注册 (i2c_driver)
  ├── V4L2 subdev (v4l2_subdev_ops)
  │   ├── core_ops: s_power (上电序列), g_register, s_register
  │   ├── video_ops: s_stream (启动/停止采集)
  │   └── pad_ops: enum_mbus_code, get_fmt, set_fmt, enum_frame_size
  ├── 寄存器初始化序列 (mode_table)
  │   ├── 1920x1080@30fps RAW10
  │   └── 其他分辨率/帧率
  ├── 电源管理 (regulator + gpio + clk)
  └── 媒体实体 (media_entity, pad)
```

### 3.2 上电序列 (s_power)

```c
static int sc450ai_s_power(struct v4l2_subdev *sd, int on)
{
    struct sc450ai *sensor = to_sc450ai(sd);

    if (on) {
        /* 上电序列 (严格按 datasheet 时序) */
        /* 1. 使能 regulator */
        regulator_enable(sensor->supplies.avdd);
        regulator_enable(sensor->supplies.dovdd);
        regulator_enable(sensor->supplies.dvdd);
        /* 2. 等待电压稳定 */
        msleep(1);
        /* 3. 拉低 RESET GPIO */
        gpiod_set_value(sensor->reset_gpio, 0);
        msleep(1);
        /* 4. 拉高 XCLK (MCLK) */
        clk_prepare_enable(sensor->xclk);
        msleep(1);
        /* 5. 释放 RESET (拉高) */
        gpiod_set_value(sensor->reset_gpio, 1);
        msleep(20);
        /* 6. 通过 I2C 写入初始寄存器 */
        sc450ai_write_regs(sensor, sc450ai_init_regs);
    } else {
        /* 下电序列 (逆序) */
        /* 停止采集 → 关 MCLK → 拉 RESET → 关 regulator */
    }
    return 0;
}
```

> **大疆关注点**：上电时序严格遵循 datasheet，时序错误可能导致 Sensor 不工作或画质异常。面试常考。

### 3.3 分辨率/帧率切换 (set_fmt)

```c
static int sc450ai_set_fmt(struct v4l2_subdev *sd,
                           struct v4l2_subdev_pad_config *cfg,
                           struct v4l2_subdev_format *format)
{
    struct sc450ai *sensor = to_sc450ai(sd);
    const struct sc450ai_mode *mode;

    /* 查找匹配的 mode */
    mode = sc450ai_find_mode(sensor, format->format.width,
                             format->format.height,
                             format->format.code);  /* mbus_code */

    if (!mode)
        return -EINVAL;

    /* 写入该 mode 的寄存器序列 */
    sc450ai_write_regs(sensor, mode->reg_list);

    /* 更新当前格式 */
    sensor->cur_mode = mode;
    format->format = *sc450ai_get_fmt(sd, cfg, format);

    return 0;
}

/* mode 定义示例 */
static const struct sc450ai_mode supported_modes[] = {
    {
        .width = 1920,
        .height = 1080,
        .max_fps = 30,
        .mbus_code = MEDIA_BUS_FMT_SBGGR10_1X10,  /* RAW10 Bayer */
        .reg_list = sc450ai_1920x1080_30fps_regs,  /* 寄存器序列 */
    },
    {
        .width = 1280,
        .height = 720,
        .max_fps = 60,
        .mbus_code = MEDIA_BUS_FMT_SBGGR10_1X10,
        .reg_list = sc450ai_1280x720_60fps_regs,
    },
};
```

### 3.4 启动/停止采集 (s_stream)

```c
static int sc450ai_s_stream(struct v4l2_subdev *sd, int enable)
{
    struct sc450ai *sensor = to_sc450ai(sd);

    if (enable) {
        /* 1. 写入 streaming-on 寄存器序列 */
        sc450ai_write_reg(sensor, SC450AI_REG_MODE_SELECT, 1);
        /* 2. 等待第一帧稳定 */
        msleep(30);
    } else {
        /* 停止 streaming */
        sc450ai_write_reg(sensor, SC450AI_REG_MODE_SELECT, 0);
    }
    return 0;
}
```

### 3.5 曝光/增益控制 (3A 接口)

```c
/* AE (自动曝光) 的硬件接口: sensor 提供曝光/增益寄存器 */
/* ISP 的 3A 算法计算后, 通过 V4L2 subdev ioctl 写入 sensor */

/* 设置曝光时间 (行数) */
static int sc450ai_set_exposure(struct v4l2_subdev *sd, u32 exposure)
{
    struct sc450ai *sensor = to_sc450ai(sd);
    /* 写曝光寄存器 (高位+中位+低位) */
    sc450ai_write_reg(sensor, SC450AI_REG_EXPOSURE_H, (exposure >> 16) & 0x0F);
    sc450ai_write_reg(sensor, SC450AI_REG_EXPOSURE_M, (exposure >> 8) & 0xFF);
    sc450ai_write_reg(sensor, SC450AI_REG_EXPOSURE_L, exposure & 0xFF);
    return 0;
}

/* 设置增益 (模拟+数字) */
static int sc450ai_set_gain(struct v4l2_subdev *sd, u32 gain)
{
    struct sc450ai *sensor = to_sc450ai(sd);
    /* 增益通常分模拟增益和数字增益两段 */
    sc450ai_write_reg(sensor, SC450AI_REG_ANA_GAIN, gain & 0xFF);
    if (gain > SC450AI_ANA_GAIN_MAX)
        sc450ai_write_reg(sensor, SC450AI_REG_DIG_GAIN,
                          gain / SC450AI_ANA_GAIN_MAX);
    return 0;
}
```

> **大疆核心**：3A 算法 (阶段九) 通过这些接口控制 Sensor。曝光/增益的精度和响应速度直接影响画质。

---

## 四、设备树相机管线配置

### 4.1 完整管线链接

```dts
/* 以 EVB 相机配置为例 (rv1126b-evb-cam-csi1.dtsi) */

/* 1. 启用 DPHY */
&csi2_dphy3 {
    status = "okay";
};

/* 2. 启用 CSI-2 Host */
&mipi2_csi2 {
    status = "okay";
};

/* 3. 启用 RKCIF */
&rkcif_mipi_lvds2 {
    status = "okay";
    /* 链接: CIF ← CSI-2 ← DPHY ← Sensor */
    port {
        cif_mipi_in2: endpoint {
            remote-endpoint = <&mipi2_csi2_output>;
        };
    };
};

/* 4. Sensor 节点 (I2C 上) */
&i2c4 {
    status = "okay";

    sc450ai: camera-sensor@27 {
        compatible = "smartsens,sc450ai";
        reg = <0x27>;                    /* I2C 地址 */
        clocks = <&cru CLK_MIPI_CAM_OUT>; /* MCLK */
        clock-names = "xvclk";

        /* 电源 */
        avdd-supply = <&vcc_mipipwr>;
        dovdd-supply = <&vcc3v3_sys>;
        dvdd-supply = <&vcc1v8_dvp>;

        /* 控制 GPIO */
        reset-gpios = <&gpio3 RK_PA6 GPIO_ACTIVE_LOW>;
        powerdom-gpios = <&gpio3 RK_PA5 GPIO_ACTIVE_HIGH>;

        /* MIPI 输出端口 → DPHY */
        port {
            sc450ai_out: endpoint {
                remote-endpoint = <&csi2_dphy3_in>;
                data-lanes = <1 2 3 4>;     /* 4 Lane */
            };
        };
    };
};

/* 5. DPHY ← Sensor 链接 */
&csi2_dphy3 {
    status = "okay";
    ports {
        port@0 {
            csi2_dphy3_in: endpoint {
                remote-endpoint = <&sc450ai_out>;
            };
        };
        port@1 {
            csi2_dphy3_out: endpoint {
                remote-endpoint = <&mipi2_csi2_input>;
            };
        };
    };
};

/* 6. CSI-2 ← DPHY, CSI-2 → CIF */
&mipi2_csi2 {
    status = "okay";
    ports {
        port@0 {
            mipi2_csi2_input: endpoint {
                remote-endpoint = <&csi2_dphy3_out>;
            };
        };
        port@1 {
            mipi2_csi2_output: endpoint {
                remote-endpoint = <&cif_mipi_in2>;
            };
        };
    };
};
```

### 4.2 管线拓扑图

```
sc450ai (I2C@0x27)
  │ sensor port (source)
  │ data-lanes = <1 2 3 4>
  ▼
csi2_dphy3 (DPHY)
  │ port0 ← sensor
  │ port1 → CSI-2
  ▼
mipi2_csi2 (CSI-2 Host)
  │ port0 ← DPHY
  │ port1 → CIF
  ▼
rkcif_mipi_lvds2 (CIF)
  │ port ← CSI-2
  ├──→ video0 (CAPTURE, DMA → DDR)
  └──→ sditf → rkisp_vir2 (ISP)
                    │
                    ▼
              rkisp_vir2_sditf → rkvpss_vir2 (VPSS)
```

---

## 五、实验 1：查看 RV1126B 相机拓扑

### 5.1 实验目标

用 `media-ctl` 查看相机管线拓扑，理解各实体间的链接关系。

### 5.2 前提条件

```bash
# 需要启用相机管线 (DTS overlay)
# 当前 sportcam 配置未启用相机, 需要:
# 1. 复制相机 DTS overlay 到内核目录
# 2. 修改 rv1126b-sportcam.dts include 相机 overlay
# 3. 重新编译内核 + dtb

# 检查 media-ctl 是否可用
which media-ctl
# 如果没有, 需在 buildroot 中启用:
# BR2_PACKAGE_LIBV4L_UTILS=y (已在 camera.config 中)
```

### 5.3 操作步骤

```bash
# 板端:
# 1. 查看所有 media 设备
ls /dev/media*
# 预期: /dev/media0 (CIF), /dev/media1 (ISP), ...

# 2. 查看拓扑
media-ctl -d /dev/media0 -p
# 预期输出:
# Media controller information
# ----------------------------
# Driver: rkcif
# ...
# Entities:
#  - sensor (1 pad, 0 links): [SOURCE]
#    -> "csi2_dphy3":0 [ENABLED]
#  - csi2_dphy3 (2 pads, 2 links)
#    <- "sensor":0 [ENABLED]
#    -> "mipi2_csi2":0 [ENABLED]
#  - mipi2_csi2 (2 pads, 2 links)
#    <- "csi2_dphy3":1 [ENABLED]
#    -> "rkcif_mipi_lvds2":0 [ENABLED]
#  - rkcif_mipi_lvds2 (1 pad, 1 link)
#    <- "mipi2_csi2":1 [ENABLED]
#    -> "video0": ...

# 3. 查看所有 V4L2 设备
ls /dev/video*
v4l2-ctl --list-devices

# 4. 查看 sensor 支持的格式
v4l2-ctl -d /dev/v4l-subdev0 --list-subdev-mbus-codes
# 预期: MEDIA_BUS_FMT_SBGGR10_1X10 等

# 5. 查看当前格式
v4l2-ctl -d /dev/v4l-subdev0 --get-subdev-fmt
```

---

## 六、实验 2：分析真实 Sensor 驱动源码

### 6.1 实验目标

阅读 SC450AI / IMX415 驱动源码，理解完整驱动结构。

### 6.2 分析清单

```bash
# PC 端: 阅读源码
# 1. SC450AI 驱动
#    kernel-6.1/drivers/media/i2c/sc450ai.c
#    重点关注:
#    - sc450ai_probe(): I2C + V4L2 + media_entity 初始化
#    - sc450ai_s_power(): 上电/下电序列
#    - sc450ai_s_stream(): 启动/停止采集
#    - sc450ai_set_fmt(): 分辨率切换
#    - sc450ai_supported_modes[]: 支持的分辨率/帧率
#    - 寄存器初始化序列 (大量寄存器写入)

# 2. IMX415 驱动
#    kernel-6.1/drivers/media/i2c/imx415.c
#    对比 SC450AI, 找共同点和差异

# 3. 驱动共性提取
#    所有 sensor 驱动都有:
#    - I2C 注册
#    - V4L2 subdev ops (core/video/pad)
#    - 电源序列 (regulator + gpio + clk)
#    - 寄存器初始化表
#    - media entity + pad
```

---

## 七、实验 3：编写虚拟 Sensor 驱动

### 7.1 实验目标

编写一个虚拟 Camera Sensor 驱动 (不连真实硬件)，验证 V4L2 subdev 注册和管线链接。

### 7.2 虚拟 Sensor 驱动 (virtual_sensor.c)

```c
#include <linux/module.h>
#include <linux/i2c.h>
#include <linux/v4l2-subdev.h>
#include <linux/media-entity.h>
#include <linux/of.h>

struct virtual_sensor {
    struct v4l2_subdev sd;
    struct media_pad pad;
    int streaming;
    int powered;
};

static const struct v4l2_mbus_framefmt default_fmt = {
    .width = 1920,
    .height = 1080,
    .code = MEDIA_BUS_FMT_SBGGR10_1X10,
    .field = V4L2_FIELD_NONE,
    .colorspace = V4L2_COLORSPACE_SRGB,
};

static int virtual_s_power(struct v4l2_subdev *sd, int on)
{
    struct virtual_sensor *vs = v4l2_get_subdevdata(sd);
    vs->powered = on;
    dev_info(sd->dev, "power %s\n", on ? "on" : "off");
    return 0;
}

static int virtual_s_stream(struct v4l2_subdev *sd, int enable)
{
    struct virtual_sensor *vs = v4l2_get_subdevdata(sd);
    vs->streaming = enable;
    dev_info(sd->dev, "streaming %s\n", enable ? "start" : "stop");
    return 0;
}

static int virtual_g_fmt(struct v4l2_subdev *sd,
                         struct v4l2_subdev_pad_config *cfg,
                         struct v4l2_subdev_format *fmt)
{
    fmt->format = default_fmt;
    return 0;
}

static int virtual_s_fmt(struct v4l2_subdev *sd,
                         struct v4l2_subdev_pad_config *cfg,
                         struct v4l2_subdev_format *fmt)
{
    fmt->format = default_fmt;
    return 0;
}

static int virtual_enum_mbus_code(struct v4l2_subdev *sd,
                                   struct v4l2_subdev_pad_config *cfg,
                                   struct v4l2_subdev_mbus_code_enum *code)
{
    if (code->index > 0)
        return -EINVAL;
    code->code = MEDIA_BUS_FMT_SBGGR10_1X10;
    return 0;
}

static int virtual_enum_frame_size(struct v4l2_subdev *sd,
                                    struct v4l2_subdev_pad_config *cfg,
                                    struct v4l2_subdev_frame_size_enum *fse)
{
    if (fse->index > 0)
        return -EINVAL;
    fse->min_width = fse->max_width = 1920;
    fse->min_height = fse->max_height = 1080;
    return 0;
}

static const struct v4l2_subdev_core_ops virtual_core_ops = {
    .s_power = virtual_s_power,
};

static const struct v4l2_subdev_video_ops virtual_video_ops = {
    .s_stream = virtual_s_stream,
};

static const struct v4l2_subdev_pad_ops virtual_pad_ops = {
    .enum_mbus_code = virtual_enum_mbus_code,
    .enum_frame_size = virtual_enum_frame_size,
    .get_fmt = virtual_g_fmt,
    .set_fmt = virtual_s_fmt,
};

static const struct v4l2_subdev_ops virtual_subdev_ops = {
    .core = &virtual_core_ops,
    .video = &virtual_video_ops,
    .pad = &virtual_pad_ops,
};

static int virtual_sensor_probe(struct i2c_client *client,
                                 const struct i2c_device_id *id)
{
    struct virtual_sensor *vs;
    int ret;

    vs = devm_kzalloc(&client->dev, sizeof(*vs), GFP_KERNEL);
    if (!vs)
        return -ENOMEM;

    v4l2_i2c_subdev_init(&vs->sd, client, &virtual_subdev_ops);
    vs->sd.flags |= V4L2_SUBDEV_FL_HAS_DEVNODE;

    vs->pad.flags = MEDIA_PAD_FL_SOURCE;
    ret = media_entity_pads_init(&vs->sd.entity, 1, &vs->pad);
    if (ret)
        return ret;
    vs->sd.entity.function = MEDIA_ENT_F_CAM_SENSOR;

    v4l2_set_subdevdata(&vs->sd, vs);

    ret = v4l2_async_register_subdev(&vs->sd);
    if (ret) {
        media_entity_cleanup(&vs->sd.entity);
        return ret;
    }

    dev_info(&client->dev, "virtual sensor registered\n");
    return 0;
}

static int virtual_sensor_remove(struct i2c_client *client)
{
    struct v4l2_subdev *sd = i2c_get_clientdata(client);
    v4l2_async_unregister_subdev(sd);
    media_entity_cleanup(&sd->entity);
    dev_info(&client->dev, "virtual sensor removed\n");
    return 0;
}

static const struct of_device_id virtual_sensor_match[] = {
    { .compatible = "my,virtual-sensor" },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, virtual_sensor_match);

static struct i2c_driver virtual_sensor_driver = {
    .probe = virtual_sensor_probe,
    .remove = virtual_sensor_remove,
    .driver = {
        .name = "virtual-sensor",
        .of_match_table = of_match_ptr(virtual_sensor_match),
    },
};

module_i2c_driver(virtual_sensor_driver);
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Virtual Camera Sensor Driver for V4L2 Pipeline Testing");
```

### 7.3 设备树

```dts
/* 在板级 DTS 中添加虚拟 sensor */
&i2c2 {
    status = "okay";

    virtual_sensor: virtual-sensor@30 {
        compatible = "my,virtual-sensor";
        reg = <0x30>;
        status = "okay";

        port {
            virtual_sensor_out: endpoint {
                remote-endpoint = <&csi2_dphy0_in>;
                data-lanes = <1 2>;
            };
        };
    };
};
```

### 7.4 验证

```bash
# 加载驱动
sudo insmod /tmp/virtual_sensor.ko
# 预期: "virtual sensor registered"

# 查看是否注册为 subdev
ls /dev/v4l-subdev*
v4l2-ctl -d /dev/v4l-subdevX --all

# 查看媒体拓扑中是否出现
media-ctl -d /dev/media0 -p | grep virtual
```

---

## 八、思考题

1. MIPI CSI-2 的 Virtual Channel (VC) 有什么用？如果两个 Sensor 共享同一组 Lane (4-Lane DPHY)，如何用 VC 区分？RV1126B 的 RKCIF 如何处理多 VC？

2. Camera Sensor 驱动的 `s_power` 上电序列中，为什么要先开电源、再释放 RESET、最后开 MCLK？如果顺序反了会怎样？

3. V4L2 subdev 的 `set_fmt` 和 `enum_frame_size` 的关系是什么？用户空间如何知道 Sensor 支持哪些分辨率？如果用户请求一个不支持的分辨率，驱动应该怎么处理？

4. 设备树中 `data-lanes = <1 2 3 4>` 表示什么？如果 Sensor 只支持 2 Lane MIPI，应该怎么配？DPHY 的 split mode 在什么场景下使用？

5. 大疆的相机产品通常支持多个分辨率和帧率组合（如 4K@30 + 1080p@60 + 720p@120）。在 Sensor 驱动中如何实现快速切换？切换时是否需要停止 streaming？

---

## 九、踩坑记录

| 日期 | 问题 | 原因 | 解决方案 |
|------|------|------|----------|
| | Sensor 不出图 | 上电时序不对 | 对照 datasheet 检查 power sequence |
| | I2C 读写 sensor 失败 | I2C 地址错 (7-bit vs 8-bit) | DTS 中 reg 用 7-bit 地址 |
| | media-ctl 找不到 sensor | DTS 管线 endpoint 未链接 | 检查 remote-endpoint 链路完整 |
| | s_stream 后无数据 | MCLK 未使能 | 检查 clocks = <&cru ...> 配置 |
| | 画面花屏 | mbus_code 不匹配 | sensor 输出格式与 CIF 配置一致 |
| | 帧率不对 | frame_interval 未设置 | set_frame_interval 或写 sensor 寄存器 |

---

## 十、下阶段预告

阶段八：**V4L2 框架 + ISP Pipeline + Media Controller**
- V4L2 核心框架：video_device / vb2_queue / buffer 生命周期
- Media Controller：entity / pad / link 拓扑管理
- RV1126B ISP V35 完整管线：CIF → SDITF → ISP → VPSS
- `media-ctl` + `v4l2-ctl` 手动配置管线采集帧
- RKCIF 4×MIPI Stream + DMA + online-to-ISP 模式

---

## 相关笔记

- [[MOC-嵌入式Linux]] — 学习路线总览
- [[bsp-peripheral-drivers]] — 阶段四：I2C (Sensor 驱动基础)
- [[bsp-device-model-dtb]] — 阶段二：设备树 (管线配置基础)
- [[v4l2-isp-deep-dive]] — 附录B：V4L2/UVC 实验数据
