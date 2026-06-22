---
tags:
  - embedded-linux
  - camera
  - v4l2
  - media-controller
  - isp
  - pipeline
  - vb2
  - rockchip
  - dji
category: embedded-linux
created: 2026-06-22
updated: 2026-06-22
status: active
soc: Rockchip RV1126B
kernel: Linux 6.1.141
isp: V35
vpss: V20
---

# 阶段八：V4L2 框架 + ISP Pipeline + Media Controller

> **大疆相机核心**：理解相机数据如何从 Sensor 流到内存。Media Controller 是大疆相机驱动架构的骨架——所有实体 (Sensor/CSI/ISP/VPSS) 都是 media entity，通过 link 组成管线。
>
> 本章是相机驱动的分水岭：从"单设备驱动"思维转向"管线式驱动"思维。

---

## 一、V4L2 核心框架

### 1.1 V4L2 设备类型

```
V4L2 设备层级:
  video_device (/dev/videoX)
    ├── capture: 采集节点 (ISP mainpath/selfpath, CIF stream)
    ├── output:  输出节点 (显示)
    ├── overlay: 叠加节点
    └── m2m:     mem2mem (编解码)

  v4l2_subdev (/dev/v4l-subdevX)
    ├── sensor subdev (Camera Sensor)
    ├── CSI subdev (MIPI CSI-2 Host)
    ├── ISP subdev (ISP 核心)
    └── VPSS subdev (后处理)

  media_device (/dev/mediaX)
    └── 管理整个管线的拓扑 (entity/pad/link)
```

### 1.2 核心数据结构

```c
/* video_device — V4L2 设备节点 (/dev/videoX) */
struct video_device {
    const struct v4l2_file_operations *fops;  /* open/ioctl/mmap 等 */
    struct v4l2_ioctl_ops *ioctl_ops;         /* V4L2 ioctl 处理 */
    struct media_entity entity;               /* 关联的 media entity */
    ...
};

/* vb2_queue — 视频缓冲区队列 */
struct vb2_queue {
    enum v4l2_buf_type type;         /* V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE */
    unsigned int io_modes;           /* VB2_MMAP | VB2_DMABUF */
    const struct vb2_ops *ops;       /* 缓冲区操作集 */
    const struct vb2_mem_ops *mem_ops; /* 内存操作 (dma-contig/dma-sg/vmalloc) */
    ...
};

/* vb2_ops — 驱动需要实现的缓冲区回调 */
struct vb2_ops {
    int (*queue_setup)(...);    /* REQBUFS 时调用: 计算 buffer 大小/数量 */
    int (*buf_prepare)(...);    /* QBUF 时调用: 准备 buffer (获取物理地址等) */
    void (*buf_queue)(...);     /* QBUF 时调用: 将 buffer 加入驱动队列 */
    int (*start_streaming)(...);/* STREAMON 时调用: 启动硬件 */
    void (*stop_streaming)(...);/* STREAMEOFF 时调用: 停止硬件 */
};

/* v4l2_subdev — 子设备 (Sensor/CSI/ISP) */
struct v4l2_subdev {
    struct media_entity entity;
    const struct v4l2_subdev_ops *ops;
    struct v4l2_subdev_pad_config *pad_cfg;
    ...
};
```

### 1.3 V4L2 采集流程 (内核侧)

