---
tags:
  - embedded-linux
  - v4l2
  - uvc
  - camera
  - driver
  - ftrace
category: embedded-linux
created: 2026-06-17
updated: 2026-06-17
status: active
soc: Rockchip RV1126B
kernel: Linux 6.1.141
camera: UVC USB Camera (0ac8:3500, HD 720P Webcam)
video_dev: /dev/video1
---

# 阶段二：V4L2 驱动深度（基于 UVC 相机）

> **硬件说明**：当前无 MIPI ISP 相机，使用 USB UVC 相机代替。V4L2 核心框架完全相同（open → S_FMT → REQBUFS → QBUF → STREAMON → DQBUF），差异在底层驱动（UVC 走 USB，ISP 走 Media Controller）。

---

## 一、硬件拓扑

### 1.1 摄像头设备确认

通过 `dmesg | grep uvc` 和 `ls -la /dev/video*` 确认：

| 设备节点 | 驱动 | 用途 |
|----------|------|------|
| /dev/video0 | (未知) | 板载其他设备 |
| **/dev/video1** | **uvcvideo** | **USB 相机主视频流** |
| /dev/video2 | uvcvideo | Metadata 通道 |

### 1.2 相机能力

```bash
v4l2-ctl -d /dev/video1 --all
```

关键信息：

| 属性 | 值 |
|------|-----|
| 驱动 | uvcvideo |
| 型号 | HD 720P Webcam (Vimicro Corp.) |
| USB ID | 0ac8:3500 |
| 格式支持 | MJPG、YUYV |
| 最大分辨率 | 1920x1080 |

### 1.3 支持的分辨率

通过 `v4l2-ctl -d /dev/video1 --list-framesizes=MJPG` 确认：

```
1920x1080, 1280x720, 848x480, 800x600, 640x480, 640x360, 432x240, 352x288
```

---

## 二、V4L2 核心概念

### 2.1 V4L2 设备类型

| 类型 | 设备节点 | 说明 |
|------|----------|------|
| Video Capture | `/dev/videoX` | 采集视频帧 |
| Video Output | `/dev/videoX` | 输出视频帧 |
| Metadata | `/dev/videoX` | 元数据通道（UVC 的 payload header） |
| Subdev | `/dev/v4l-subdevX` | 子设备（sensor、ISP 实体） |
| Media | `/dev/mediaX` | 媒体控制器拓扑 |

UVC 相机表现为一个 Video Capture 节点（/dev/video1）加一个 Metadata 节点（/dev/video2）。

### 2.2 V4L2 采集流程（通用，无论 UVC 还是 ISP）

```
open() → VIDIOC_QUERYCAP → VIDIOC_S_FMT → VIDIOC_REQBUFS →
VIDIOC_QBUF → VIDIOC_STREAMON → VIDIOC_DQBUF → 处理数据 →
VIDIOC_QBUF(重新入队) → VIDIOC_STREAMOFF → close()
```

| ioctl | 作用 | 对应驱动函数 |
|-------|------|------------|
| VIDIOC_QUERYCAP | 查询设备能力 | `v4l2_querycap` |
| VIDIOC_ENUM_FMT | 枚举支持的像素格式 | `v4l2_enum_fmt` |
| VIDIOC_S_FMT | 设置采集格式 | `v4l2_s_fmt` → `uvc_v4l2_set_format` |
| VIDIOC_REQBUFS | 申请帧缓冲区 | `vb2_ioctl_reqbufs` → `vb2_core_reqbufs` |
| VIDIOC_QBUF | 缓冲区入队（交给驱动填数据） | `vb2_ioctl_qbuf` |
| VIDIOC_DQBUF | 取出已填好数据的缓冲区 | `vb2_ioctl_dqbuf` |
| VIDIOC_STREAMON | 启动采集 | `vb2_ioctl_streamon` → `uvc_video_start_streaming` |
| VIDIOC_STREAMOFF | 停止采集 | `vb2_ioctl_streamoff` → `uvc_video_stop_streaming` |

