---
tags:
  - embedded-linux
  - camera
  - mpp
  - hardware-codec
  - h264
  - h265
  - rockit
  - video-encode
  - rockchip
  - dji
category: embedded-linux
created: 2026-06-22
updated: 2026-06-22
status: active
soc: Rockchip RV1126B
kernel: Linux 6.1.141
encoder: VEPU511
mpp_version: 1.0.11
---

# 阶段十：MPP 硬件编解码管线

> **大疆相机核心**：相机录像和图传的核心是硬件 H.265 编码。大疆 O4 图传做到 <20ms 端到端延迟，依赖的就是紧耦合的硬件编码管线。
>
> 本章从 MPP MPI 接口到 Rockit 框架，理解相机"采集→编码→存储/传输"的完整链路。

---

## 一、MPP 在相机管线中的位置

```
[Sensor] → [ISP] → [VPSS] → [MPP 编码] → [H.265 码流]
                                        ↓
                                 ┌─── 文件存储 (MP4)
                                 ├─── RTSP 推流
                                 └─── 图传 (O4 SDR)
```

MPP (Media Process Platform) 接收 ISP/VPSS 输出的 NV12 帧，硬件编码为 H.264/H.265 码流。

---

## 二、RV1126B 编解码硬件

| 硬件 | 型号 | 编码 | 解码 |
|------|------|------|------|
| 编码器 | VEPU511 | H.264/H.265/JPEG | — |
| 解码器 | VDPU384A | — | H.264/H.265/JPEG |
| 解码器 | VDPU720 | — | JPEG |

### 2.1 编码性能参考

| 分辨率 | 帧率 | 编码格式 | 码率 | VEPU511 能力 |
|--------|------|---------|------|-------------|
| 4K (3840×2160) | 30fps | H.265 | 50Mbps | 支持 |
| 1080p | 60fps | H.265 | 20Mbps | 支持 |
| 1080p | 30fps | H.264 | 10Mbps | 支持 |
| 720p | 120fps | H.264 | 8Mbps | 支持 |

> **大疆图传**：O4 使用自研 P1 SoC 做 H.265 编码 + SDR 调制，延迟 <20ms。RV1126B 的 VEPU511 是同类硬件，可以学习编码管线的原理。

---

## 三、MPP 架构

### 3.1 层次结构

```
应用层 (rkipc / 自定义程序)
  │
  ├── Rockit 框架 (RK_MPI_* API)
  │   ├── VI (Video Input)  ← 从 ISP/VPSS 获取帧
  │   ├── VENC (Video Encode) ← 硬件编码
  │   ├── VO (Video Output)  ← 显示输出
  │   ├── VENC + VO 绑定管线
  │   └── RGN (Region/OSD) ← 水印叠加
  │
  ├── MPP MPI (mpp_create / encode_put_frame)
  │   └── 底层 MPI 接口 (更灵活但更复杂)
  │
  ├── RKAIQ (3A) ← 阶段九
  │
  └── RGA (2D加速) ← 阶段十一

内核层
  └── mpp_service 驱动 (/dev/mpp_service)
      ├── rkvenc (VEPU511 编码器)
      └── rkvdec (VDPU384A 解码器)
```

### 3.2 Rockit vs MPP MPI

| 维度 | Rockit (RK_MPI_*) | MPP MPI (mpp_*) |
|------|-------------------|-----------------|
| 抽象层级 | 高 (管线自动管理) | 低 (手动管理) |
| VI/VENC/VO 绑定 | 自动 | 手动 |
| 适用场景 | 产品级应用 | 学习/定制 |
| rkipc 使用 | ✅ | 内部调用 |
| 灵活性 | 中 | 高 |

> **学习建议**：先用 MPP MPI 理解编码原理（阶段十实验），再用 Rockit 理解完整管线（阶段十二 Capstone）。

---

## 四、MPP MPI 编码流程

### 4.1 同步编码 API

