---
tags:
  - embedded-linux
  - mpp
  - hardware-codec
  - h264
  - rga
  - rockchip
  - video-encode
category: embedded-linux
created: 2026-06-18
updated: 2026-06-18
status: active
soc: Rockchip RV1126B
kernel: Linux 6.1.141
mpp_version: 1.0.11
codec_hw: VEPU511 (encode) + VDPU384A/VDPU720 (decode)
---

# 阶段三：MPP 硬件编解码

> **MPP (Media Process Platform)** 是 Rockchip 的通用媒体处理软件平台，为上层应用提供统一的硬件编解码接口（MPI）。RV1126B 搭载 VEPU511 编码器和 VDPU384A/VDPU720 解码器，支持 H.264/H.265/JPEG 编解码。
>
> 相比 CPU 软件编码（如 x264），硬件编码可将编码功耗降低 10~20 倍，同时释放 CPU 核心处理其他任务。

---

## 一、MPP 系统架构

### 1.1 层次结构

```
┌─────────────────────────────────────┐
│          应用层 (用户代码)            │
│  mpi_enc_test / 自定义编码程序       │
├─────────────────────────────────────┤
│          MPI 接口 (rk_mpi.h)         │
│  mpp_create / mpp_init / control    │
│  encode_put_frame / encode_get_packet│
├─────────────────────────────────────┤
│          MPP 用户态库               │
│  librockchip_mpp.so                 │
│  ├─ OSAL (系统抽象层)               │
│  ├─ HAL (硬件抽象层)                │
│  ├─ Codec (H.264/H.265 编码器)      │
│  └─ kmpp (内核 mpp_service 封装)    │
├─────────────────────────────────────┤
│       内核驱动 (mpp_service)         │
│  /dev/mpp_service                   │
│  ├─ rkvenc (RKV 编码器)             │
│  ├─ rkvdec (RKV 解码器)             │
│  ├─ vepu (VPU 编码器)               │
│  └─ vdpu (VPU 解码器)               │
├─────────────────────────────────────┤
│        硬件编解码 IP 核              │
│  VEPU511 (编码) + VDPU384A (解码)    │
└─────────────────────────────────────┘
```

### 1.2 RV1126B 硬件编解码能力

根据 `external/mpp/mpp/soc.cmake` 中 RV1126B 的 SoC 配置：

```
SoC:          RV1126B
硬件模块:     VDPU384A, VDPU720, VEPU511
解码支持:     H.264, H.265, JPEG
编码支持:     H.264, H.265, JPEG
```

| 操作 | 硬件模块 | 最大分辨率 | 性能参考 |
|------|---------|-----------|---------|
| H.264 编码 | VEPU511 | 1920x1080 | ~30fps+ |
| H.265 编码 | VEPU511 | 1920x1080 | ~30fps+ |
| JPEG 编码 | VEPU511 | 1920x1080 | ~30fps+ |
| H.264 解码 | VDPU384A | 1920x1080 | 60fps+ |
| H.265 解码 | VDPU384A | 1920x1080 | 60fps+ |
| JPEG 解码 | VDPU720 | 1920x1080 | 60fps+ |

### 1.3 内核驱动确认

```bash
# 板端检查 mpp_service 设备
ls -la /dev/mpp_service
# 预期: crw-rw---- 1 root video 10, 58 ...

cat /proc/mpp_service
# 预期输出各硬件模块的状态和版本信息

# 检查内核模块
lsmod | grep mpp
lsmod | grep rkvdec
lsmod | grep rkvenc
```

### 1.4 MPP 库确认

```bash
# 检查 MPP 库
ls -la /lib/librockchip_mpp* /usr/lib/librockchip_mpp*
# 预期: librockchip_mpp.so / librockchip_mpp.so.1

# 查看版本
strings /lib/librockchip_mpp.so.1 | grep "MPP_VERSION"
```

---

## 二、MPI 编程模型

### 2.1 MPP 编码流程

