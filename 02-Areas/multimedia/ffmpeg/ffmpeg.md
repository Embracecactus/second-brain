---
tags:
  - multimedia
  - ffmpeg
  - video
  - audio
  - codec
  - streaming
  - embedded
  - v4l2
category: multimedia/ffmpeg
created: 2026-06-09
status: learning
aliases:
  - FFmpeg
  - ffmpeg学习笔记
---

# FFmpeg 多媒体处理框架

## 项目概述

FFmpeg 是一个完整的跨平台多媒体处理框架，提供音视频的录制、转换、流媒体推流等核心功能。项目包含多个核心库（libavcodec、libavformat、libavfilter 等）和命令行工具（ffmpeg、ffplay、ffprobe），是多媒体领域事实上的标准基础设施。

## 技术栈

- **语言**: C（核心库），少量汇编优化（x86 SIMD、ARM NEON、RISC-V RVV）
- **构建系统**: 自定义 configure 脚本 + GNU Make
- **许可证**: LGPL（核心），GPL（部分可选组件）
- **支持平台**: Linux、Windows、macOS、嵌入式（ARM64、RISC-V）
- **硬件加速**: V4L2、DRM/KMS、QSV（Intel）、NVENC/NVDEC（NVIDIA）、MPP（Rockchip）、HiVPP（海思）

## 架构设计

### 核心库模块

| 库 | 职责 | 关键源码目录 |
|---|---|---|
| libavcodec | 编解码器实现（H.264/H.265/AAC/AV1 等） | `libavcodec/` |
| libavformat | 容装格式解析与流媒体协议（RTSP/RTMP/SRT） | `libavformat/` |
| libavfilter | 音视频滤镜图（scale/overlay/rotate/deshake） | `libavfilter/` |
| libavutil | 基础工具（像素格式、数学运算、内存管理） | `libavutil/` |
| libswresample | 音频重采样与混合 | `libswresample/` |
| libswscale | 色彩空间转换与图像缩放 | `libswscale/` |
| libavdevice | 设备输入输出抽象（V4L2、ALSA） | `libavdevice/` |

### 命令行工具

| 工具 | 功能 |
|---|---|
| ffmpeg | 多媒体转换/处理命令行工具箱 |
| ffplay | 最小化多媒体播放器 |
| ffprobe | 媒体文件信息分析工具 |

### 关键设计决策

1. **分离架构**: 编解码（codec）、封装格式（format）、滤镜（filter）完全解耦，通过统一接口组合
2. **硬件抽象层**: `libavutil/hwcontext*.c` 提供统一的硬件加速抽象，支持多种后端
3. **滤镜图模型**: 基于有向图的滤镜连接方式，支持复杂处理管线
4. **零拷贝设计**: AVFrame 引用计数机制减少内存拷贝
5. **模块化注册**: 所有编解码器、格式、协议通过注册表机制动态发现

## 核心数据流

### 解封装 + 解码流程

```
avformat_open_input()
    ↓
avformat_find_stream_info()
    ↓
av_read_frame()  →  AVPacket
    ↓
avcodec_send_packet()
    ↓
avcodec_receive_frame()  →  AVFrame
```

### 编码 + 封装流程

```
AVFrame (原始数据)
    ↓
avcodec_send_frame()
    ↓
avcodec_receive_packet()  →  AVPacket
    ↓
av_interleaved_write_frame()
```

### 滤镜处理流程

```
avfilter_graph_alloc()
    ↓
avfilter_graph_create_filter()  (构建滤镜图)
    ↓
av_buffersrc_add_frame()  (输入帧)
    ↓
av_buffersink_get_frame()  (输出帧)
```

## 关键源码入口

| 功能 | 源文件 |
|---|---|
| H.264 解码器 | `libavcodec/h264dec.c`, `libavcodec/h264_ps.c` |
| H.264 编码器 | `libavcodec/h264enc.c`, `libavcodec/h264_cabac.c` |
| V4L2 设备采集 | `libavdevice/v4l2.c`, `libavcodec/v4l2_*.c` |
| 滤镜基类 | `libavfilter/avfilter.c`, `libavfilter/video.h` |
| RTSP 协议 | `libavformat/rtsp*.c` |
| RTMP 协议 | `libavformat/rtmp*.c` |
| 色彩转换 | `libswscale/*.c` |
| 硬件上下文 | `libavutil/hwcontext*.c` |
| ffmpeg 命令行入口 | `fftools/ffmpeg.c` |
| 容器格式注册 | `libavformat/allformats.c` |

## 代码示例

### 快速解码示例（quick_start.c 核心片段）