```c
#include "rk_mpi.h"

/* 1. 创建 MPP 实例 */
MppCtx ctx;
MppApi *mpi;
mpp_create(&ctx, &mpi);

/* 2. 初始化为 H.265 编码器 */
mpp_init(ctx, MPP_CTX_ENC, MPP_VIDEO_CodingHEVC);

/* 3. 配置编码参数 */
MppEncCfg cfg;
mpp_enc_cfg_init(&cfg);
mpp_enc_cfg_set_s32(cfg, "prep:width", 1920);
mpp_enc_cfg_set_s32(cfg, "prep:height", 1080);
mpp_enc_cfg_set_s32(cfg, "prep:format", MPP_FMT_YUV420SP); // NV12
mpp_enc_cfg_set_s32(cfg, "rc:mode", MPP_ENC_RC_MODE_CBR);
mpp_enc_cfg_set_s32(cfg, "rc:bps_target", 4000000);  // 4Mbps
mpp_enc_cfg_set_s32(cfg, "rc:fps_in_num", 30);
mpp_enc_cfg_set_s32(cfg, "rc:gop", 30);
mpp_enc_cfg_set_s32(cfg, "codec:type", MPP_VIDEO_CodingHEVC);
mpi->control(ctx, MPP_ENC_SET_CFG, cfg);

/* 4. 获取 SPS/PPS 头 */
MppPacket header;
mpi->control(ctx, MPP_ENC_GET_EXTRA_INFO, &header);
/* 写入文件头 */

/* 5. 编码循环 */
for (each frame) {
    /* 准备输入帧 */
    MppFrame frame;
    mpp_frame_init(&frame);
    mpp_frame_set_fmt(frame, MPP_FMT_YUV420SP);
    mpp_frame_set_buffer(frame, mpp_buf);

    /* 编码 (阻塞) */
    mpi->encode_put_frame(ctx, frame);

    /* 获取码流 */
    MppPacket packet;
    mpi->encode_get_packet(ctx, &packet);
    /* 写入文件 */
    mpp_packet_deinit(&packet);
    mpp_frame_deinit(&frame);
}

/* 6. 销毁 */
mpp_destroy(ctx);
```

### 4.2 关键编码参数

| 参数 | 说明 | 大疆关注点 |
|------|------|-----------|
| `rc:mode` | CBR/VBR/FIX_QP | 图传用 CBR (固定码率, 网络稳定) |
| `rc:bps_target` | 目标码率 | 4K@30: 50Mbps; 1080p@30: 4-10Mbps |
| `rc:gop` | GOP 大小 | 图传用小 GOP (低延迟); 录像用大 GOP (压缩率) |
| `rc:fps_in_num` | 输入帧率 | 30/60/120 |
| `h264:profile` | H.264 档次 | High (100); 图传用 Baseline (低延迟) |
| `h265:profile` | H.265 档次 | Main (1) |
| `split:mode` | Slice 分割 | 图传用小 slice (低延迟, 单包小) |
| `rc:qp_init` | 初始 QP | 影响首帧质量 |
| `rc:qp_max/qp_min` | QP 范围 | 限制质量波动 |

### 4.3 H.264 vs H.265 对比

| 维度 | H.264 | H.265 | 大疆选择 |
|------|-------|-------|---------|
| 压缩效率 | 基准 | 比 H.264 省 30-50% 码率 | H.265 (节省带宽) |
| 编码延迟 | 略低 | 略高 | 图传场景权衡 |
| 硬件支持 | VEPU511 | VEPU511 | 都支持 |
| 兼容性 | 最广 | 较广 | H.264 兼容性好 |
| 4K 支持 | 支持 | 更适合 | H.265 适合 4K |

> **大疆图传选择**：O4 用 H.265 (节省 30% 带宽 → 更远距离/更高画质)。录像也用 H.265。

---

## 五、Rockit 框架 (产品级管线)

### 5.1 Rockit 管线架构

```
rkipc 使用 Rockit 搭建的完整管线:

  [VI] ← ISP/VPSS 输出
   │
   ├──→ [VENC] → H.265 码流 → 文件 (MP4) / RTSP
   │
   └──→ [VO] → 显示 (MIPI DSI / HDMI)

  [RGN] → OSD 水印叠加到 VENC/VO
  [GDC] → EIS 防抖校正
```

### 5.2 RK_MPI API (rkipc 实际使用)

```c
/* rkipc video.c 中的 Rockit 管线 */

/* 1. VI (Video Input) — 从 ISP 获取帧 */
RK_MPI_VI_SetDevAttr(vi_dev, &vi_attr);
RK_MPI_VI_EnableDev(vi_dev);
RK_MPI_VI_SetChnAttr(vi_chn, &chn_attr);
RK_MPI_VI_EnableChn(vi_chn);

/* 2. VENC (Video Encode) — 硬件编码 */
RK_MPI_VENC_CreateChn(venc_chn, &venc_attr);
/* venc_attr 包含: H.265, 4K, 50Mbps, CBR, GOP=30 */

/* 3. VO (Video Output) — 显示 */
RK_MPI_VO_BindLayer(vo_layer, vi_chn);
RK_MPI_VO_SetLayerAttr(vo_layer, &layer_attr);
RK_MPI_VO_EnableLayer(vo_layer);

/* 4. 绑定管线: VI → VENC */
RK_MPI_SYS_Bind(&vi_chn, &venc_chn);
/* 绑定后, VI 每帧自动送给 VENC, 无需手动 */

/* 5. 获取编码码流 */
RK_MPI_VENC_GetStream(venc_chn, &stream, timeout);
/* 写入文件或推流 */
RK_MPI_VENC_ReleaseStream(venc_chn, &stream);

/* 6. OSD 水印 */
RK_MPI_RGN_SetBitMap(rgn_handle, &bitmap);
```