```
mpp_create()                     ← 创建 MPP 实例 + 获取 MPI 接口
  │
mpp_init(MPP_CTX_ENC, coding)    ← 初始化编码器 (H.264/H.265)
  │
mpi->control(MPP_ENC_SET_CFG)    ← 配置编码参数 (分辨率、码率、帧率、GOP)
  │
mpi->control(MPP_ENC_GET_EXTRA_INFO)  ← 获取 SPS/PPS 头信息
  │
  ┌─────────────────────────────────────┐
  │  encode_put_frame(frame)   ← 输入 YUV 帧
  │  encode_get_packet(packet)  ← 获取 H.264 码流包
  │  (循环直到所有帧编码完成)
  └─────────────────────────────────────┘
  │
mpp_destroy()                    ← 销毁 MPP 实例
```

### 2.2 核心数据结构

| 结构体 | 作用 | 关键操作 |
|--------|------|---------|
| `MppCtx` | MPP 实例上下文 | `mpp_create` / `mpp_init` / `mpp_destroy` |
| `MppApi` | MPI 函数指针表 | 通过 `mpp_create` 获取，包含 `encode_put_frame` / `encode_get_packet` / `control` |
| `MppFrame` | 图像帧（输入） | `mpp_frame_init` / `mpp_frame_set_*` / `mpp_frame_deinit` |
| `MppPacket` | 码流包（输出） | `mpp_packet_init` / `mpp_packet_get_*` / `mpp_packet_deinit` |
| `MppBuffer` | 硬件缓冲区 | `mpp_buffer_get` / `mpp_buffer_put` |
| `MppEncCfg` | 编码器配置 | `mpp_enc_cfg_init` / `mpp_enc_cfg_set_*` / `mpp_enc_cfg_deinit` |

### 2.3 关键编码配置参数

| 配置键 | 说明 | 常用值 |
|--------|------|--------|
| `rc:mode` | 码率控制模式 | `MPP_ENC_RC_MODE_CBR` (固定码率) |
| `rc:bps_target` | 目标码率 (bps) | `4000000` (4Mbps) |
| `rc:fps_in_num` | 输入帧率分子 | `30` |
| `rc:fps_in_denom` | 输入帧率分母 | `1` |
| `rc:gop` | GOP 大小 (I帧间隔) | `30` (每秒一个I帧) |
| `prep:width` | 图像宽度 (像素) | `1920` |
| `prep:height` | 图像高度 (像素) | `1080` |
| `prep:hor_stride` | 水平跨度 (字节) | `1920` (NV12) |
| `prep:ver_stride` | 垂直跨度 | `1080` |
| `prep:format` | 像素格式 | `MPP_FMT_YUV420SP` (NV12) |
| `codec:type` | 编码类型 | `MPP_VIDEO_CodingAVC` (H.264) |
| `h264:profile` | H.264 档次 | `100` (High) |
| `h264:level` | H.264 级别 | `41` (Level 4.1) |

### 2.4 YUYV → NV12 转换

UVC 相机输出 YUYV 格式，而 MPP 硬件编码器输入需要 NV12。NV12 布局：

```
[Y 平面]: W × H 字节 (每个像素 1 字节 Y)
[UV 平面]: W × H/2 字节 (每 2×2 像素共用 1 组 UV，交错存储)
```

YUYV → NV12 转换公式（简化直接拷贝 Y，UV 取平均）：

```c
// 伪代码：逐像素转换
for (y = 0; y < height; y++) {
    for (x = 0; x < width; x += 2) {
        // YUYV 布局: [Y0][U0][Y1][V0]
        dst_y[y * width + x]     = src[(y * width + x) * 2];
        dst_y[y * width + x + 1] = src[(y * width + x) * 2 + 2];

        // UV 每 2 行取一次 (隔行降采样)
        if (y % 2 == 0) {
            dst_uv[(y/2) * width + x]     = src[(y * width + x) * 2 + 1]; // U
            dst_uv[(y/2) * width + x + 1] = src[(y * width + x) * 2 + 3]; // V
        }
    }
}
```

---

## 三、实验 1：编码一帧 YUYV 到 H.264

### 3.1 实验目标

用 MPP 硬件编码器把阶段二抓到的 `frame.yuv` (YUYV 1920x1080) 编码成 H.264 码流，保存为 `output.h264`。

