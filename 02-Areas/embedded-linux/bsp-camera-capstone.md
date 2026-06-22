---
tags:
  - embedded-linux
  - camera
  - capstone
  - pipeline
  - sensor
  - isp
  - 3a
  - mpp
  - rga
  - rockchip
  - dji
category: embedded-linux
created: 2026-06-22
updated: 2026-06-22
status: active
soc: Rockchip RV1126B
kernel: Linux 6.1.141
---

# 阶段十二：相机 Capstone — 完整相机管线项目

> **大疆相机核心**：这是整个学习路线的收尾——整合阶段七~十一所有知识，搭建一个能跑的完整相机管线。这是面试展示作品。
>
> 项目目标：Sensor 采集 → ISP 处理 → 3A 运行 → MPP 编码 → 文件保存，完整链路跑通。

---

## 一、项目概述

### 1.1 目标

搭建一个完整的相机管线，实现：

```
[Sensor] → [DPHY] → [CSI-2] → [CIF] → [ISP V35] → [VPSS] → [MPP H.265] → MP4 文件
                                ↑
                        [RKAIQ 3A] (AE/AWB/AF)
```

### 1.2 功能清单

| 功能 | 技术来源 | 阶段 |
|------|---------|------|
| Media Controller 管线配置 | media-ctl + v4l2-ctl | 八 |
| V4L2 采集 NV12 帧 | V4L2 MPLANE ioctl | 八 |
| 3A 算法运行 | RKAIQ uAPI2 | 九 |
| H.265 硬件编码 | MPP MPI | 十 |
| OSD 时间戳叠加 | RGA Blending | 十一 |
| 多路输出 (主码流+预览) | VPSS + Rockit | 十~十一 |
| 性能测量 (帧率/延迟/功耗) | Ftrace + power | 全程 |

### 1.3 预期输出

```
/tmp/camera_capture.h265  — H.265 编码的视频文件
/tmp/camera_preview.nv12  — 预览帧 (可选)
/tmp/camera_stats.log     — 性能统计日志
```

---

## 二、完整管线代码 (camera_pipeline.c)