> **Rockit 的优势**：`RK_MPI_SYS_Bind` 自动连接 VI→VENC，帧在管线中自动流动，无需用户态手动传递。这是产品级代码的高效模式。

---

## 六、实验 1：MPP 编码一帧 NV12 → H.265

### 6.1 实验目标

用 MPP MPI 将一帧 NV12 (来自 ISP 或测试图) 编码为 H.265，验证编码流程。

### 6.2 操作步骤

```bash
# 使用附录C 的 mpp_enc_oneframe 程序 (已生成)
# 输入: NV12 帧 (来自阶段八的 ISP 采集)
# 输出: H.265 码流

# 板端:
# 如果有 ISP 采集的 NV12 帧
sudo /tmp/mpp_enc_oneframe /tmp/isp_frame.nv12 /tmp/output.h265

# 如果没有 ISP 帧, 生成测试图
python3 -c "
import os
w, h = 1920, 1080
# Y 平面: 渐变
y = bytes([int(i*256/w) for i in range(w)] * h)
# UV 平面: 交替
uv = bytes([128, 128] * (w * h // 4))
with open('/tmp/test.nv12', 'wb') as f:
    f.write(y + uv)
"
sudo /tmp/mpp_enc_oneframe /tmp/test.nv12 /tmp/output.h265

# 验证
ls -la /tmp/output.h265
# 预期: 几 KB (单帧 I-frame)

hexdump -C /tmp/output.h265 | head -5
# H.265 起始码: 00 00 00 01 40 ... (VPS)
#              00 00 00 01 42 ... (SPS)
#              00 00 00 01 44 ... (PPS)
#              00 00 00 01 26 ... (IDR slice)
```

### 6.3 编码耗时测量

```bash
# 在程序中添加时间测量
# 或用 time 命令
time sudo /tmp/mpp_enc_oneframe /tmp/test.nv12 /tmp/output.h265

# 预期:
# real ~10ms (硬件编码一帧 1080p)
# 对比: x264 软件编码同样帧 ~30-50ms
```

---

## 七、实验 2：连续编码多帧 (模拟录像)

### 7.1 实验目标

循环编码多帧，模拟相机录像场景。

### 7.2 程序修改

```c
/* 在 mpp_enc_oneframe.c 基础上改为循环编码 */
/* 关键: encode_put_frame + encode_get_packet 循环 */

for (int i = 0; i < 100; i++) {
    /* 读取第 i 帧 NV12 (从文件或 ISP) */
    /* ... */

    /* 编码 */
    mpi->encode_put_frame(ctx, frame);
    mpi->encode_get_packet(ctx, &packet);

    /* 写入文件 */
    fwrite(packet_data, 1, packet_len, fp);
    mpp_packet_deinit(&packet);
    mpp_frame_deinit(&frame);
}
```

### 7.3 验证

```bash
# 编码 100 帧
sudo /tmp/mpp_enc_multi /tmp/frames/ /tmp/video.h265 100

# 用 ffplay 播放
scp rooter@192.168.1.109:/tmp/video.h265 .
ffplay -f h265 video.h265

# 检查码流信息
ffprobe -i video.h265
# 预期: H.265, 1920x1080, 30fps, CBR 4Mbps
```

---

## 八、实验 3：编码参数调优

### 8.1 实验目标

调整 GOP、码率、QP 范围，对比画质和码流大小。

### 8.2 参数对比

```bash
# 1. 低延迟配置 (图传场景)
# GOP=1 (全 I 帧), 高码率, 无 B 帧
# rc:gop=1, rc:bps_target=20000000

# 2. 高压缩配置 (录像场景)
# GOP=60, 中码率, 有 B 帧
# rc:gop=60, rc:bps_target=8000000

# 3. 固定质量配置
# FIX_QP 模式, qp=28
# rc:mode=FIX_QP, rc:qp_init=28

# 对比同一段视频:
# - 文件大小
# - 画质 (PSNR / 主观)
# - 编码耗时
```

| 配置 | GOP | 码率 | 文件大小 | 画质 | 延迟 | 适用 |
|------|-----|------|---------|------|------|------|
| 低延迟 | 1 | 20Mbps | 大 | 中 | 最低 | 图传 |
| 平衡 | 30 | 8Mbps | 中 | 高 | 中 | 录像 |
| 高压缩 | 60 | 4Mbps | 小 | 中高 | 高 | 存储 |

---

## 九、实验 4：Ftrace 追踪 MPP 编码路径

### 9.1 操作步骤