### 3.2 程序源码 (mpp_enc_oneframe.c)

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>

#include "rk_mpi.h"

#define YUYV_SIZE (1920 * 1080 * 2)
#define WIDTH     1920
#define HEIGHT    1080

/* YUYV → NV12 软件转换 */
static void yuyv_to_nv12(unsigned char *yuyv, unsigned char *nv12,
                          int width, int height)
{
    unsigned char *y_plane = nv12;
    unsigned char *uv_plane = nv12 + width * height;

    for (int y = 0; y < height; y++) {
        for (int x = 0; x < width; x += 2) {
            int src_idx = (y * width + x) * 2;
            int dst_y   = y * width + x;

            y_plane[dst_y]     = yuyv[src_idx];       /* Y0 */
            y_plane[dst_y + 1] = yuyv[src_idx + 2];   /* Y1 */

            if (y % 2 == 0) {
                int dst_uv = (y / 2) * width + x;
                uv_plane[dst_uv]     = yuyv[src_idx + 1];  /* U */
                uv_plane[dst_uv + 1] = yuyv[src_idx + 3];  /* V */
            }
        }
    }
}

int main(int argc, char **argv)
{
    const char *input = (argc > 1) ? argv[1] : "frame.yuv";
    const char *output = (argc > 2) ? argv[2] : "output.h264";
    int ret;

    /* 1. 读取 YUYV 文件 */
    FILE *fp = fopen(input, "rb");
    if (!fp) { perror("fopen input"); return 1; }
    unsigned char *yuyv = malloc(YUYV_SIZE);
    fread(yuyv, 1, YUYV_SIZE, fp);
    fclose(fp);
    printf("Read %s (%d bytes)\n", input, YUYV_SIZE);

    /* 2. 创建 MPP 实例 */
    MppCtx ctx = NULL;
    MppApi *mpi = NULL;
    ret = mpp_create(&ctx, &mpi);
    if (ret) { fprintf(stderr, "mpp_create failed\n"); return 1; }
    printf("MPP created\n");

    /* 3. 初始化为 H.264 编码器 */
    ret = mpi->control(ctx, MPP_SET_OUTPUT_TIMEOUT, NULL);
    MppCodingType coding = MPP_VIDEO_CodingAVC;
    ret = mpp_init(ctx, MPP_CTX_ENC, coding);
    if (ret) { fprintf(stderr, "mpp_init failed\n"); return 1; }
    printf("MPP init as H264 encoder\n");

    /* 4. 配置编码参数 */
    MppEncCfg cfg = NULL;
    mpp_enc_cfg_init(&cfg);

    /* 基本参数 */
    mpp_enc_cfg_set_s32(cfg, "prep:width",       WIDTH);
    mpp_enc_cfg_set_s32(cfg, "prep:height",      HEIGHT);
    mpp_enc_cfg_set_s32(cfg, "prep:hor_stride",  WIDTH);
    mpp_enc_cfg_set_s32(cfg, "prep:ver_stride",  HEIGHT);
    mpp_enc_cfg_set_s32(cfg, "prep:format",      MPP_FMT_YUV420SP);
    mpp_enc_cfg_set_s32(cfg, "prep:rotation",    MPP_ENC_ROT_0);

    /* 码率控制: CBR 4Mbps */
    mpp_enc_cfg_set_s32(cfg, "rc:mode",          MPP_ENC_RC_MODE_CBR);
    mpp_enc_cfg_set_s32(cfg, "rc:bps_target",    4000000);
    mpp_enc_cfg_set_s32(cfg, "rc:bps_max",       4000000);
    mpp_enc_cfg_set_s32(cfg, "rc:bps_min",       4000000);
    mpp_enc_cfg_set_s32(cfg, "rc:fps_in_num",    30);
    mpp_enc_cfg_set_s32(cfg, "rc:fps_in_denom",  1);
    mpp_enc_cfg_set_s32(cfg, "rc:fps_out_num",   30);
    mpp_enc_cfg_set_s32(cfg, "rc:fps_out_denom", 1);
    mpp_enc_cfg_set_s32(cfg, "rc:gop",           30);

    /* H.264 编码参数 */
    mpp_enc_cfg_set_s32(cfg, "codec:type",       MPP_VIDEO_CodingAVC);
    mpp_enc_cfg_set_s32(cfg, "h264:profile",     100);  /* High Profile */
    mpp_enc_cfg_set_s32(cfg, "h264:level",       41);   /* Level 4.1 */
    mpp_enc_cfg_set_s32(cfg, "h264:cabac_en",    1);

    ret = mpi->control(ctx, MPP_ENC_SET_CFG, cfg);
    if (ret) { fprintf(stderr, "MPP_ENC_SET_CFG failed %d\n", ret); return 1; }
    printf("Encoder configured: CBR 4Mbps, H264 High@L4.1\n");

    /* 5. 获取 SPS/PPS 头信息 */
    MppPacket header = NULL;
    mpi->control(ctx, MPP_ENC_GET_EXTRA_INFO, &header);
    if (header) {
        void *hdr_data = mpp_packet_get_data(header);
        size_t hdr_len = mpp_packet_get_length(header);
        printf("SPS/PPS header: %zu bytes\n", hdr_len);
        FILE *hdr_fp = fopen(output, "wb");
        fwrite(hdr_data, 1, hdr_len, hdr_fp);
        fclose(hdr_fp);
        mpp_packet_deinit(&header);
    }

    /* 6. 准备输入帧 (NV12) */
    unsigned char *nv12 = malloc(WIDTH * HEIGHT * 3 / 2);
    yuyv_to_nv12(yuyv, nv12, WIDTH, HEIGHT);
    free(yuyv);

    MppFrame frame = NULL;
    mpp_frame_init(&frame);
    mpp_frame_set_width(frame, WIDTH);
    mpp_frame_set_height(frame, HEIGHT);
    mpp_frame_set_hor_stride(frame, WIDTH);
    mpp_frame_set_ver_stride(frame, HEIGHT);
    mpp_frame_set_fmt(frame, MPP_FMT_YUV420SP);
    mpp_frame_set_pts(frame, 0);

    /* 分配 MPP buffer 并拷贝 NV12 数据 */
    MppBuffer buf = NULL;
    mpp_buffer_get(NULL, &buf, WIDTH * HEIGHT * 3 / 2);
    memcpy(mpp_buffer_get_ptr(buf), nv12, WIDTH * HEIGHT * 3 / 2);
    free(nv12);
    mpp_frame_set_buffer(frame, buf);
    printf("Input frame prepared (NV12, %dx%d)\n", WIDTH, HEIGHT);

    /* 7. 编码 (同步接口) */
    ret = mpi->encode_put_frame(ctx, frame);
    if (ret) { fprintf(stderr, "encode_put_frame failed %d\n", ret); return 1; }

    MppPacket packet = NULL;
    ret = mpi->encode_get_packet(ctx, &packet);
    if (ret) { fprintf(stderr, "encode_get_packet failed %d\n", ret); return 1; }

    /* 8. 保存编码输出 */
    if (packet) {
        void *data = mpp_packet_get_data(packet);
        size_t len = mpp_packet_get_length(packet);
        printf("Encoded frame: %zu bytes\n", len);

        FILE *out_fp = fopen(output, "ab");
        fwrite(data, 1, len, out_fp);
        fclose(out_fp);

        mpp_packet_deinit(&packet);
    }
    printf("Saved to %s\n", output);

    /* 9. 清理 */
    mpp_frame_deinit(&frame);
    mpp_buffer_put(buf);
    mpp_enc_cfg_deinit(cfg);
    mpp_destroy(ctx);
    return 0;
}
```

### 3.3 编译 & 运行

```bash
# PC 端交叉编译
export PATH=$PWD/prebuilts/gcc/linux-x86/aarch64/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin:$PATH
aarch64-none-linux-gnu-gcc -static -o mpp_enc_oneframe mpp_enc_oneframe.c \
    -I external/mpp/inc/ -L external/mpp/lib/ -lrockchip_mpp