```c
/*
 * camera_pipeline.c — RV1126B 完整相机管线
 *
 * 流程:
 *   1. media-ctl 配置 Sensor→DPHY→CSI→CIF→ISP 管线
 *   2. V4L2 从 ISP mainpath 采集 NV12 帧
 *   3. MPP 将 NV12 编码为 H.265
 *   4. 保存 H.265 文件
 *
 * 编译: aarch64-none-linux-gnu-gcc -o camera_pipeline camera_pipeline.c \
 *       -I external/mpp/inc/ -lrockchip_mpp -I external/linux-rga/include/ -lrga
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <linux/videodev2.h>
#include <time.h>
#include "rk_mpi.h"

/* ===== 配置 ===== */
#define ISP_DEV         "/dev/video20"    /* ISP mainpath (根据实际调整) */
#define WIDTH           1920
#define HEIGHT          1080
#define NV12_SIZE       (WIDTH * HEIGHT * 3 / 2)
#define BUF_NUM         4
#define FRAME_COUNT     100               /* 编码 100 帧 */

struct v4l2_buf {
    void *start;
    size_t length;
};

/* ===== V4L2 采集部分 ===== */

static int v4l2_open_and_setup(int *fd_out)
{
    int fd = open(ISP_DEV, O_RDWR);
    if (fd < 0) { perror("open ISP"); return -1; }

    /* QUERYCAP */
    struct v4l2_capability cap;
    ioctl(fd, VIDIOC_QUERYCAP, &cap);
    printf("[V4L2] driver=%s card=%s\n", cap.driver, cap.card);

    /* S_FMT: NV12 多平面 */
    struct v4l2_format fmt;
    memset(&fmt, 0, sizeof(fmt));
    fmt.type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
    fmt.fmt.pix_mp.width = WIDTH;
    fmt.fmt.pix_mp.height = HEIGHT;
    fmt.fmt.pix_mp.pixelformat = V4L2_PIX_FMT_NV12;
    fmt.fmt.pix_mp.num_planes = 1;
    if (ioctl(fd, VIDIOC_S_FMT, &fmt) < 0) {
        perror("S_FMT"); close(fd); return -1;
    }
    printf("[V4L2] fmt=%dx%d NV12\n", fmt.fmt.pix_mp.width, fmt.fmt.pix_mp.height);

    /* REQBUFS */
    struct v4l2_requestbuffers req;
    memset(&req, 0, sizeof(req));
    req.count = BUF_NUM;
    req.type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
    req.memory = V4L2_MEMORY_MMAP;
    ioctl(fd, VIDIOC_REQBUFS, &req);

    /* QUERYBUF + MMAP + QBUF */
    struct v4l2_buf bufs[BUF_NUM];
    for (int i = 0; i < BUF_NUM; i++) {
        struct v4l2_buffer buf;
        struct v4l2_plane planes[1];
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
            PROT_READ | PROT_WRITE, MAP_SHARED, fd, planes[0].m.mem_offset);

        buf.m.planes = planes;
        ioctl(fd, VIDIOC_QBUF, &buf);
    }
    printf("[V4L2] %d buffers mapped\n", BUF_NUM);

    *fd_out = fd;
    return 0;
}

static int v4l2_streamon(int fd)
{
    enum v4l2_buf_type type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
    return ioctl(fd, VIDIOC_STREAMON, &type);
}

static int v4l2_dqbuf(int fd, int *index, size_t *bytesused)
{
    struct v4l2_buffer buf;
    struct v4l2_plane planes[1];
    memset(&buf, 0, sizeof(buf));
    memset(planes, 0, sizeof(planes));
    buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
    buf.memory = V4L2_MEMORY_MMAP;
    buf.m.planes = planes;
    buf.length = 1;
    if (ioctl(fd, VIDIOC_DQBUF, &buf) < 0) {
        perror("DQBUF"); return -1;
    }
    *index = buf.index;
    *bytesused = planes[0].bytesused;
    return 0;
}

static int v4l2_qbuf(int fd, int index)
{
    struct v4l2_buffer buf;
    struct v4l2_plane planes[1];
    memset(&buf, 0, sizeof(buf));
    memset(planes, 0, sizeof(planes));
    buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
    buf.memory = V4L2_MEMORY_MMAP;
    buf.index = index;
    buf.m.planes = planes;
    buf.length = 1;
    return ioctl(fd, VIDIOC_QBUF, &buf);
}

/* ===== MPP 编码部分 ===== */

static int mpp_setup(MppCtx *ctx, MppApi **mpi)
{
    MppEncCfg cfg;

    if (mpp_create(ctx, mpi) < 0) { printf("[MPP] create failed\n"); return -1; }
    if (mpp_init(*ctx, MPP_CTX_ENC, MPP_VIDEO_CodingHEVC) < 0) {
        printf("[MPP] init failed\n"); return -1;
    }

    mpp_enc_cfg_init(&cfg);
    mpp_enc_cfg_set_s32(cfg, "prep:width", WIDTH);
    mpp_enc_cfg_set_s32(cfg, "prep:height", HEIGHT);
    mpp_enc_cfg_set_s32(cfg, "prep:hor_stride", WIDTH);
    mpp_enc_cfg_set_s32(cfg, "prep:ver_stride", HEIGHT);
    mpp_enc_cfg_set_s32(cfg, "prep:format", MPP_FMT_YUV420SP);
    mpp_enc_cfg_set_s32(cfg, "rc:mode", MPP_ENC_RC_MODE_CBR);
    mpp_enc_cfg_set_s32(cfg, "rc:bps_target", 8000000);
    mpp_enc_cfg_set_s32(cfg, "rc:fps_in_num", 30);
    mpp_enc_cfg_set_s32(cfg, "rc:gop", 30);
    mpp_enc_cfg_set_s32(cfg, "codec:type", MPP_VIDEO_CodingHEVC);
    (*mpi)->control(*ctx, MPP_ENC_SET_CFG, cfg);
    mpp_enc_cfg_deinit(cfg);

    printf("[MPP] H.265 encoder: %dx%d CBR 8Mbps\n", WIDTH, HEIGHT);
    return 0;
}

/* ===== 主流程 ===== */

int main(int argc, char **argv)
{
    const char *output = (argc > 1) ? argv[1] : "/tmp/camera.h265";
    int vfd, ret;
    MppCtx mpp_ctx;
    MppApi *mpi;
    struct v4l2_buf bufs[BUF_NUM];
    struct timespec ts_start, ts_end;
    long total_bytes = 0;

    printf("=== RV1126B Camera Pipeline ===\n");
    printf("Output: %s\n", output);

    /* 1. V4L2 采集初始化 */
    ret = v4l2_open_and_setup(&vfd);
    if (ret) return 1;

    /* 2. MPP 编码初始化 */
    ret = mpp_setup(&mpp_ctx, &mpi);
    if (ret) return 1;

    /* 3. 获取 H.265 SPS/PPS 头 */
    FILE *fp = fopen(output, "wb");
    MppPacket header = NULL;
    mpi->control(mpp_ctx, MPP_ENC_GET_EXTRA_INFO, &header);
    if (header) {
        fwrite(mpp_packet_get_data(header), 1,
               mpp_packet_get_length(header), fp);
        total_bytes += mpp_packet_get_length(header);
        printf("[HDR] SPS/PPS: %zu bytes\n", mpp_packet_get_length(header));
    }

    /* 4. 启动采集 */
    v4l2_streamon(vfd);
    printf("[V4L2] streaming started\n");

    /* 5. 主循环: 采集 → 编码 */
    clock_gettime(CLOCK_MONOTONIC, &ts_start);

    for (int i = 0; i < FRAME_COUNT; i++) {
        int buf_index;
        size_t bytesused;

        /* 5a. DQBUF — 取一帧 NV12 */
        if (v4l2_dqbuf(vfd, &buf_index, &bytesused) < 0) break;

        /* 5b. 准备 MPP 帧 */
        MppFrame frame;
        MppBuffer mpp_buf;
        mpp_frame_init(&frame);
        mpp_frame_set_width(frame, WIDTH);
        mpp_frame_set_height(frame, HEIGHT);
        mpp_frame_set_hor_stride(frame, WIDTH);
        mpp_frame_set_ver_stride(frame, HEIGHT);
        mpp_frame_set_fmt(frame, MPP_FMT_YUV420SP);
        mpp_buffer_get(NULL, &mpp_buf, NV12_SIZE);
        /* 注意: 生产环境用 dmabuf 零拷贝, 这里简化用 memcpy */
        memcpy(mpp_buffer_get_ptr(mpp_buf), bufs[buf_index].start, NV12_SIZE);
        mpp_frame_set_buffer(frame, mpp_buf);

        /* 5c. 编码 */
        mpi->encode_put_frame(mpp_ctx, frame);

        /* 5d. 获取码流 */
        MppPacket packet = NULL;
        mpi->encode_get_packet(mpp_ctx, &packet);
        if (packet) {
            void *data = mpp_packet_get_data(packet);
            size_t len = mpp_packet_get_length(packet);
            fwrite(data, 1, len, fp);
            total_bytes += len;
            mpp_packet_deinit(&packet);
        }

        /* 5e. 释放资源 */
        mpp_buffer_put(mpp_buf);
        mpp_frame_deinit(&frame);

        /* 5f. buffer 重新入队 */
        v4l2_qbuf(vfd, buf_index);

        if (i % 10 == 0)
            printf("[FRAME] %d/%d encoded\n", i, FRAME_COUNT);
    }

    clock_gettime(CLOCK_MONOTONIC, &ts_end);
    double elapsed = (ts_end.tv_sec - ts_start.tv_sec) +
                     (ts_end.tv_nsec - ts_start.tv_nsec) / 1e9;

    /* 6. 停止 + 清理 */
    enum v4l2_buf_type type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE;
    ioctl(vfd, VIDIOC_STREAMOFF, &type);
    mpp_destroy(mpp_ctx);
    fclose(fp);
    close(vfd);

    /* 7. 性能统计 */
    printf("\n=== Performance ===\n");
    printf("Frames: %d\n", FRAME_COUNT);
    printf("Time: %.2f s\n", elapsed);
    printf("FPS: %.1f\n", FRAME_COUNT / elapsed);
    printf("Output: %s (%ld bytes)\n", output, total_bytes);
    printf("Avg bitrate: %.1f Mbps\n",
           (total_bytes * 8.0 / elapsed) / 1e6);

    return 0;
}
```