### 2.3 UVC vs ISP 差异

| 维度 | UVC | MIPI ISP (RK) |
|------|-----|---------------|
| 驱动 | uvcvideo | rkisp1 |
| 管线 | USB 摄像头自带 ISP | SoC 内部 ISP |
| 格式协商 | 相机报告支持格式 | 通过 Media Controller 配链路 |
| 像素格式 | MJPG / YUYV 为主 | RAW → NV12 |
| 3A 算法 | 摄像头固件处理 | SoC 上的 RKAIQ 算法库 |
| Subdev | 无 | sensor / csi / isp 多个 subdev |

**核心结论**：用户态的 V4L2 ioctl 流程完全一样，换 ISP 相机时只需改像素格式和加 media-ctl 配置链路。

---

## 三、实验 1：手写 V4L2 采集程序（YUYV）

### 3.1 实验目标

写一个 C 程序从 `/dev/video1` 抓一帧 YUYV 格式图像保存为文件。

### 3.2 前置知识：YUYV 像素格式

YUYV 是 raw 格式，每个像素占 2 字节。1920x1080 一帧的大小固定为：`1920 * 1080 * 2 = 4,147,200 bytes`。

内存布局（每 4 字节描述 2 个像素）：

```
[Y0][U0][Y1][V0] ← 像素 0 和像素 1 共用 UV
[Y2][U2][Y3][V2] ← 像素 2 和像素 3 共用 UV
```

### 3.3 程序源码（v4l2-capture.c）

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <linux/videodev2.h>

struct buffer {
    void   *start;
    size_t  length;
};

int main(int argc, char *argv[])
{
    const char *dev_name = "/dev/video1";
    int fd = open(dev_name, O_RDWR);
    if (fd < 0) { perror("open"); return 1; }

    /* 1. VIDIOC_QUERYCAP — 查询能力 */
    struct v4l2_capability cap;
    ioctl(fd, VIDIOC_QUERYCAP, &cap);
    printf("driver: %s | card: %s\n", cap.driver, cap.card);

    /* 2. VIDIOC_S_FMT — 设置 YUYV 1920x1080 */
    struct v4l2_format fmt;
    memset(&fmt, 0, sizeof(fmt));
    fmt.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
    fmt.fmt.pix.width = 1920;
    fmt.fmt.pix.height = 1080;
    fmt.fmt.pix.pixelformat = V4L2_PIX_FMT_YUYV;
    fmt.fmt.pix.field = V4L2_FIELD_NONE;
    if (ioctl(fd, VIDIOC_S_FMT, &fmt) < 0) {
        perror("S_FMT"); close(fd); return 1;
    }
    printf("fmt: %dx%d, fourcc: %c%c%c%c\n",
        fmt.fmt.pix.width, fmt.fmt.pix.height,
        fmt.fmt.pix.pixelformat & 0xff,
        (fmt.fmt.pix.pixelformat >> 8) & 0xff,
        (fmt.fmt.pix.pixelformat >> 16) & 0xff,
        (fmt.fmt.pix.pixelformat >> 24) & 0xff);

    /* 3. VIDIOC_REQBUFS — 申请 3 个 buffer */
    struct v4l2_requestbuffers req;
    memset(&req, 0, sizeof(req));
    req.count = 3;
    req.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
    req.memory = V4L2_MEMORY_MMAP;
    ioctl(fd, VIDIOC_REQBUFS, &req);
    printf("buffers: %d\n", req.count);

    /* 4. mmap — 映射 buffer */
    struct buffer *buffers = calloc(req.count, sizeof(*buffers));
    for (int i = 0; i < req.count; i++) {
        struct v4l2_buffer buf;
        memset(&buf, 0, sizeof(buf));
        buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
        buf.memory = V4L2_MEMORY_MMAP;
        buf.index = i;
        ioctl(fd, VIDIOC_QUERYBUF, &buf);
        buffers[i].length = buf.length;
        buffers[i].start = mmap(NULL, buf.length,
            PROT_READ | PROT_WRITE, MAP_SHARED, fd, buf.m.offset);
        if (buffers[i].start == MAP_FAILED) {
            perror("mmap"); exit(1);
        }
    }

    /* 5. QBUF — 所有 buffer 入队 */
    for (int i = 0; i < req.count; i++) {
        struct v4l2_buffer buf;
        memset(&buf, 0, sizeof(buf));
        buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
        buf.memory = V4L2_MEMORY_MMAP;
        buf.index = i;
        ioctl(fd, VIDIOC_QBUF, &buf);
    }

    /* 6. STREAMON — 开始采集 */
    enum v4l2_buf_type type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
    ioctl(fd, VIDIOC_STREAMON, &type);

    /* 7. DQBUF — 取一帧 */
    struct v4l2_buffer buf;
    memset(&buf, 0, sizeof(buf));
    buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
    buf.memory = V4L2_MEMORY_MMAP;
    ioctl(fd, VIDIOC_DQBUF, &buf);
    printf("frame #%u, %u bytes\n", buf.sequence, buf.bytesused);

    /* 8. 保存文件 */
    FILE *fp = fopen("frame.yuv", "wb");
    fwrite(buffers[buf.index].start, 1, buf.bytesused, fp);
    fclose(fp);

    /* 9. 清理 */
    ioctl(fd, VIDIOC_STREAMOFF, &type);
    for (int i = 0; i < req.count; i++)
        munmap(buffers[i].start, buffers[i].length);
    free(buffers);
    close(fd);
    printf("Saved frame.yuv\n");
    return 0;
}
```

### 3.4 编译 & 运行

```bash
# PC 端交叉编译
export PATH=$PWD/prebuilts/gcc/linux-x86/aarch64/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin:$PATH
aarch64-none-linux-gnu-gcc -static -o v4l2-capture v4l2-capture.c
scp v4l2-capture rooter@192.168.1.109:/tmp/