```bash
echo function_graph | sudo tee /sys/kernel/tracing/current_tracer
echo '*mpp*|*rkvenc*|*vepu*' | sudo tee /sys/kernel/tracing/set_ftrace_filter
echo 256 | sudo tee /sys/kernel/tracing/max_graph_depth

echo | sudo tee /sys/kernel/tracing/trace
sudo /tmp/mpp_enc_oneframe /tmp/test.nv12 /tmp/output.h265
sudo cat /sys/kernel/tracing/trace > /tmp/mpp_trace.log
echo nop | sudo tee /sys/kernel/tracing/current_tracer
```

### 9.2 预期调用链

```
用户态:
  encode_put_frame()
    → ioctl(fd, MPP_CMD_SEND_FRAME)
      ↓ 内核态
  mpp_service_ioctl()
    → mpp_venc_send_frame()
      → rkvenc_enc_one_frame()
        → 配置 VEPU511 硬件寄存器
        → 启动编码 (写寄存器触发)
        → wait_for_completion()  ← 等待硬件中断
          → rkvenc_irq_handler() ← 编码完成中断
            → complete()          ← 唤醒
        → 读取编码结果
  → 回到用户态
  encode_get_packet()
```

> **关键发现**：MPP 编码是"中断驱动"的——发起编码后 CPU 睡眠等待，VEPU511 硬件完成后中断唤醒。编码期间 CPU 几乎零占用。

---

## 十、rkipc 录像管线分析

### 10.1 rkipc 配置

```ini
# /etc/rkipc.ini
[video.source]
camera_id = 0
width = 3840    # 4K
height = 2160
fps = 30

[video.0]       # 主码流
width = 3840
height = 2160
codec = H.265
rc_mode = CBR
max_rate = 50000   # 50Mbps
fps = 30
gop = 30

[display]
width = 1080
height = 1920
type = MIPI
rotation = 1       # 竖屏
```

### 10.2 rkipc 管线流程

```
1. RK_MPI_SYS_Init()          — MPP 系统初始化
2. rk_isp_init(0, "/etc/iqfiles") — RKAIQ 3A 启动
3. rk_video_init()            — 视频管线搭建
   ├── VI: 从 ISP 获取 4K NV12
   ├── VENC: H.265 编码 (50Mbps CBR)
   ├── VO: 1080p 显示 (旋转 90°)
   ├── Bind: VI → VENC (自动)
   └── Bind: VI → VO (自动)
4. 录像: VENC GetStream → 写 MP4 文件
5. 预览: VI → VO → MIPI 显示
```

---

## 十一、思考题

1. 大疆图传要求 <20ms 端到端延迟。从 Sensor 采集到编码输出的延迟由哪些部分组成？每个部分大约多少 ms？如何优化？

2. H.265 的 GOP=1 (全 I 帧) 会让码率暴涨，但延迟最低。大疆图传用什么策略在"低延迟"和"合理码率"间平衡？提示：Slice 分割 + 小 GOP。

3. Rockit 的 `RK_MPI_SYS_Bind` 自动连接 VI→VENC，帧在内核中自动流动。这比用户态手动 GetFrame/SendFrame 有什么优势？零拷贝体现在哪里？

4. 如果编码器的输出码率不稳定（某些帧特别大），在图传中会导致什么问题？CBR 码率控制如何解决？

5. 大疆相机支持"边录边传"——同时录像 (高码率 H.265 存文件) 和图传 (低码率 H.264 传输)。在 Rockit/MPP 中如何实现双路编码？

---

## 十二、踩坑记录

| 日期 | 问题 | 原因 | 解决方案 |
|------|------|------|----------|
| | mpp_create 失败 | MPP 库未安装 | `ls /lib/librockchip_mpp*` 确认 |
| | encode_put_frame 阻塞 | 硬件编码器未使能 | 检查 `dmesg | grep mpp` |
| | 输出 0 字节 | SPS/PPS 未获取 | 检查 MPP_ENC_GET_EXTRA_INFO |
| | 画面花屏 | NV12 数据不对齐 | 确认 hor_stride/ver_stride 对齐 |
| | 编码延迟高 | GOP 太大或 B 帧多 | 图传用 GOP=1 或小 slice |

---

## 十三、下阶段预告

阶段十一：**RGA 2D 加速 + 图像后处理**
- RGA 硬件：缩放、旋转、格式转换、裁剪、blending
- librga API (im2d)
- VPSS 后处理 + EIS 防抖
- OSD 水印叠加

---

## 相关笔记

- [[MOC-嵌入式Linux]] — 学习路线总览
- [[bsp-v4l2-isp-pipeline]] — 阶段八：ISP 输出 NV12 (MPP 输入)
- [[bsp-isp-3a-rkaiq]] — 阶段九：3A 算法 (画质基础)
- [[mpp-hardware-codec]] — 附录C：MPP 编码实验 (基础数据)