> **注意**：这个程序是简化版（用 memcpy 而非 dmabuf 零拷贝）。生产代码用 Rockit 框架 (`RK_MPI_SYS_Bind`) 自动连接 VI→VENC，无需手动传递帧。

---

## 三、实验 1：运行完整管线

### 3.1 前提条件

```bash
# 1. DTS 中相机管线已启用 (camera DTS overlay)
# 2. RKAIQ 3A server 已启动
# 3. MPP 库已安装
# 4. media-ctl / v4l2-ctl 可用

# 验证管线拓扑
media-ctl -d /dev/media0 -p | grep -E "sensor|dphy|csi2|cif|isp"

# 验证 3A server
ps | grep rkaiq

# 验证 MPP
ls /lib/librockchip_mpp*
```

### 3.2 配置管线 + 运行

```bash
# 1. 用 media-ctl 配置管线 (格式沿管线传播)
media-ctl -d /dev/media0 --set-v4l2 '"sc450ai":0[fmt:SBGGR10_1X10/1920x1080]'
media-ctl -d /dev/media0 --set-v4l2 '"csi2_dphy3":0[fmt:SBGGR10_1X10/1920x1080]'
media-ctl -d /dev/media0 --set-v4l2 '"mipi2_csi2":0[fmt:SBGGR10_1X10/1920x1080]'
media-ctl -d /dev/media0 --set-v4l2 '"rkcif_mipi_lvds2":0[fmt:SBGGR10_1X10/1920x1080]'
# 启用 link
media-ctl -d /dev/media0 -l '"sc450ai":0 -> "csi2_dphy3":0 [1]'
# ... 其他 link

# 2. 运行管线
sudo /tmp/camera_pipeline /tmp/camera.h265

# 预期输出:
# === RV1126B Camera Pipeline ===
# [V4L2] driver=rkisp card=rkisp
# [V4L2] fmt=1920x1080 NV12
# [V4L2] 4 buffers mapped
# [MPP] H.265 encoder: 1920x1080 CBR 8Mbps
# [HDR] SPS/PPS: 120 bytes
# [V4L2] streaming started
# [FRAME] 0/100 encoded
# [FRAME] 10/100 encoded
# ...
# === Performance ===
# Frames: 100
# Time: 3.35 s
# FPS: 29.9
# Output: /tmp/camera.h265 (3341234 bytes)
# Avg bitrate: 8.0 Mbps
```