```
用户空间 ioctl 流程          内核回调
─────────────────────────────────────────────────
open("/dev/video0")     →  video_device.fops.open
  │
VIDIOC_QUERYCAP         →  ioctl_ops.vidioc_querycap
  │
VIDIOC_S_FMT            →  ioctl_ops.vidioc_s_fmt_vid_cap_mplane
  │                        → subdev.set_fmt (传递到 Sensor)
  │
VIDIOC_REQBUFS(n)       →  vb2_ops.queue_setup
  │                        → 计算每 buffer 大小, 分配
  │
VIDIOC_QBUF × n         →  vb2_ops.buf_prepare + buf_queue
  │                        → buffer 入队
  │
mmap()                  →  vb2_mem_ops.mmap
  │                        → 映射 buffer 到用户空间
  │
VIDIOC_STREAMON         →  vb2_ops.start_streaming
  │                        → subdev.s_stream(1) (启动 Sensor)
  │                        → 启动 DMA
  │
                        →  [硬件中断: 帧完成]
                        →  vb2_buffer_done(buf, VB2_BUF_STATE_DONE)
  │
VIDIOC_DQBUF            →  从完成队列取出一帧
  │                        → 用户空间拿到已填充的 buffer
  │
VIDIOC_QBUF             →  重新入队 (循环使用)
  │
VIDIOC_STREAMOFF        →  vb2_ops.stop_streaming
  │                        → subdev.s_stream(0) (停止 Sensor)
close()
```

### 1.4 vb2 Buffer 生命周期

```
                    驱动内部队列
                   ┌──────────────┐
  QBUF ──────────→ │  queued      │ ──→ 驱动取用
  (用户入队)        │  buffers     │
                   └──────┬───────┘
                          │ 驱动取出用于 DMA
                   ┌──────▼───────┐
                   │  active      │ ──→ 硬件 DMA 写入
                   │  (DMA中)     │
                   └──────┬───────┘
                          │ 帧完成中断
                   ┌──────▼───────┐
  DQBUF ←───────── │  done        │ ──→ 用户取走
  (用户取帧)        │  (已完成)    │
                   └──────────────┘
                          │ 用户 QBUF 重新入队
                          └──→ 回到 queued
```

> **大疆面试必考**：vb2 buffer 的 QBUF→DQBUF 生命周期。零拷贝的关键是 mmap 后用户态直接访问 DMA buffer。

---

## 二、Media Controller

### 2.1 为什么需要 Media Controller

传统 V4L2 只有 video 节点，无法表达"Sensor → CSI → ISP → 输出"这种多级管线。Media Controller 引入了 entity/pad/link 模型：

```
Entity (实体)     — 一个硬件模块 (Sensor, CSI, ISP, video 节点)
  Pad (端口)      — Entity 的输入/输出端口
    SOURCE pad   — 输出端口
    SINK pad     — 输入端口
  Link (链接)     — 两个 Pad 之间的连接
    SOURCE pad → SINK pad
```

### 2.2 RV1126B 相机管线 Media 拓扑

```
media-ctl -d /dev/media0 -p 输出:

Entity: sc450ai (CAM_SENSOR, 1 pad)
  Pad 0: SOURCE                    ← Sensor 输出
    → "csi2_dphy3":0 [ENABLED]    ← 链接到 DPHY 输入

Entity: csi2_dphy3 (PHY, 2 pads)
  Pad 0: SINK                     ← DPHY 输入
    ← "sc450ai":0 [ENABLED]
  Pad 1: SOURCE                   ← DPHY 输出
    → "mipi2_csi2":0 [ENABLED]

Entity: mipi2_csi2 (CSI-2, 2 pads)
  Pad 0: SINK
    ← "csi2_dphy3":1 [ENABLED]
  Pad 1: SOURCE
    → "rkcif_mipi_lvds2":0 [ENABLED]

Entity: rkcif_mipi_lvds2 (CIF, 1 pad)
  Pad 0: SINK
    ← "mipi2_csi2":1 [ENABLED]
    → "rkcif_mipi_lvds2_sditf":0 [ENABLED]  ← 到 ISP

Entity: rkisp_vir2 (ISP, 2 pads)
  Pad 0: SINK
    ← "rkcif_mipi_lvds2_sditf":1 [ENABLED]
  Pad 1: SOURCE
    → "rkvpss_vir2":0 [ENABLED]

Entity: video0 (V4L2 capture, 1 pad)  ← ISP mainpath 输出
  Pad 0: SINK
    ← ISP internal link

Entity: rkvpss_vir2 (VPSS, 2 pads)
  Pad 0: SINK
    ← "rkisp_vir2":1
  Pad 1: SOURCE
    → video output nodes
```

### 2.3 Media Controller API

