---
title: ffmpeg-learning
category: multimedia/ffmpeg
tags:
  - ffmpeg
  - 嵌入式
  - 视频编解码
  - V4L2
  - rockchip
  - rv1126b
  - 多媒体
created: 2026-06-09
status: 进行中
project_path: /home/lijian/project/ffmpeg-learning
---

# ffmpeg-learning

## 项目概述

这是一个面向嵌入式相机系统的 FFmpeg 学习项目，目标硬件为瑞芯微 RV1126B 开发板（ARM64 嵌入式 Linux），旨在掌握 FFmpeg 核心框架、V4L2 视频采集及嵌入式媒体处理流程，为进入大疆/影石等相机公司做准备。

## 技术栈

| 类别 | 技术 |
|------|------|
| 多媒体框架 | FFmpeg (libavformat, libavcodec, libavfilter, libswscale, libswresample) |
| 视频采集 | V4L2 (Video4Linux2) |
| 视频编码 | H.264 |
| 嵌入式平台 | Rockchip RV1126B (ARM64) |
| 操作系统 | Embedded Linux (Ubuntu/Debian) |
| 推流协议 | RTSP |
| 文件系统 | ext4, 自动扩容机制 (resize2fs) |
| 构建系统 | Makefile, Rockchip SDK build.sh |

## 架构与关键设计

### FFmpeg 核心架构

```
应用层 (ffmpeg.c)
    ↓
libavformat (封装层) → AVFormatContext, AVStream
    ↓
libavcodec (编解码层) → AVCodecContext, AVPacket, AVFrame
    ↓
libavfilter (滤镜层)
    ↓
libswscale / libswresample
```

### 数据流

```
av_read_frame() → AVPacket (压缩数据)
    ↓
avcodec_send_packet()
    ↓
avcodec_receive_frame()
    ↓
AVFrame (YUV 原始像素)
```

### V4L2 采集流程

```
open(/dev/video0)
  → VIDIOC_QUERYCAP (查询设备能力)
  → VIDIOC_S_FMT (设置像素格式/分辨率)
  → VIDIOC_REQBUFS (申请缓冲区)
  → mmap (内存映射, 零拷贝)
  → VIDIOC_QBUF (缓冲区入队)
  → VIDIOC_STREAMON (启动采集)
  → 循环: VIDIOC_DQBUF → 处理 → VIDIOC_QBUF
  → VIDIOC_STREAMOFF (停止采集)
```

### 嵌入式预览方案（无 X11 环境）

| 方案 | 描述 | 推荐度 |
|------|------|--------|
| RTSP 推流 | 板子推流，PC 用 ffplay/VLC 播放 | 推荐 |
| Framebuffer | 直接写入 /dev/fb0 | 备选 |
| MJPEG HTTP | 浏览器访问 | 备选 |
| 文件传输 | 保存到文件后拷贝到 PC 查看 | 调试用 |

### RK SDK 分区与自动扩容

RV1126B 的 eMMC 分区布局（约 58GB）：

| 分区 | 大小 | 用途 |
|------|------|------|
| p1 | 4M | uboot |
| p2 | 4M | misc |
| p3 | 64M | boot |
| p4 | 4M | amp/recovery |
| p5 | 128M | recovery |
| p6 | 32M | backup |
| p7 | 6G | rootfs (grow) |
| p8 | 128M | oem |
| p9 | 51.9G | userdata (grow) |

自动扩容机制：`parameter.txt` 定义 grow 分区 → `resize-all.service` 启动时触发 → `resize-helper` 调用 `resize2fs` 等工具在线扩容 → `.resized` 标记文件防止重复执行。

## 核心知识点

### AVPacket vs AVFrame

- **AVPacket**：压缩数据（如 H.264 NALU），来自容器解封装或送往编码器
- **AVFrame**：原始像素数据（YUV），来自解码器或送往编码器
- 两者均使用引用计数管理生命周期：`av_packet_ref()` / `av_packet_unref()`, `av_frame_ref()` / `av_frame_unref()`

### V4L2 缓冲区机制

- **QBUF** (Queue Buffer)：将缓冲区放入队列，等待内核填充数据
- **DQBUF** (Dequeue Buffer)：从队列取出已填充的缓冲区
- **mmap** 方式实现零拷贝，避免用户态与内核态之间的数据复制

### FFmpeg 入口函数

| 工具 | 源码位置 |
|------|----------|
| ffmpeg 命令 | `fftools/ffmpeg.c:981` |
| ffplay 播放器 | `fftools/ffplay.c:3847` |
| ffprobe 分析器 | `fftools/ffprobe.c:3319` |

### RK SDK 分区配置

- 分区定义文件：`device/rockchip/.chips/rv1126b/parameter.txt`
- CMDLINE 格式：`大小@起始地址(分区名)`，单位为扇区（1扇区 = 512字节）
- `grow` 标记使分区自动使用剩余空间
- 分区必须连续，修改后需重新编译烧录固件