### 3.3 验证输出

```bash
# 拷回 PC 播放
scp rooter@192.168.1.109:/tmp/camera.h265 .

# 用 ffplay 播放
ffplay -f h265 camera.h265

# 查看码流信息
ffprobe -i camera.h265
# 预期: H.265, 1920x1080, 30fps, 8Mbps

# 检查画质
# - 白平衡是否正确 (白色物体看起来是白色)
# - 曝光是否合适 (不过亮/过暗)
# - 降噪是否适当 (噪点 vs涂抹感)
# - 锐度是否合适
```

---

## 四、实验 2：性能测量

### 4.1 端到端延迟测量

```bash
# 用 Ftrace 追踪各阶段时间:
echo function_graph | sudo tee /sys/kernel/tracing/current_tracer
echo '*rkisp*|*rkcif*|*mpp*|*rkvenc*' | sudo tee /sys/kernel/tracing/set_ftrace_filter
echo 128 | sudo tee /sys/kernel/tracing/max_graph_depth

echo | sudo tee /sys/kernel/tracing/trace
sudo /tmp/camera_pipeline /tmp/camera.h265
sudo cat /sys/kernel/tracing/trace > /tmp/pipeline_trace.log
echo nop | sudo tee /sys/kernel/tracing/current_tracer
```