```c
/* 内核侧: 注册 media entity */
struct media_entity entity;
struct media_pad pads[2];

pads[0].flags = MEDIA_PAD_FL_SINK;    /* 输入端口 */
pads[1].flags = MEDIA_PAD_FL_SOURCE;  /* 输出端口 */

media_entity_pads_init(&entity, 2, pads);
entity.function = MEDIA_ENT_F_PROC_VIDEO_SCALER; /* 实体类型 */

/* 创建 link */
media_create_pad_link(source_entity, source_pad,
                      sink_entity, sink_pad, flags);
/* flags: MEDIA_LNK_FL_ENABLED (启用) | MEDIA_LNK_FL_IMMUTABLE (不可断开) */
```

```bash
# 用户侧: media-ctl 操作
# 查看拓扑
media-ctl -d /dev/media0 -p

# 启用/禁用 link
media-ctl -d /dev/media0 -l '"sc450ai":0 -> "csi2_dphy3":0 [1]'
#                                          [1]=启用 [0]=禁用

# 设置 subdev 格式 (沿管线传播)
media-ctl -d /dev/media0 --set-v4l2 '"sc450ai":0[fmt:SBGGR10_1X10/1920x1080]'
media-ctl -d /dev/media0 --set-v4l2 '"csi2_dphy3":0[fmt:SBGGR10_1X10/1920x1080]'
media-ctl -d /dev/media0 --set-v4l2 '"mipi2_csi2":0[fmt:SBGGR10_1X10/1920x1080]'
media-ctl -d /dev/media0 --set-v4l2 '"rkcif_mipi_lvds2":0[fmt:SBGGR10_1X10/1920x1080]'
```

---

## 三、RV1126B 相机管线详解

### 3.1 完整管线

```
[Sensor] → [DPHY] → [CSI-2] → [RKCIF] ─┬─ CAPTURE: DMA → DDR (raw)
                                        │
                                        └─ SDITF → [RKISP V35] ─┬─ mainpath → DDR (NV12/YUV)
                                                                  │                 ↓
                                                                  ├─ selfpath → DDR  [VPSS V20] → 多路输出
                                                                  │
                                                                  ├─ stats → DDR (3A 统计)
                                                                  │
                                                                  └─ params ← DDR (3A 参数)
```

### 3.2 RKCIF (Camera Interface Frontend)

RKCIF 是相机数据的第一站，负责从 MIPI CSI-2 接收数据。

```c
/* RKCIF 注册的实体 (dev.c) */
rkcif_register_platform_subdevs() {
    rkcif_register_stream_vdevs();  // 4×MIPI video stream 节点
    rkcif_register_scale_vdevs();   // scale video 节点
    rkcif_register_tools_vdevs();   // tools video 节点
    rkcif_register_luma_vdev();     // luma 统计节点
    cif_subdev_notifier();          // async 绑定 sensor
}
```

| Stream 模式 | 说明 | 数据去向 |
|-------------|------|---------|
| `CAPTURE` | 直接 DMA 到 DDR | 用户直接获取 raw 数据 |
| `TOISP` | 在线传给 ISP | 不经过 DDR, 低延迟 |
| `TOSCALE` | 传给 scale 引擎 | 缩放后输出 |
| `TOISP_RDBK` | ISP 回读模式 | 先存 DDR 再给 ISP |
| `ROCKIT` | rockit 管线 | 给 Rockit 框架管理 |

> **大疆关注点**：`TOISP` 模式 (online) 是低延迟的关键——数据不落 DDR，直接从 CIF 传到 ISP。大疆图传的 <20ms 延迟依赖这种在线管线。

### 3.3 RKISP V35 (Image Signal Processor)

ISP 是相机管线的核心——Raw Bayer 数据在这里变成可用的 YUV/RGB 图像。