```c
// 打开输入文件
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, argv[1], NULL, NULL);
avformat_find_stream_info(fmt_ctx, NULL);

// 查找视频流并创建解码器
int video_stream_idx = -1;
for (unsigned int i = 0; i < fmt_ctx->nb_streams; i++) {
    if (fmt_ctx->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_VIDEO) {
        video_stream_idx = i;
        break;
    }
}

const AVCodec *codec = avcodec_find_decoder(
    fmt_ctx->streams[video_stream_idx]->codecpar->codec_id);
AVCodecContext *codec_ctx = avcodec_alloc_context3(codec);
avcodec_parameters_to_context(codec_ctx, codec_par);
avcodec_open2(codec_ctx, codec, NULL);

// 解码循环
AVPacket *packet = av_packet_alloc();
AVFrame *frame = av_frame_alloc();
while (av_read_frame(fmt_ctx, packet) >= 0) {
    if (packet->stream_index == video_stream_idx) {
        avcodec_send_packet(codec_ctx, packet);
        while (avcodec_receive_frame(codec_ctx, frame) == 0) {
            // 处理解码后的 frame
        }
    }
    av_packet_unref(packet);
}
```

编译命令：`gcc -o quick_start quick_start.c -lavformat -lavcodec -lavutil -lswscale`

## 构建与运行

### 基础编译（推荐学习用）

```bash
./configure --enable-shared --disable-static --disable-doc
make -j$(nproc)
sudo make install
```

### 完整编译（含 GPL 组件）

```bash
./configure --enable-gpl --enable-version3 --enable-nonfree --enable-libx264
make -j$(nproc)
```

### 交叉编译（ARM64 嵌入式平台）

```bash
./configure --cross-prefix=aarch64-linux-gnu- \
    --enable-cross-compile \
    --arch=aarch64 \
    --target-os=linux \
    --enable-shared
```

### 仅编译示例代码

```bash
cd doc/examples && make
```

### 清理

```bash
make distclean
```

## 学习路线

1. **入门**: 命令行使用，理解容器/编解码器/像素格式概念
2. **API 编程**: 从 `doc/examples/decode_video.c`、`encode_video.c` 开始
3. **源码阅读**: 掌握 AVFormatContext -> AVCodecContext -> AVFrame 数据流
4. **深入编解码**: H.264 源码（`libavcodec/h264*.c`），理解 DCT/量化/运动估计
5. **硬件加速**: V4L2 相机采集（`libavdevice/v4l2.c`），硬件编解码上下文
6. **滤镜开发**: 视频处理滤镜（`libavfilter/vf_*.c`），自定义滤镜编写
7. **流媒体**: RTSP/RTMP 推流实现，低延迟优化

## 关键概念

- **I/P/B 帧**: I 帧（关键帧，可独立解码）、P 帧（前向预测）、B 帧（双向预测）
- **GOP**: Group of Pictures，两个 I 帧之间的帧序列
- **YUV 色彩空间**: YUV420P（每 4 个 Y 共享一组 UV）、NV12（半平面存储）
- **码率控制**: CBR（恒定码率）、VBR（可变码率）、CRF（恒定质量因子）
- **零拷贝**: AVFrame 引用计数机制，避免不必要的内存复制
- **帧内预测**: 利用同一帧内相邻像素进行预测（H.264 的核心技术之一）
- **帧间预测**: 利用参考帧进行运动补偿预测
- **环路滤波（Deblocking）**: 去除块效应的后处理滤波器

## 相关资源

- FFmpeg 官方文档: [https://ffmpeg.org](https://ffmpeg.org)
- FFmpeg Wiki: [https://trac.ffmpeg.org/wiki](https://trac.ffmpeg.org/wiki)
- 编程示例目录: `doc/examples/`
- 离线文档: `doc/` 目录
- 学习路线图: `LEARNING_PLAN.md`
- 面试高频题: H.264 I/P/B 帧区别、YUV 存储格式、AVFrame 引用计数、V4L2 采集流程、实时编码延迟优化

## 相关笔记

- [[H.264 编解码原理]]
- [[V4L2 相机采集]]
- [[RTSP 流媒体协议]]
- [[YUV 色彩空间]]
- [[视频滤镜开发]]
- [[嵌入式硬件加速]]

## 相关笔记

- [[ffmpeg-learning]] — FFmpeg 学习项目
- [[rv1126b]] — RV1126B 运动相机项目（V4L2 + FFmpeg）
- [[selfMedia]] — 猪猪猪序员自媒体（FFmpeg 视频处理）
- [[douyin-crawler]] — 抖音爬虫（FFmpeg 音频提取）
- [[h618-buildroot]] — H618 完整开发笔记（VPU + FFmpeg）