# 如果 SDK 中无预编译的 MPP 库，则需先构建 buildroot 获取:
# 方法 1: 从板子拷贝
# scp rooter@192.168.1.109:/lib/librockchip_mpp.so.1 /tmp/

# 方法 2: 使用 SDK 中 NPU 示例附带的 MPP 库
# cp external/rknpu2/examples/3rdparty/mpp/Linux/aarch64/librockchip_mpp.so* /tmp/

# 部署
scp mpp_enc_oneframe rooter@192.168.1.109:/tmp/

# 板端运行
sudo /tmp/mpp_enc_oneframe /tmp/frame.yuv /tmp/output.h264
```

### 3.4 验证结果

```bash
# 查看文件大小
ls -la /tmp/output.h264
# 预期: 几千字节 (单帧 I-frame, 4Mbps 码率)

# 用 hexdump 查看码流头部
hexdump -C /tmp/output.h264 | head -20
# 预期开头: 00 00 00 01 67 ... (SPS)
#           00 00 00 01 68 ... (PPS)
#           00 00 00 01 65 ... (IDR slice)

# scp 回 PC 用 ffplay 或 Elecard 播放检验
scp rooter@192.168.1.109:/tmp/output.h264 .
ffplay -f h264 output.h264
```

### 3.5 预期耗时

| 阶段 | 耗时 (估算) | 说明 |
|------|------------|------|
| YUYV→NV12 转换 | ~5-10ms | CPU 逐像素转换 |
| mpp_buffer_get + memcpy | ~1-2ms | 拷贝到 MPP buffer |
| encode_put_frame (阻塞) | ~3-8ms | 硬件编码 + 等待完成 |
| encode_get_packet | ~0.1ms | 取回编码包 |
| **总计** | **~10-20ms** | 远快于 x264 软件编码 (30-50ms) |

---

## 四、实验 2：用 Ftrace 追踪 MPP 编码路径

### 4.1 实验目标

用 Ftrace `function_graph` 追踪 MPP 编码调用的内核路径，理解硬件编码的驱动层次。

### 4.2 操作步骤

```bash
# 查看 mpp_service 相关可追踪函数
sudo cat /sys/kernel/tracing/available_filter_functions | grep -E "mpp_service|rkv|vepu|vdpu" | head -30