```c
/* RKISP V35 注册的实体 (dev.c) */
rkisp_register_platform_subdevs() {
    rkisp_register_isp_subdev();      // ISP 核心 subdev
    rkisp_register_csi_subdev();      // CSI subdev (ISP 内部)
    rkisp_register_bridge_subdev();   // bridge subdev
    rkisp_register_stream_v35();      // V35 特有的 capture 节点
    rkisp_register_dmarx_vdev();      // DMA raw 接收
    rkisp_register_stats_vdev();      // 3A 统计节点
    rkisp_register_params_vdev();     // 3A 参数节点
    rkisp_register_luma_vdev();       // MIPI luma
    rkisp_register_pdaf_vdev();       // PDAF
    isp_subdev_notifier();            // 绑定 sensor
}

/* V35 capture 注册的 stream (capture_v35.c) */
rkisp_register_stream_v35() {
    rkisp_stream_init(dev, RKISP_STREAM_MP);   // mainpath (主输出)
    rkisp_stream_init(dev, RKISP_STREAM_SP);   // selfpath (辅助输出)
    rkisp_stream_init(dev, RKISP_STREAM_VIR);  // 虚拟 stream
}
```

ISP V35 输出路径：

```
ISP 输出:
  mainpath ──→ DDR (高分辨率, 如 4K NV12)
  selfpath  ──→ DDR (低分辨率, 如 1080p NV16)

  mainpath 和 selfpath 可同时输出不同分辨率
  → 大疆多路输出: 主码流(4K) + 预览(1080p) + 缩略图
```

ISP 支持的输出格式 (capture_v35.c)：

| 格式 | FourCC | 说明 |
|------|--------|------|
| NV12 | `V4L2_PIX_FMT_NV12` | YUV420 半平面 (最常用) |
| NV21 | `V4L2_PIX_FMT_NV21` | NV12 的 UV 交换 |
| NV16 | `V4L2_PIX_FMT_NV16` | YUV422 半平面 |
| NV61 | `V4L2_PIX_FMT_NV61` | NV16 的 UV 交换 |
| UYVY | `V4L2_PIX_FMT_UYVY` | YUV422 交织 |
| YUYV | `V4L2_PIX_FMT_YUYV` | YUV422 交织 |

### 3.4 ISP Stats / Params 节点 (3A 接口)

```
ISP 3A 数据流:

  [Sensor 输出 Raw] → ISP 处理 → [YUV 输出]
                          │
                          ├──→ stats video node (/dev/videoX)
                          │    每帧输出: 直方图、AE 统计、AWB 统计、AF 统计
                          │    → RKAIQ 3A 算法读取并计算
                          │
                          ←── params video node (/dev/videoX)
                               每帧输入: AE 参数(曝光/增益)、AWB 参数(白平衡)、
                               NR 参数(降噪)、Sharpen 参数(锐化)...
                               ← RKAIQ 3A 算法计算后写入
```

> **关键理解**：ISP 的 stats/params 节点是 3A 算法和硬件 ISP 之间的桥梁。每帧处理完成后 stats 上报，3A 算法计算新参数，通过 params 下发给下一帧。这是阶段九的核心内容。

### 3.5 VPSS V20 (Video Post-Processing Subsystem)

VPSS 位于 ISP 之后，提供视频后处理：

```
ISP mainpath → VPSS → 多路输出
                     ├── 主码流 (4K → MPP 编码)
                     ├── 预览流 (1080p → 显示)
                     └── 分析流 (720p → NPU AI 推理)
```

```c
/* VPSS 注册 (dev.c) */
rkvpss_register_platform_subdevs() {
    // VPSS V4L2 subdev
    // stream video nodes (多路输出)
    // async notifier (绑定 ISP)
}
```

---

## 四、设备树管线配置 (完整版)

### 4.1 SDITF 链接 (CIF → ISP)