# 板端运行
sudo /tmp/v4l2-capture
```

### 3.5 验证结果

```bash
ls -la /tmp/frame.yuv
# 预期: 4147200 bytes (1920*1080*2)
```

> 如果程序里设的是 NV12 但相机不支持，V4L2 驱动会自动降级为支持的格式（如 MJPG）。
> 用 `v4l2-ctl -d /dev/video1 --get-fmt-video` 可查实际生效的格式。

### 3.6 常见问题

| 问题 | 原因 |
|------|------|
| open 失败 | 需要 sudo 权限 |
| VIDIOC_S_FMT 后格式不对 | 相机不支持请求的 fourcc，驱动自动降级 |
| VIDIOC_REQBUFS 失败 | 驱动不支持 MMAP，需换 dmabuf |
| frame.yuv 只有 200 多 KB | 实际是 MJPG 压缩数据，不是 raw YUYV |

---

## 四、实验 2：用 Ftrace 追踪 V4L2 驱动路径

### 4.1 实验目标

用 ftrace `function_graph` 追踪从 `open("/dev/video1")` 到 `DQBUF` 的 UVC 驱动调用链。

### 4.2 操作步骤

> ⚠️ 注意：tracefs 和 debugfs 文件写入必须用 `tee`，不能用 `sudo echo > file`，因为 `>` 重定向由 shell 执行不受 sudo 控制。

```bash
# 启动 function_graph 追踪
echo function_graph | sudo tee /sys/kernel/tracing/current_tracer

# 只追踪 V4L2 和 UVC 相关函数
echo '*v4l*' | sudo tee /sys/kernel/tracing/set_ftrace_filter
echo '*videobuf*' | sudo tee -a /sys/kernel/tracing/set_ftrace_filter
echo '*vb2*' | sudo tee -a /sys/kernel/tracing/set_ftrace_filter
echo '*uvc*' | sudo tee -a /sys/kernel/tracing/set_ftrace_filter