### 4.2 延迟分解

| 阶段 | 预期耗时 | 说明 |
|------|---------|------|
| Sensor 曝光 | 10-30ms | 取决于 AE 设置 |
| Sensor → CSI → CIF | ~1ms | MIPI 传输 |
| ISP 处理 | ~2-5ms | Raw → NV12 |
| V4L2 DQBUF | ~0.1ms | 从内核取帧 |
| MPP 编码 | ~3-8ms | NV12 → H.265 |
| **端到端** | **~20-45ms** | Sensor 到码流 |

> **大疆图传对比**：大疆 O4 端到端 <20ms，通过在线模式 (CIF→ISP 不落 DDR) + 硬件直连 (VI→VENC 无拷贝) 实现。

---

## 五、实验 3：对比不同编码参数

```bash
# 1. 低延迟 (图传场景)
# 修改程序: GOP=1, CBR=20Mbps, H.264 Baseline
# 预期: 延迟最低, 文件最大

# 2. 高压缩 (录像场景)
# GOP=60, CBR=4Mbps, H.265
# 预期: 延迟较高, 文件最小

# 3. 固定质量
# FIX_QP, qp=28
# 预期: 画质一致, 码率随场景变化

# 对比三组输出的:
# - 文件大小
# - ffprobe 码率信息
# - 主观画质
```

---

## 六、项目设计文档模板

```markdown
# RV1126B 相机管线设计文档

## 1. 概述
- 功能: Sensor 采集 → ISP 处理 → 3A → H.265 编码 → 文件
- 平台: RV1126B (ISP V35 + VEPU511)
- 输出: 1920x1080@30fps H.265, 8Mbps CBR

## 2. 管线架构
  Sensor → DPHY → CSI-2 → CIF → ISP → V4L2 → MPP → H.265
                                       ↑
                                   RKAIQ 3A

## 3. 技术选型
- 采集: V4L2 MPLANE + mmap (生产环境用 dmabuf 零拷贝)
- 编码: MPP MPI (生产环境用 Rockit RK_MPI_Bind)
- 3A: RKAIQ uAPI2 (独立 3A server)

## 4. 性能指标
- 帧率: 30fps
- 端到端延迟: ~30ms
- CPU 占用: <10% (硬件加速)
- 码率: 8Mbps (CBR)

## 5. 优化方向
- 零拷贝: V4L2 dmabuf → MPP buffer (省一次 memcpy)
- 在线模式: CIF → ISP 不落 DDR
- Rockit: RK_MPI_SYS_Bind 自动管线
- 多路: 主码流 + 预览 + 缩略图
```

---

## 七、Code Review 检查清单

| 检查项 | 标准 | 通过 |
|--------|------|------|
| V4L2 ioctl 返回值检查 | 所有 ioctl 检查 < 0 | |
| buffer 释放 | munmap + close 配对 | |
| MPP 资源释放 | mpp_destroy + buffer_put | |
| 错误处理 | 异常路径有清理 | |
| 内存对齐 | hor_stride/ver_stride 正确 | |
| 帧率控制 | 不超过 Sensor 最大帧率 | |
| 编码参数 | GOP/码率/QP 合理 | |
| 文件完整性 | SPS/PPS + 帧数据完整 | |

---

## 八、面试展示要点

### 8.1 能说清楚的完整链路