```dts
/* rv1126b.dtsi 中 ISP 虚拟节点 */
rkisp_vir0: rkisp-vir0 {
    compatible = "rockchip,rkisp-vir";
    status = "disabled";
    port {
        #address-cells = <1>;
        #size-cells = <0>;
        rkisp_vir0_in: endpoint@0 {
            reg = <0>;
            remote-endpoint = <&rkcif_mipi_lvds_sditf0>;
        };
    };
};

/* SDITF (ISP→VPSS) */
rkisp_vir0_sditf: rkisp-vir0-sditf {
    compatible = "rockchip,rkisp-sditf";
    status = "disabled";
    port {
        rkisp_vir0_sditf_in: endpoint {
            remote-endpoint = <&isp_sditf0>;
        };
        rkisp_vir0_sditf_out: endpoint {
            remote-endpoint = <&vpss0_in>;
        };
    };
};

/* VPSS 虚拟节点 */
rkvpss_vir0: rkvpss-vir0 {
    compatible = "rockchip,rkvpss-vir";
    status = "disabled";
    port {
        rkvpss_vir0_in: endpoint {
            remote-endpoint = <&rkisp_vir0_sditf_out>;
        };
    };
};
```

### 4.2 完整链路 (DTS remote-endpoint 链)

```
sensor.out → dphy.in         (sensor → DPHY)
dphy.out → csi2.in           (DPHY → CSI-2)
csi2.out → cif.in            (CSI-2 → CIF)
cif.sditf → isp_vir.in       (CIF → ISP, 在线模式)
isp.sditf → vpss_vir.in      (ISP → VPSS)
```

---

## 五、实验 1：手动配置管线采集一帧

### 5.1 实验目标

用 `media-ctl` + `v4l2-ctl` 手动配置 RV1126B 相机管线，采集一帧 NV12 图像。

### 5.2 操作步骤

```bash
# 前提: DTS 中相机管线已启用 (需要 camera DTS overlay)
# 前提: rootfs 中有 media-ctl 和 v4l2-ctl

# 板端:

# 1. 查看拓扑
media-ctl -d /dev/media0 -p

# 2. 找到各实体名称
# sensor: sc450ai
# DPHY: csi2_dphy3
# CSI-2: mipi2_csi2
# CIF: rkcif_mipi_lvds2
# ISP capture: /dev/videoX (mainpath)

# 3. 设置 Sensor 格式 (RAW10 1920x1080)
media-ctl -d /dev/media0 --set-v4l2 '"sc450ai":0[fmt:SBGGR10_1X10/1920x1080]'

# 4. 设置 DPHY 格式 (沿管线传播)
media-ctl -d /dev/media0 --set-v4l2 '"csi2_dphy3":0[fmt:SBGGR10_1X10/1920x1080]'

# 5. 设置 CSI-2 格式
media-ctl -d /dev/media0 --set-v4l2 '"mipi2_csi2":0[fmt:SBGGR10_1X10/1920x1080]'

# 6. 设置 CIF 格式
media-ctl -d /dev/media0 --set-v4l2 '"rkcif_mipi_lvds2":0[fmt:SBGGR10_1X10/1920x1080]'

# 7. 启用所有 link
media-ctl -d /dev/media0 -l '"sc450ai":0 -> "csi2_dphy3":0 [1]'
media-ctl -d /dev/media0 -l '"csi2_dphy3":1 -> "mipi2_csi2":0 [1]'
media-ctl -d /dev/media0 -l '"mipi2_csi2":1 -> "rkcif_mipi_lvds2":0 [1]'
# CIF → ISP link
media-ctl -d /dev/media0 -l '"rkcif_mipi_lvds2_sditf":1 -> "rkisp_vir2":0 [1]'

# 8. 设置 ISP 输出格式 (NV12)
v4l2-ctl -d /dev/videoX \
    --set-fmt-video width=1920,height=1080,pixelformat=NV12

# 9. 请求 buffer + 采集
v4l2-ctl -d /dev/videoX \
    --stream-mmap --stream-count=1 \
    --stream-to=/tmp/frame_nv12.raw

# 10. 检查输出
ls -la /tmp/frame_nv12.raw
# 预期: 1920*1080*3/2 = 3110400 bytes (NV12)
```

### 5.3 预期结果