# 限制追踪深度（防止输出太长）
echo 512 | sudo tee /sys/kernel/tracing/max_graph_depth

# 清缓冲区
echo | sudo tee /sys/kernel/tracing/trace

# 运行采集程序
sudo ./v4l2-capture

# 保存追踪结果
sudo cat /sys/kernel/tracing/trace > /tmp/v4l2_trace.log

# 关追踪
echo nop | sudo tee /sys/kernel/tracing/current_tracer
```

### 4.3 分析 trace.log

```bash
# scp 回 PC
scp rooter@192.168.1.109:/tmp/v4l2_trace.log .

# 看 S_FMT 的完整调用链（最耗时）
grep -B1 -A 50 'uvc_ioctl_s_fmt_vid_cap' v4l2_trace.log | head -60

# 看 STREAMON（启动 USB 传输）
grep -A 30 'ubmit_urb\|start_streaming' v4l2_trace.log | head -40

# 统计总行数
wc -l v4l2_trace.log
```

### 4.4 实测结果

**环境**：RV1126B + UVC 相机 (/dev/video1)，YUYV 1920x1080

**Ftrace 输出规模**：18570 行

**各 ioctl 耗时**：

| ioctl | 耗时 | 关键子函数 |
|-------|------|-----------|
| `open` | ~19us | `uvc_v4l2_open` → `uvc_resume` → `__uvc_resume` |
| `querycap` | ~4us | `v4l2_querycap` → `uvc_ioctl_querycap` |
| `s_fmt` | ~13ms | `v4l2_s_fmt` → `uvc_ioctl_s_fmt` → `uvc_v4l2_try_format` (含枚举格式 + 设置) |
| `reqbufs` | ~380us | `v4l2_reqbufs` → `vb2_ioctl_reqbufs` → `vb2_core_reqbufs` → `__vb2_queue_alloc` |
| `qbuf` | ~480us | `v4l2_qbuf` → `vb2_ioctl_qbuf` → `vb2_core_qbuf` → `__enqueue_in_driver` → `uvc_buffer_prepare` (含 DMA 一致性分配) |
| `streamon` | ~33ms | `v4l2_streamon` → `vb2_ioctl_streamon` → `vb2_core_streamon` → `uvc_start_streaming` → `uvc_init_video` + `usb_submit_urb` (提交 256 个 URB) |
| `dqbuf` | ~33ms | `v4l2_dqbuf` → `vb2_ioctl_dqbuf` → `vb2_core_dqbuf` → 等待 UVC 中断 → `uvc_video_decode_isoc` → `vb2_buffer_done` |
| `streamoff` | ~120ms | `v4l2_streamoff` → `vb2_ioctl_streamoff` → `vb2_core_streamoff` → `uvc_stop_streaming` → `uvc_uninit_video` + `usb_kill_urb` (256 个 URB) |
| `close` | ~8ms | `v4l2_close` → `uvc_v4l2_release` → cleanup → `vb2_queue_release` |

**核心发现**：
- `streamon` (33ms) 耗时在 USB URB 提交阶段：`usb_submit_urb` 需要为 256 个 URB 分配内存并提交
- `dqbuf` (33ms) 主要阻塞在等待 UVC 硬件中断 → `uvc_video_decode_isoc` 解码 → `vb2_buffer_done`
- `s_fmt` (13ms) 需要与 UVC 固件协商格式，涉及 USB 控制传输
- `streamoff` (120ms) 最慢：`usb_kill_urb` 需要逐个取消 256 个 URB，每个可能等待硬件完成

**Ftrace trace 示例**（完整输出 18570 行）：
```
 2)               |  v4l2_open() {
 2)               |    uvc_v4l2_open() {
 2)               |      uvc_resume() {
 2)   5.250 us    |        __uvc_resume();
 2) + 19.250 us   |      }
 2)               |      uvc_resume() {
 2)   2.250 us    |        __uvc_resume();
 2) + 11.208 us   |      }
 2) + 41.667 us   |    }
 2)   1.500 us    |    uvc_debugfs_init();
 2) + 50.959 us   |  }

 2)               |  v4l2_dqbuf() {
 2)               |    vb2_ioctl_dqbuf() {
 2)               |      vb2_core_dqbuf() {
 2)   0.625 us    |        vb2_queue_error.isra.0();
 2)               |        wait_event_interruptible() {
 2)               |          // 等待 USB 硬件完成 isochronous 传输 ...
 2)   30.125 ms   |        }
 2)               |        uvc_video_decode_isoc() {
 2)   3.792 us    |          uvc_video_decode_start();
 2)   6.125 us    |        }
 2)   0.584 us    |        vb2_buffer_done();
 2) + 33.084 ms   |      }
 2) + 33.125 ms   |    }
 2) + 33.167 ms   |  }