# 设置追踪
echo function_graph | sudo tee /sys/kernel/tracing/current_tracer
echo '*mpp*' | sudo tee /sys/kernel/tracing/set_ftrace_filter
echo '*rkv*' | sudo tee -a /sys/kernel/tracing/set_ftrace_filter
echo '*vepu*' | sudo tee -a /sys/kernel/tracing/set_ftrace_filter
echo '*encode*' | sudo tee -a /sys/kernel/tracing/set_ftrace_filter
echo 256 | sudo tee /sys/kernel/tracing/max_graph_depth

# 清缓冲区
echo | sudo tee /sys/kernel/tracing/trace

# 运行编码程序
sudo /tmp/mpp_enc_oneframe /tmp/frame.yuv /tmp/output.h264

# 保存追踪结果
sudo cat /sys/kernel/tracing/trace > /tmp/mpp_enc_trace.log
sudo wc -l /tmp/mpp_enc_trace.log

# 关追踪
echo nop | sudo tee /sys/kernel/tracing/current_tracer

# scp 回 PC 分析
scp rooter@192.168.1.109:/tmp/mpp_enc_trace.log .
```

### 4.3 分析重点

1. `mpp_service` 驱动从用户态 `ioctl` 到硬件寄存器操作的全路径
2. `rkv_enc` / `vepu` 编码器的硬件启动和中断处理
3. DMA 映射 / 解映射路径
4. 硬件编码 vs 软件 memcpy 的时间占比

### 4.4 预期调用链

```
用户态:
  mpi->encode_put_frame()
    → mpp_enc_put_frame() [用户态]
      → kmpp_venc_put_frame()
        → ioctl(fd, ...) ──→ 内核态

内核态:
  mpp_service_ioctl()
    → mpp_service_ioctl_default()
      → mpp_venc_process()
        → rkvenc_enc_one_frame() 或 vepu_enc_one_frame()
          → 配置硬件寄存器
          → 启动编码硬件
          → 等待 completion (硬件中断)
            → rkvenc_irq_handler() / vepu_irq_handler()
              → complete() 唤醒等待
          → 读取编码结果
  → 回到用户态, encode_get_packet() 获取码流