```
NV12 帧大小 = width × height × 3 / 2
            = 1920 × 1080 × 1.5
            = 3,110,400 bytes

hexdump -C /tmp/frame_nv12.raw | head -5
# Y 平面: 0x00~0x00200000 (1920*1080=2073600 bytes)
# UV 平面: 0x00200000~0x002F7600 (1036800 bytes)
```

---

## 六、实验 2：C 语言 V4L2 ISP 采集程序

### 6.1 实验目标

编写 C 程序通过 V4L2 从 ISP mainpath 采集一帧 NV12，验证对 ioctl 流程的理解。

### 6.2 源码 (isp_capture.c)

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <linux/videodev2.h>

#define DEV     "/dev/video20"   /* ISP mainpath (根据实际调整) */
#define WIDTH   1920
#define HEIGHT  1080
#define BUF_NUM 3

struct buffer {
    void   *start;
    size_t  length;
};

int main(void)
{
    int fd = open(DEV, O_RDWR);
    if (fd < 0) { perror("open"); return 1; }

    /* 1. QUERYCAP */
    struct v4l2_capability cap;
    ioctl(fd, VIDIOC_QUERYCAP, &cap);
    printf("driver: %s, card: %s\n", cap.driver, cap.card);

    /* 2. S_FMT — NV12 (多平面) */
    struct v4l2_format fmt;
    memset(&fmt, 0, sizeof(fmt));
    fmt.type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
    fmt.fmt.pix_mp.width = WIDTH;
    fmt.fmt.pix_mp.height = HEIGHT;
    fmt.fmt.pix_mp.pixelformat = V4L2_PIX_FMT_NV12;
    fmt.fmt.pix_mp.num_planes = 1;
    ioctl(fd, VIDIOC_S_FMT, &fmt);
    printf("fmt: %dx%d NV12\n", fmt.fmt.pix_mp.width, fmt.fmt.pix_mp.height);

    /* 3. REQBUFS */
    struct v4l2_requestbuffers req;
    memset(&req, 0, sizeof(req));
    req.count = BUF_NUM;
    req.type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
    req.memory = V4L2_MEMORY_MMAP;
    ioctl(fd, VIDIOC_REQBUFS, &req);
    printf("buffers: %d\n", req.count);

    /* 4. QUERYBUF + MMAP */
    struct buffer bufs[BUF_NUM];
    struct v4l2_buffer buf;
    struct v4l2_plane planes[1];
    for (int i = 0; i < req.count; i++) {
        memset(&buf, 0, sizeof(buf));
        memset(planes, 0, sizeof(planes));
        buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
        buf.memory = V4L2_MEMORY_MMAP;
        buf.index = i;
        buf.m.planes = planes;
        buf.length = 1;
        ioctl(fd, VIDIOC_QUERYBUF, &buf);
        bufs[i].length = planes[0].length;
        bufs[i].start = mmap(NULL, planes[0].length,
            PROT_READ | PROT_WRITE, MAP_SHARED,
            fd, planes[0].m.mem_offset);
        printf("buf[%d]: %zu bytes @ %p\n", i, bufs[i].length, bufs[i].start);
    }

    /* 5. QBUF — 所有 buffer 入队 */
    for (int i = 0; i < req.count; i++) {
        memset(&buf, 0, sizeof(buf));
        memset(planes, 0, sizeof(planes));
        buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
        buf.memory = V4L2_MEMORY_MMAP;
        buf.index = i;
        buf.m.planes = planes;
        buf.length = 1;
        ioctl(fd, VIDIOC_QBUF, &buf);
    }

    /* 6. STREAMON */
    enum v4l2_buf_type type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
    ioctl(fd, VIDIOC_STREAMON, &type);

    /* 7. DQBUF — 取一帧 */
    memset(&buf, 0, sizeof(buf));
    memset(planes, 0, sizeof(planes));
    buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
    buf.memory = V4L2_MEMORY_MMAP;
    buf.m.planes = planes;
    buf.length = 1;
    ioctl(fd, VIDIOC_DQBUF, &buf);
    printf("frame #%u, bytesused=%u\n",
           buf.sequence, planes[0].bytesused);

    /* 8. 保存 */
    FILE *fp = fopen("/tmp/isp_frame.nv12", "wb");
    fwrite(bufs[buf.index].start, 1, planes[0].bytesused, fp);
    fclose(fp);
    printf("Saved /tmp/isp_frame.nv12\n");

    /* 9. 清理 */
    ioctl(fd, VIDIOC_STREAMOFF, &type);
    for (int i = 0; i < req.count; i++)
        munmap(bufs[i].start, bufs[i].length);
    close(fd);
    return 0;
}
```

> **与阶段二 UVC 程序的区别**：
> - UVC 用 `V4L2_BUF_TYPE_VIDEO_CAPTURE` (单平面)
> - ISP 用 `V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE` (多平面)
> - ISP 需要先用 `media-ctl` 配置管线，UVC 即插即用
> - ISP 输出 NV12 (ISP 处理后)，UVC 输出 YUYV (Sensor 直出)

---

## 七、实验 3：Ftrace 追踪 ISP 帧处理路径

### 7.1 实验目标

用 Ftrace 追踪从 `STREAMON` 到 `DQBUF` 的 ISP 内核调用路径。

### 7.2 操作步骤

```bash
# 板端:
echo function_graph | sudo tee /sys/kernel/tracing/current_tracer
echo '*rkisp*|*rkcif*|*vb2*' | sudo tee /sys/kernel/tracing/set_ftrace_filter
echo 256 | sudo tee /sys/kernel/tracing/max_graph_depth