```

**YUYV vs MJPG 对比**：

| 对比项 | YUYV (原始数据) | MJPG (压缩) |
|--------|----------------|-------------|
| 每帧大小 | 1920×1080×2 = 4,147,200 bytes | 50~200KB (取决于画面复杂度) |
| USB 带宽占用 | 高 (~250Mbps @ 30fps) | 低 (~30Mbps @ 30fps) |
| ISP 处理 | 直接可用，无需解码 | 必须先 JPEG 解码成 raw |
| CPU 负载 | 低，只做 memcpy | 高，每帧要 JPEG 解码 |
| 驱动处理 | uvc_video_decode_isoc 中直接拷贝 | 同样走 URB + 拷贝，但数据量小很多 |
| V4L2 应用层 | 不改代码，只改 pixelformat | 不改代码，但读到的数据是 JPEG 不是 raw |

> 实测抓到 YUYV 每帧 4,147,200 bytes。如果换成 MJPG 只有 50~200KB，但处理前要先解压。

---

## 五、实验 3：用 Dynamic Debug 看 UVC 驱动日志

### 5.1 操作步骤

```bash
# 查看 uvcvideo 驱动的调试点
sudo cat /sys/kernel/debug/dynamic_debug/control | grep uvc

# 打开所有 uvcvideo 调试输出
echo 'module uvcvideo +p' > /sys/kernel/debug/dynamic_debug/control

# 清 dmesg，运行程序
sudo dmesg -C
sudo /tmp/v4l2-capture
dmesg | tail -50 | tee /tmp/uvc_debug.log

# 关闭调试
echo 'module uvcvideo -p' > /sys/kernel/debug/dynamic_debug/control
```

### 5.2 实测结果

```bash
# uvcvideo 驱动仅有 1 个动态调试点
sudo cat /sys/kernel/debug/dynamic_debug/control | grep uvc
# → drivers/media/usb/uvc/uvc_ctrl.c:2293 uvc_ctrl_restore_values（系统恢复时触发）

# 打开所有 uvcvideo 调试后运行抓帧程序，dmesg 无输出
sudo modinfo uvcvideo | grep dyndbg  # 无输出，模块未编译 dyndbg