```

---

## 五、实验 3：编码性能对比 (MPP vs CPU)

### 5.1 实验目标

对比 MPP 硬件编码和纯 CPU 软件编码的吞吐量和码流质量。

### 5.2 CPU 软件编码 (x264)

```bash
# 检查板端是否有 x264
which x264 2>/dev/null || echo "x264 not installed"

# 如果有, 把 YUYV 转成 y4m 然后用 x264 编码
# 先用 ffmpeg 或手动转换 (这里用 python 生成 y4m 头)
python3 -c "
import sys
# 生成 NV12 的 y4m 头
header = 'YUV4MPEG2 W1920 H1080 F30:1 Ip A0:0 C420mpeg2\n'
with open('frame.yuv', 'rb') as f:
    raw = f.read()
with open('frame.y4m', 'wb') as f:
    f.write(header.encode())
    f.write(raw)
"
time x264 --input-depth 8 --input-csp i420 -o /tmp/cpu.h264 frame.y4m
```

### 5.3 对比结果记录

| 指标 | MPP 硬件编码 | x264 软件编码 (估算) |
|------|-------------|-------------------|
| 编码一帧耗时 | ~5ms | ~30-50ms |
| CPU 占用率 | <5% (硬件卸载) | ~100% (单核满) |
| 码率控制精度 | 硬件 CBR | 软件多遍 |
| 画面质量 (同码率) | 略低于 x264 | 更高 |
| 4Mbps 下 1080p 码流大小 | ~500KB (1帧 I) | ~500KB |
| 功耗 | ~0.3W | ~2-3W |

### 5.4 关键结论

- **MPP 硬件编码适合实时流**：低延迟、低功耗、CPU 几乎零占用
- **x264 适合录制存档**：同等码率下 PSNR 略高，但编码慢
- **实际产品通常混合使用**：预览流用硬件编码，录制用 x264 多遍编码

---

## 六、实验 4：MPP 多帧连续编码 (可选)

### 6.1 实验目标

模拟视频采集循环：从 UVC 相机抓多帧 YUYV，逐帧送入 MPP 编码器，输出 H.264 码流。

### 6.2 关键思路

```c
for (int i = 0; i < frame_count; i++) {
    // 1. 从 UVC 相机 DQBUF 一帧
    // 2. YUYV → NV12 转换
    // 3. MPP 编码
    mpi->encode_put_frame(ctx, frame);
    mpi->encode_get_packet(ctx, &packet);
    // 4. 写文件或推流
    fwrite(packet_data, 1, packet_len, fp);
    mpp_packet_deinit(&packet);
}
```

> **注意**：实际编码程序的 `encode_get_packet` 需要在循环中多次调用，直到返回 EOS 或超时。编码器内部可能有帧重排序延迟。

---

## 七、实验 5：RGA 2D 硬件加速 (可选)

### 7.1 RGA 简介

RGA (Raster Graphic Acceleration) 是 Rockchip 的 2D 图形硬件加速模块，支持：
- 图像缩放 (Resize)
- 格式转换 (Color Space Conversion)
- 旋转 / 镜像
- 裁剪

### 7.2 librga API 使用

```c
#include <im2d.h>
#include <rga.h>

// 配置源图像
rga_buffer_t src = wrapbuffer_virtualaddr(src_virt, width, height, RK_FORMAT_YUYV_422);
// 配置目标图像
rga_buffer_t dst = wrapbuffer_virtualaddr(dst_virt, width, height, RK_FORMAT_NV12);
// 执行格式转换 + 缩放
im_convert(src, dst);
```

### 7.3 用 RGA 替代 CPU 做 YUYV → NV12

```bash
# 检查板端 librga
ls -la /lib/librga* /usr/lib/librga*

# 编译 RGA 转换程序
aarch64-none-linux-gnu-gcc -static -o rga_convert rga_convert.c \
    -I external/linux-rga/include/ -lrga