echo | sudo tee /sys/kernel/tracing/trace

# 运行采集程序
sudo /tmp/isp_capture

sudo cat /sys/kernel/tracing/trace > /tmp/isp_trace.log
echo nop | sudo tee /sys/kernel/tracing/current_tracer
```

### 7.3 预期调用链

```
STREAMON:
  vb2_streamon()
    → rkisp_stream_start_streaming()
      → vb2_start_streaming()
        → subdev.s_stream(1)
          → rkisp_isp_sd_s_stream()
            → rkcif_s_stream() → sensor s_stream(1)
      → 启动 ISP 硬件
      → 启动 DMA (mainpath)

帧完成中断:
  rkisp_mi_v35_isr()              ← ISP 帧完成中断
    → mi_frame_end()
      → vb2_buffer_done()          ← 标记 buffer 完成
        → 唤醒 DQBUF 等待

DQBUF:
  vb2_ioctl_dqbuf()
    → vb2_core_dqbuf()
      → 返回已完成的 buffer
```

---

## 八、实验 4：分析 ISP stats/params 数据流

### 8.1 实验目标

理解 3A 算法与 ISP 之间的数据交换机制。

### 8.2 操作步骤

```bash
# 查看 ISP 注册的所有 video 节点
v4l2-ctl --list-devices
# 预期:
# rkcif (media0):
#   /dev/video0  (CIF stream 0)
#   /dev/video1  (CIF stream 1)
# rkisp (media1):
#   /dev/video10 (ISP mainpath)
#   /dev/video11 (ISP selfpath)
#   /dev/video12 (ISP stats)    ← 3A 统计输出
#   /dev/video13 (ISP params)   ← 3A 参数输入
#   /dev/video14 (ISP dmarx)
# rkvpss (media2):
#   /dev/video20 (VPSS output)

# 查看 stats 节点能力
v4l2-ctl -d /dev/video12 --all
# 预期: type = VIDEO_CAPTURE, 用于读取 ISP 统计数据

# 查看 params 节点能力
v4l2-ctl -d /dev/video13 --all
# 预期: type = VIDEO_OUTPUT, 用于写入 ISP 参数
```

### 8.3 数据流理解

```
帧 N 处理:
  1. ISP 硬件处理 Raw → YUV
  2. ISP 硬件生成统计 (直方图/AE/AWB/AF)
  3. stats 节点 DQBUF → RKAIQ 读取统计
  4. RKAIQ 计算: 新的曝光/增益/白平衡/降噪参数
  5. params 节点 QBUF → 写入新参数给帧 N+1
  6. 循环