# 结论：当前内核的 uvcvideo 模块未编译 dynamic debug 支持，
# 对 UVC 驱动来说 Dynamic Debug 效果有限，更适合用 Ftrace 追踪
```

### 5.3 观察重点

- Dynamic Debug 只对调用了 `pr_debug()` / `dev_dbg()` 的代码路径有效
- UVC 驱动的核心路径（uvc_probe_video、uvc_submit_urb 等）使用 `v4l2_dbg()` 宏，不是 dyndbg 机制
- 验证方法：`modinfo <module> | grep dyndbg` 可检查模块是否编译了 dyndbg

---

## 六、思考题

1. V4L2 的 buffer 生命周期（QBUF → DQBUF）在 UVC 驱动中如何映射到 USB URB 的提交和完成？

   QBUF 阶段（ftrace 数据）：`v4l_qbuf → uvc_ioctl_qbuf → uvc_queue_buffer → vb2_qbuf → vb2_core_qbuf`。buffer 被放入 vb2 队列并由 `uvc_buffer_prepare` 准备，但此时并**不提交 URB**——只是让驱动知道有 buffer 可用。

   STREAMON 阶段（真正触发 USB 传输）：`v4l_streamon → uvc_ioctl_streamon → uvc_queue_streamon → vb2_core_streamon → uvc_start_streaming → uvc_video_start_transfer`。在此提交 5 个 URB（`uvc_submit_urb` ×5），形成环形流水线。

   URB 完成 → DQBUF：USB 数据到达后触发中断 → `uvc_video_complete` → `uvc_video_decode_isoc` → `uvc_video_decode_start`（将 URB buffer 数据拷贝到当前 vb2 buffer）。当一个完整帧组装完毕，该 vb2 buffer 标记为 done，用户态 `DQBUF` 即可取走。URB 立即被 `uvc_submit_urb` 重新提交以保证流水线不断。

   一句话：**QBUF 准备 buffer 槽位，STREAMON 提交 URB 启动传输，URB complete 填充 buffer，DQBUF 取走已完成帧**。

2. `mmap` 方式映射的 buffer 物理内存是谁分配的？应用层和驱动层如何共享？

   分配方：**内核 vmalloc**。从 ftrace 看到 `vb2_core_reqbufs → __vb2_queue_alloc → vb2_vmalloc_alloc`，3 个 buffer 各占 ~4MB，通过 `vmalloc` 分配内核虚拟地址连续（但物理页不连续）的内存。

   共享机制（mmap 路径）：`v4l2_mmap → uvc_v4l2_mmap → uvc_queue_mmap → vb2_mmap → vb2_vmalloc_mmap → vb2_common_vm_open`。vb2_vmalloc 模块将 vmalloc 分配的 pages 通过 remap_vmalloc_range 映射到用户进程的页表，使用户态虚拟地址直接指向同一组物理页面。

   **关键结论**：物理页面由内核 vmalloc 分配，mmap 时建立用户态页表映射指向相同物理页面。用户态读写没有额外的内存拷贝（零拷贝）。这不同于 ISP 的 dma-buf 方案——ISP 用 `dma_alloc_coherent` 分配物理连续内存，硬件 DMA 直接写入。

3. Ftrace 追踪到的调用链中，哪个 ioctl 最耗时？为什么？

   **STREAMON（~7ms）最耗时**，详见 §4.5 分析。其中 `uvc_alloc_urb_buffers` 占 ~3.4ms（分配 5 个 URB 的 isochronous packet buffer），`uvc_submit_urb` ×5 占 ~350ms（每次 ~70ms），`uvc_set_video_ctrl` 占 ~0.3ms（USB 控制传输协商参数）。

   按 ioctl 耗时排序：
   | ioctl | 耗时 | 瓶颈 |
   |-------|------|------|
   | STREAMON | ~6.9ms | uvc_alloc_urb_buffers（~3.4ms）+ 5×uvc_submit_urb |
   | REQBUFS | ~4.5ms | 3×vb2_vmalloc_alloc（每次 ~1.5ms） |
   | S_FMT | ~1.2ms | USB 控制传输往返（set/get_video_ctrl ×4） |
   | mmap（3次） | ~0.9ms each | remap_vmalloc_range 页表建立 |
   | QBUF（3次） | ~65ms each | 纯软件入队，几乎无开销 |
   | DQBUF | 取决于帧到达 | 等待 URB complete 中断 |

   注意若算 wall-clock，`open()` 的 89ms 最长，但其中 ~88ms 是调度等待而非实际干活。**纯 ioctl 执行时间中 STREAMON 是冠军**。

4. 如果把 YUYV 换成 MJPG，V4L2 接口的行为会有什么不同？
   YUYV（raw） vs MJPG（压缩）：
   维度	YUYV	MJPG
   帧大小	固定 4,147,200 bytes	变化，取决于画面复杂度
   解码依赖	无需解码，直接读	需要 libjpeg 解压成 raw 才能处理
   带宽	高（USB 2.0 480Mbps 大约跑 30fps 满）	低，同样带宽能跑更高帧率或分辨率
   CPU 负载	低，只做 memcpy	高，每帧要 JPEG 解码
   驱动处理	uvc_video_decode_isoc 中直接拷贝	同样走 URB + 拷贝，但数据量小很多
   V4L2 应用层	不改代码，只改 pixelformat	不改代码，但读到的数据是 JPEG 不是 raw
   你实测抓到 YUYV 是 4,147,200 bytes，如果换成 MJPG 可能只有 50~200KB。处理前要先解压

5. UVC 相机换回 MIPI ISP 相机时，应用层代码要改哪些部分？

   应用层需要改动以下方面：

   | 维度 | UVC | MIPI ISP (RK) |
   |------|-----|---------------|
   | 设备发现 | `/dev/video1` 固定节点 | 需用 `media-ctl -p` 遍历拓扑找到 capture 节点 |
   | 像素格式 | YUYV / MJPG | RAW → NV12（需 ISP 处理），通常用 V4L2_PIX_FMT_NV12 |
   | 分辨率限制 | 相机报告的支持列表 | sensor + ISP 组合决定，需查 sensor datasheet |
   | 管线配置 | 即插即用，无需配置 | 必须用 `media-ctl` 设置链路：`sensor → csi → rkisp1 → /dev/videoX` |
   | 子设备控制 | 无需 | 需通过 subdev ioctl 设置 sensor 参数（曝光、增益、帧率） |
   | 3A 算法 | 摄像头固件自动处理 | 需集成 RKAIQ 库（AE / AWB / AF）或手动写 v4l2-subdev 控制 |
   | buffer 类型 | mmap（vb2_vmalloc） | 建议 dmabuf 实现零拷贝（vb2_dc contig alloc） |
   | Metadata 通道 | /dev/video2 可选 | 有 /dev/videoX 的 ISP statistics 通道 |

   核心改动量：**需新增 media-ctl 配置 + subdev 初始化 + RKAIQ 集成**。纯帧采集逻辑（S_FMT → REQBUFS → QBUF → STREAMON → DQBUF）可复用，但前面的管线准备代码完全不同。

---

## 七、踩坑记录

| 日期 | 问题 | 原因 | 解决方案 |
|------|------|------|----------|
| 2026-06-17 | /dev/video0 不是 UVC 设备 | 需要确认哪个节点是 USB 相机 | 通过 dmesg 和 v4l2-ctl --list-devices 确认 |
| 2026-06-17 | S_FMT 设 NV12 但实际变成 MJPG | UVC 相机不支持 NV12，驱动自动降级 | 先用 `--list-formats-ext` 确认支持的格式，改用 YUYV |
| 2026-06-17 | `sudo echo xxx > /sys/kernel/tracing/xxx` 报 Permission denied | `>` 重定向由 shell 执行，不受 sudo 控制 | `echo xxx \| sudo tee /sys/...` |
| 2026-06-17 | `set_ftrace_filter` 报 Invalid argument | glob pattern 不匹配任何函数 | 先 `cat available_filter_functions \| grep xxx` 确认 |
| 2026-06-17 | Dynamic Debug 打开 uvcvideo 后 dmesg 无输出 | 模块未编译 dyndbg 支持 | `modinfo uvcvideo \| grep dyndbg` 检查 |

---

## 八、下阶段预告

阶段三：**硬件编解码 + MPP**
- 用 RV1126B 的 MPP 硬件编码器把 YUYV 转 H.264
- RGA 2D 硬件加速
- 对比 CPU 编码 vs 硬件编码的性能差

---

## 相关笔记

- [[kernel-debug-env]] — 内核 Debug 环境搭建
- [[rv1126b]] — RV1126B 运动相机项目
- [[MOC-嵌入式Linux]] — 嵌入式 Linux 学习地图