# 运行
sudo /tmp/rga_convert /tmp/frame.yuv /tmp/frame_nv12.bin 1920 1080
```

> **性能对比**：CPU 逐像素转换 YUYV→NV12 在 1920x1080 上约 5-10ms，RGA 硬件完成仅需 ~0.5ms。

---

## 八、V4L2 + MPP 完整 pipeline (参考)

生产环境中典型运动相机 pipeline：

```
UVC Camera
    │
    ▼
V4L2 DQBUF (YUYV frame)
    │
    ├──→ MPP Encode (H.264)  →  RTSP 推流
    │
    └──→ RGA Resize (640x360)  →  MPP Encode (H.264)  →  本地录制
                                        │
                                        ▼
                                   MPP Decode (回放预览)
                                        │
                                        ▼
                                   DRM/KMS 显示
```

---

## 九、思考题

1. MPP 编码器输入为什么需要 NV12 格式而不是 YUYV？这和硬件编码器的内存读取方式有什么关系？

2. `encode_put_frame` 是一个阻塞函数，它会等待硬件编码完成才返回。这有什么好处？有什么缺点？如何解决？

3. 对比 Ftrace 中 MPP 编码路径和 V4L2 UVC 采集路径，两者的内核态执行时间分布有什么不同？为什么硬件编码器不需要类似 `uvc_alloc_urb_buffers` 的内存分配？

4. 如果你需要在 1920x1080@30fps 下连续编码，CPU 做 YUYV→NV12 转换 + MPP 编码，最可能出现的瓶颈在哪里？如何优化？

5. RGA 做格式转换和 CPU 做格式转换的本质区别是什么？RGA 转换后的数据在内存中有什么特殊要求才能被 MPP 硬件编码器直接使用？

---

## 十、踩坑记录

| 日期 | 问题 | 原因 | 解决方案 |
|------|------|------|----------|
| 2026-06-18 | mpp_create 失败 | MPP 库未安装或版本不匹配 | `ls /lib/librockchip_mpp*` 确认库存在 |
| 2026-06-18 | encode_put_frame 阻塞不返回 | 硬件编码器时钟未使能或驱动问题 | 检查 `dmesg` 中 mpp_service 相关错误 |
| 2026-06-18 | MPP 不支持 YUYV 输入 | 硬件编码器只接受 NV12 格式 | 用 RGA 或 CPU 做格式转换 |
| 2026-06-18 | 编码输出为 0 字节 | SPS/PPS 头信息未正确获取 | 检查 `MPP_ENC_GET_EXTRA_INFO` 调用时机 |
| 2026-06-18 | Ftrace 未捕获 mpp_service 函数 | `set_ftrace_filter` 未匹配 | `cat available_filter_functions \| grep -E "mpp|rkv"` 确认 |

---

## 十一、下阶段预告

> **学习路线已对标大疆 BSP 岗位 JD 重新规划**，详见 [[MOC-嵌入式Linux]]。

阶段四：**Bootloader + 系统启动流程**（JD: Bootloader移植、启动优化）
- Boot chain：ROM → DDR init → SPL → ATF → U-Boot → Kernel
- U-Boot 配置编译、FIT 镜像、分区表
- 启动时间测量与优化

阶段五：**Linux 设备模型 + 设备树**（JD: 设备模型、驱动开发）
- Bus/Device/Driver 模型、DTS binding、platform driver
- 编写第一个内核驱动

阶段六：**中断处理 + 并发**（JD: 中断处理）
- GIC-400、IRQ 注册、top/bottom half、spinlock/mutex

阶段七：**外设驱动 I2C/SPI/UART**（JD: 外设接口协议）
- I2C/SPI/UART 子系统、写完整外设驱动

阶段八：**电源管理**（JD: 电源管理）
- Clock/Regulator/Runtime PM/Suspend-Resume

阶段九：**Capstone 完整 BSP 驱动项目**

---

## 相关笔记

- [[kernel-debug-env]] — 内核 Debug 环境搭建
- [[v4l2-isp-deep-dive]] — V4L2/UVC 驱动深度分析
- [[rv1126b]] — RV1126B 运动相机项目
- [[MOC-嵌入式Linux]] — 嵌入式 Linux 学习地图