每帧都有一次 stats→算法→params 的往返。
3A 算法的处理延迟必须 < 1帧时间 (33ms@30fps)。
```

---

## 九、UVC vs ISP 管线对比

> 附录B 的 UVC 实验与本章 ISP 管线的核心区别：

| 维度 | UVC (附录B) | ISP 管线 (本章) |
|------|------------|----------------|
| Sensor | USB 摄像头自带 | MIPI CSI Sensor (I2C 配置) |
| ISP | USB 摄像头内置 | SoC ISP V35 (可编程) |
| 管线 | 单节点 (/dev/video1) | 多级 (Sensor→DPHY→CSI→CIF→ISP→VPSS) |
| Media Controller | 不需要 | 必须 (media-ctl 配管线) |
| 格式 | YUYV/MJPG (Sensor 直出) | NV12 (ISP 处理后) |
| 3A | 摄像头固件处理 | RKAIQ 用户态算法 |
| buffer 类型 | 单平面 | 多平面 (MPLANE) |
| 零拷贝 | 1次拷贝 (URB→vb2) | 0次拷贝 (DMA 直写) |
| 延迟 | 高 (USB 往返) | 低 (在线模式) |
| 灵活性 | 低 (固件固定) | 高 (每帧可调参数) |

> **大疆选择 ISP 管线的原因**：画质可控 (3A 算法)、低延迟 (在线模式)、灵活 (多路输出)、零拷贝 (DMA)。

---

## 十、思考题

1. `V4L2_BUF_TYPE_VIDEO_CAPTURE` 和 `V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE` 的区别是什么？为什么 ISP 用多平面而 UVC 用单平面？

2. Media Controller 中 link 的 `ENABLED` 和 `IMMUTABLE` 标志有什么区别？什么场景下需要动态断开 link？

3. RKCIF 的 `TOISP` (在线) 模式和 `CAPTURE` (DMA到DDR) 模式的延迟差异有多大？为什么在线模式更快但灵活性更低？

4. ISP 的 stats 和 params 节点为什么是 video 节点而不是 sysfs 或 ioctl？这种设计有什么好处？

5. 大疆的多相机产品 (如 Mavic 3 三摄) 如何在 Media Controller 中表达？多个 Sensor 如何共享或独占 ISP 资源？

---

## 十一、踩坑记录

| 日期 | 问题 | 原因 | 解决方案 |
|------|------|------|----------|
| | media-ctl 找不到设备 | 相机 DTS 未启用 | 添加 camera DTS overlay, status="okay" |
| | STREAMON 失败 | 管线 link 未全部启用 | 用 media-ctl 检查所有 link [ENABLED] |
| | 采集帧全黑 | Sensor 未上电或 s_stream 失败 | 检查 sensor s_power 和 s_stream dmesg |
| | 帧大小不对 | NV12 vs NV21 vs YUYV 混淆 | 确认 S_FMT 的 pixelformat |
| | DQBUF 阻塞不返回 | 没有帧完成中断 | 检查 ISP IRQ 和 sensor streaming |
| | MPLANE buf.length=0 | QUERYBUF 前没 REQBUFS | 确保 REQBUFS → QUERYBUF → QBUF 顺序 |

---

## 十二、下阶段预告

阶段九：**ISP 3A 算法 + RKAIQ 集成**
- ISP 处理流水线各模块 (BLC→LSC→AWB→Demosaic→NR→Sharpen)
- 3A 算法原理 (AE/AWB/AF)
- RKAIQ 架构与 65+ 算法模块
- IQ 调校文件 (JSON) 解析
- 实战：运行 3A server，调参观察画质变化

---

## 相关笔记

- [[MOC-嵌入式Linux]] — 学习路线总览
- [[bsp-csi-sensor-driver]] — 阶段七：MIPI CSI + Sensor 驱动
- [[v4l2-isp-deep-dive]] — 附录B：V4L2/UVC 实验数据 (UVC 基础对比)
- [[bsp-device-model-dtb]] — 阶段二：设备模型 (Media Entity 基础)