```
面试官: "描述一下相机从 Sensor 到编码输出的完整流程"

回答框架:
1. Sensor 通过 I2C 配置寄存器, 输出 RAW Bayer 数据
2. RAW 数据经 MIPI CSI-2 (D-PHY) 传到 SoC 的 RKCIF
3. RKCIF 通过 SDITF 在线传给 ISP V35
4. ISP 做 BLC→LSC→AWB→Demosaic→NR→Sharpen→CSC, 输出 NV12
5. RKAIQ 3A server 每帧读 ISP stats, 计算 AE/AWB 参数, 写回 ISP params
6. NV12 帧通过 V4L2 DQBUF 取出, 送入 MPP
7. VEPU511 硬件编码为 H.265 码流
8. 码流写入文件或推流

关键数字:
- Sensor RAW10 1920x1080@30fps = ~62MB/s 带宽
- MIPI CSI-2 4-Lane ~4Gbps 够用
- ISP 处理 ~3-5ms/帧
- MPP 编码 ~3-8ms/帧
- 端到端延迟 ~20-40ms
- CPU 占用 <10% (硬件加速)
```

### 8.2 能回答的深度问题

| 问题 | 回答要点 |
|------|---------|
| 零拷贝如何实现? | V4L2 dmabuf → MPP buffer 共享物理内存, 无 memcpy |
| 3A 如何保证实时? | 3A server 每帧处理 <10ms, 远小于 33ms 帧间隔 |
| 多路输出怎么做? | VPSS 从 ISP mainpath 分出多路, 不同分辨率/格式 |
| 图传低延迟怎么做? | 在线模式 (CIF→ISP 不落 DDR) + 小 GOP + Slice 分割 |
| 夜景画质怎么优化? | 提高 AE 增益 + AI 降噪 (AIBNR) + 长曝光 + 多帧融合 |

---

## 九、学习路线完成总结

### 完成状态

| 阶段 | 文档 | JD 覆盖 | 状态 |
|------|------|---------|------|
| 一 | bsp-boot-flow | Bootloader/启动 | ✅ |
| 二 | bsp-device-model-dtb | 设备模型/设备树 | ✅ |
| 三 | bsp-interrupt-concurrency | 中断/并发 | ✅ |
| 四 | bsp-peripheral-drivers | I2C/SPI/UART | ✅ |
| 五 | bsp-power-management | 电源管理 | ✅ |
| 六 | bsp-capstone-driver | BSP 综合 | ✅ |
| 七 | bsp-csi-sensor-driver | MIPI CSI/Sensor | ✅ |
| 八 | bsp-v4l2-isp-pipeline | V4L2/ISP/Media Controller | ✅ |
| 九 | bsp-isp-3a-rkaiq | 3A/RKAIQ/IQ | ✅ |
| 十 | bsp-mpp-codec-pipeline | MPP/Rockit/H.265 | ✅ |
| 十一 | bsp-rga-postprocess | RGA/EIS/OSD | ✅ |
| 十二 | bsp-camera-capstone | 相机管线综合 | ✅ |
| 附录A | kernel-debug-env | Ftrace/Lockdep | ✅ |
| 附录B | v4l2-isp-deep-dive | UVC 实验 | ✅ |
| 附录C | mpp-hardware-codec | MPP 实验 | ✅ |

### 大疆相机岗位覆盖率

| JD 要求 | 覆盖 |
|---------|------|
| 嵌入式Linux系统架构 | ✅ |
| Bootloader/Kernel移植 | ✅ |
| C语言 + 调试 | ✅ |
| UART/I2C/SPI协议 | ✅ |
| USB协议 | ✅ |
| Linux设备模型 | ✅ |
| 中断处理 | ✅ |
| 电源管理 | ✅ |
| **V4L2/Media Controller** | ✅ |
| **MIPI CSI/Sensor驱动** | ✅ |
| **ISP Pipeline/3A** | ✅ |
| **硬件编解码** | ✅ |
| **图像处理/EIS/OSD** | ✅ |
| 驱动设计编码测试 | ✅ |
| 技术文档 | ✅ |

---

## 相关笔记

- [[MOC-嵌入式Linux]] — 学习路线总览
- [[dji-bsp-analysis]] — 大疆对标分析
- [[bsp-csi-sensor-driver]] — 阶段七
- [[bsp-v4l2-isp-pipeline]] — 阶段八
- [[bsp-isp-3a-rkaiq]] — 阶段九
- [[bsp-mpp-codec-pipeline]] — 阶段十
- [[bsp-rga-postprocess]] — 阶段十一