## 重要代码片段

### V4L2 采集关键结构

```c
// 源码位置: libavdevice/v4l2.c
struct video_data {      // V4L2 上下文
    int fd;              // 设备文件描述符
    // ...
};
struct v4l2_buffer {     // 缓冲区描述
    // ...
};
struct buf_data {        // mmap 缓冲区信息
    // ...
};
```

### 引用计数管理

```c
av_packet_ref(dst, src);    // 增加引用
av_packet_unref(pkt);       // 减少引用, 引用为0时释放
av_frame_ref(dst, src);
av_frame_unref(frame);
av_buffer_create(...);      // 创建自动管理释放的 buffer
```

### RK SDK 分区解析核心函数

```bash
# 源码位置: device/rockchip/common/scripts/partition-helper
rk_partition_parse()        # 解析 parameter.txt 为 <name> <size> 格式
rk_partition_size_kb(name)  # 获取分区大小 (KB)
rk_partition_size(name)     # 获取分区大小 (扇区)
rk_partition_start(name)    # 获取分区起始地址
```

### 文件系统扩容核心逻辑

```bash
# 源码位置: disk-helpers/usr/bin/disk-helper
resize_ext2() {
    check_tool resize2fs BR2_PACKAGE_E2FSPROGS_RESIZE2FS || return 1
    resize2fs $DEV   # 自动扩展到分区实际大小
}

resize_part() {
    if [ -f $MOUNT_POINT/.resized ]; then return; fi  # 防重复
    remount_part rw "for online-resize"
    if eval $FSRESIZE; then
        touch $MOUNT_POINT/.resized  # 标记已扩容
        sync
    fi
}
```

## 构建/运行方法

### FFmpeg 常用命令

```bash
# 列出摄像头支持的格式
ffmpeg -f v4l2 -list_formats all -i /dev/video0

# 采集预览 (PC 端)
ffplay -f v4l2 -video_size 640x480 -i /dev/video0

# 采集保存
ffmpeg -f v4l2 -video_size 640x480 -i /dev/video0 -t 10 output.mp4

# V4L2 设备调试
v4l2-ctl --list-devices
v4l2-ctl --device=/dev/video0 --list-formats
```

### 嵌入式板子网络配置

```bash
ip addr add 192.168.1.109/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.1.100
```

### RK SDK 构建流程

```bash
cd /home/lijian/project/rv1126b/atk/atk_dlrv1126b_linux6.1_sdk_release_v1.2.1_20260327
./build.sh -c 01_atk_dlrv1126b_automipi_defconfig   # 选择配置
./build.sh                                           # 完整编译
./build.sh firmware                                  # 仅打包固件
upgrade_tool update output/firmware/Update.img       # 烧录
```

### 集成 disk-helpers 到自定义文件系统

```bash
cd ubuntu_debian
sudo mount rootfs.ext4 mnt
sudo cp -r ../device/rockchip/common/overlays/rootfs/disk-helpers/usr/* mnt/usr/
sudo cp ../device/rockchip/common/overlays/rootfs/disk-helpers/resize-all.service mnt/etc/systemd/system/
sudo chroot mnt /bin/bash -c "systemctl enable resize-all.service"
sudo umount mnt
make mkrootfsext4
```

## 项目文件结构

```
ffmpeg-learning/
├── README.md                                    # 学习日志与进度记录
├── CLAUDE.md                                    # 项目上下文与 FFmpeg 知识速查
└── docs/
    ├── rk-sdk-partition.md                      # RK SDK 分区配置详解
    ├── rk-sdk-auto-resize.md                    # 自动扩容机制原理
    └── rk-sdk-ubuntu-resize-integration.md      # 集成扩容到自定义文件系统
```

## 相关笔记链接

- [[rockchip-rv1126b]] - 瑞芯微 RV1126B 开发板笔记
- [[v4l2-video-capture]] - V4L2 视频采集详解
- [[ffmpeg-architecture]] - FFmpeg 框架与数据流
- [[h264-encoding]] - H.264 编码原理
- [[rtsp-streaming]] - RTSP 推流方案
- [[embedded-linux]] - 嵌入式 Linux 开发笔记
- [[rockchip-sdk-partition]] - RK SDK 分区与固件打包
- [[disk-helpers-auto-resize]] - 文件系统自动扩容机制

## 相关笔记

- [[ffmpeg]] — FFmpeg 多媒体处理框架
- [[rv1126b]] — RV1126B 运动相机项目（目标硬件平台）
- [[rv1126-notes]] — RV1126B 嵌入式开发笔记
- [[rk]] — Rockchip Linux SDK
- [[camera-diag-skills]] — Camera 诊断技能
- [[skill]] — GenICam 属性管理与专家系统
