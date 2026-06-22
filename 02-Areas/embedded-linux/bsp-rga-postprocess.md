---
tags:
  - embedded-linux
  - camera
  - rga
  - 2d-acceleration
  - eis
  - osd
  - vpss
  - rockchip
  - dji
category: embedded-linux
created: 2026-06-22
updated: 2026-06-22
status: active
soc: Rockchip RV1126B
kernel: Linux 6.1.141
rga: RGA2
vpss: V20
---

# 阶段十一：RGA 2D 加速 + 图像后处理

> **大疆相机核心**：预览缩放、OSD 水印、EIS 防抖校正都依赖 2D 硬件加速。大疆运动相机 (Osmo Action) 的 EIS 尤其依赖 GDC 畸变校正。
>
> 本章覆盖 RGA 硬件加速、VPSS 后处理、EIS 防抖、OSD 叠加四个主题。

---

## 一、RGA 硬件

### 1.1 RGA 功能

```
RGA (Raster Graphic Acceleration) — 2D 图形硬件加速器

支持操作:
  ├── 格式转换: NV12 → RGB888, YUYV → NV12, ...
  ├── 缩放: 4K → 1080p, 1080p → 720p (双线性/双三次)
  ├── 旋转: 0/90/180/270 度
  ├── 镜像: 水平/垂直翻转
  ├── 裁剪: 从大图中截取区域
  ├── Blending: 两张图叠加 (Alpha 混合)
  └── 颜色填充: 矩形区域填充
```

### 1.2 RGA 在相机管线中的作用

```
[ISP] → NV12 4K
  │
  ├──→ [RGA 缩放] → 1080p NV12 → [VO] 显示
  ├──→ [RGA 旋转] → 竖屏 1080x1920 → [VO] 显示
  ├──→ [RGA 格式转换] → RGB888 → [NPU] AI 推理
  └──→ [RGA 裁剪] → 人脸区域 → [NPU] 人脸识别

[OSD 生成] → 时间戳/Logo PNG
  └──→ [RGA Blending] 叠加到预览/编码帧上
```

### 1.3 RV1126B RGA 版本

| 版本 | 驱动 | 特性 |
|------|------|------|
| RGA2 | `drivers/video/rockchip/rga2/` | 缩放/旋转/格式转换/Blending |
| RGA3 | `drivers/video/rockchip/rga3/` | 更高性能, 更多格式 |

```dts
/* rv1126b.dtsi */
rga2_core0: rga@20a00000 {
    compatible = "rockchip,rga2";
    reg = <0x20a00000 0x1000>;
    interrupts = <GIC_SPI 180 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&cru ACLK_RGA>, <&cru HCLK_RGA>;
    clock-names = "aclk_rga", "hclk_rga";
    status = "disabled";
};
```

---

## 二、librga API

### 2.1 im2d 接口 (推荐)

```c
#include "im2d.h"
#include "rga.h"

/* im2d 是 librga 的高层 API, 简化操作 */

/* 1. 格式转换: NV12 → RGB888 */
rga_buffer_t src = wrapbuffer_virtualaddr(nv12_ptr, 1920, 1080,
                                          RK_FORMAT_YCbCr_420_SP);
rga_buffer_t dst = wrapbuffer_virtualaddr(rgb_ptr, 1920, 1080,
                                          RK_FORMAT_BGR_888);
imcvtcolor(src, dst, RK_FORMAT_YCbCr_420_SP, RK_FORMAT_BGR_888);

/* 2. 缩放: 1920x1080 → 1280x720 */
rga_buffer_t src = wrapbuffer_virtualaddr(src_ptr, 1920, 1080,
                                          RK_FORMAT_YCbCr_420_SP);
rga_buffer_t dst = wrapbuffer_virtualaddr(dst_ptr, 1280, 720,
                                          RK_FORMAT_YCbCr_420_SP);
imresize(src, dst);

/* 3. 旋转 90 度 */
imrotate(src, dst, 90);

/* 4. 裁剪: 从 (100,100) 截取 640x480 */
im_rect rect = {100, 100, 640, 480};
imcrop(src, dst, rect);

/* 5. Blending: 叠加水印 */
imblend(bg, logo, dst);
```

### 2.2 imcheck 验证参数

```c
/* 执行前验证参数合法性 */
IM_STATUS status = imcheck(src, dst, src_rect, dst_rect);
if (status != IM_STATUS_NOERROR) {
    printf("RGA params invalid: %s\n", imStrError(status));
    return -1;
}
```

---

## 三、VPSS 后处理

### 3.1 VPSS V20 在管线中的位置

```
[ISP mainpath] → [VPSS] → 多路输出
                    │
                    ├── 主码流 (4K → MPP 编码)
                    ├── 预览流 (1080p → VO 显示)
                    └── 分析流 (720p → NPU)
```

VPSS 和 RGA 的功能有重叠（都能缩放/格式转换），但：

| 维度 | VPSS | RGA |
|------|------|-----|
| 位置 | ISP 后, 管线内 (在线) | 独立, 管线外 (离线) |
| 延迟 | 最低 (在线处理) | 需要额外一次 DDR 读写 |
| 灵活性 | 固定功能 | 任意操作 |
| 适用 | 管线内多路输出 | 管线外图像处理 |

> **大疆策略**：管线内用 VPSS (低延迟多路输出)，管线外用 RGA (OSD/AI 预处理)。

---

## 四、EIS 电子防抖

### 4.1 EIS 原理

```
问题: 手持/飞行时画面抖动

EIS 解决方案:
  1. 陀螺仪 (IMU) 记录机身运动
  2. 预测画面运动量 (基于 IMU 数据)
  3. 对每帧做反向补偿 (平移+旋转)
  4. 裁剪掉边缘补偿产生的黑边

数据流:
  IMU 陀螺仪数据 ──→ EIS 算法 ──→ GDC 参数
                                    ↓
  原始帧 ──→ [GDC 畸变校正] ──→ 稳定帧
              (平移+旋转+裁剪)
```

### 4.2 RV1126B EIS 实现

```c
/* rkipc 使用 RkEis 库 */
/* app/rkipc/src/rv1126b_dv/video/rkeis_config_*.json */

/* EIS 配置 */
{
    "eis_mode": 1,           // 0=关, 1=开
    "eis_strength": 0.8,     // 防抖强度
    "crop_ratio": 0.9,       // 裁剪比例 (0.9 = 裁掉 10% 边缘)
    "imu_sample_rate": 200,  // IMU 采样率 Hz
    "motion_filter": "kalman" // 运动滤波
}

/* Rockit API */
RK_MPI_GDC_SetAttr(gdc_handle, &gdc_attr);
/* gdc_attr 包含 EIS 计算的变换矩阵 */
```

> **大疆 EIS**：Osmo Action 的 "RockSteady" 和 Mavic 的 "超采防抖" 都是 EIS + GDC 实现。大疆的优势在于 IMU 数据和帧的精确时间对齐。

---

## 五、OSD 水印叠加

### 5.1 OSD 在管线中的位置

```
[VI] → [VENC] → H.265 码流
        ↑
    [RGN] → OSD 叠加 (时间戳/Logo/传感器信息)
```

### 5.2 Rockit OSD API

```c
/* rkipc 中的 OSD 实现 */
RK_MPI_RGN_Create(rgn_handle, &region_attr);

/* 设置 OSD 内容 (RGBA 位图) */
BITMAP_S bitmap;
bitmap.pixelFormat = RK_FMT_ARGB8888;
bitmap.width = 200;
bitmap.height = 40;
bitmap.pData = timestamp_rgba_data;  // 渲染好的时间戳图片
RK_MPI_RGN_SetBitMap(rgn_handle, &bitmap);

/* 设置 OSD 位置 */
POINT_S pos = {10, 10};  // 左上角偏移
RK_MPI_RGN_SetDisplayPos(rgn_handle, &pos);
```

### 5.3 时间戳渲染

```c
/* 用 freetype 渲染文字到 RGBA buffer */
/* rkipc 使用 freetype 库 */
#include <ft2build.h>
#include FT_FREETYPE_H

/* 渲染 "2026-06-22 14:30:00" 到 RGBA buffer */
FT_Render_Text(font, "2026-06-22 14:30:00", rgba_buf, 200, 40);
/* 然后通过 RK_MPI_RGN_SetBitMap 叠加到编码流 */
```

---

## 六、实验 1：RGA 格式转换 + 缩放

### 6.1 实验目标

用 librga 将 NV12 帧转换为 RGB888 并缩放到 640x360，对比 CPU 和 RGA 的性能。

### 6.2 操作步骤

```bash
# 检查 RGA 驱动
ls /dev/rga
# 或
cat /proc/rga 2>/dev/null

# 编译 RGA 测试程序
aarch64-none-linux-gnu-gcc -o rga_test rga_test.c \
    -I external/linux-rga/include/ -lrga

# 板端运行
sudo /tmp/rga_test /tmp/isp_frame.nv12 /tmp/output.rgb 640 360
# 预期: RGA 转换 + 缩放 ~0.5ms
# 对比: CPU 逐像素转换 ~5-10ms
```

### 6.3 RGA 测试程序 (rga_test.c)

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "im2d.h"
#include "rga.h"

int main(int argc, char **argv)
{
    const char *in_file = argv[1];
    const char *out_file = argv[2];
    int dst_w = atoi(argv[3]);
    int dst_h = atoi(argv[4]);

    int src_w = 1920, src_h = 1080;

    /* 读取 NV12 输入 */
    FILE *fp = fopen(in_file, "rb");
    unsigned char *src = malloc(src_w * src_h * 3 / 2);
    fread(src, 1, src_w * src_h * 3 / 2, fp);
    fclose(fp);

    /* 分配输出 buffer */
    unsigned char *dst = malloc(dst_w * dst_h * 3);  /* RGB888 */

    /* RGA 操作: NV12 → RGB888 + 缩放 */
    rga_buffer_t rga_src = wrapbuffer_virtualaddr(src, src_w, src_h,
                                                   RK_FORMAT_YCbCr_420_SP);
    rga_buffer_t rga_dst = wrapbuffer_virtualaddr(dst, dst_w, dst_h,
                                                   RK_FORMAT_BGR_888);

    IM_STATUS ret = imresize(rga_src, rga_dst);
    if (ret != IM_STATUS_SUCCESS) {
        printf("RGA failed: %s\n", imStrError(ret));
        return 1;
    }
    ret = imcvtcolor(rga_src, rga_dst,
                     RK_FORMAT_YCbCr_420_SP, RK_FORMAT_BGR_888);

    /* 保存 */
    fp = fopen(out_file, "wb");
    fwrite(dst, 1, dst_w * dst_h * 3, fp);
    fclose(fp);

    printf("RGA: %dx%d NV12 → %dx%d RGB done\n", src_w, src_h, dst_w, dst_h);
    return 0;
}
```

---

## 七、实验 2：RGA 旋转 (竖屏预览)

### 7.1 实验目标

用 RGA 将 1920x1080 横屏帧旋转为 1080x1920 竖屏。

### 7.2 操作

```c
/* 旋转 90 度 */
rga_buffer_t src = wrapbuffer_virtualaddr(src_ptr, 1920, 1080,
                                          RK_FORMAT_YCbCr_420_SP);
rga_buffer_t dst = wrapbuffer_virtualaddr(dst_ptr, 1080, 1920,
                                          RK_FORMAT_YCbCr_420_SP);
imrotate(src, dst, 90);
/* 宽高互换: 1920x1080 → 1080x1920 */
```

> **大疆运动相机**：Osmo Action 的竖屏模式就是用 RGA 旋转实现的。

---

## 八、实验 3：OSD 水印叠加

### 8.1 实验目标

用 RGA Blending 将时间戳文字叠加到视频帧上。

### 8.2 操作

```c
/* 1. 生成时间戳 RGBA 图片 (用 freetype 或直接画) */
/* 2. 用 RGA 叠加 */

rga_buffer_t bg = wrapbuffer_virtualaddr(frame_nv12, 1920, 1080,
                                          RK_FORMAT_YCbCr_420_SP);
rga_buffer_t logo = wrapbuffer_virtualaddr(timestamp_rgba, 200, 40,
                                            RK_FORMAT_BGRA_8888);
rga_buffer_t dst = wrapbuffer_virtualaddr(output_nv12, 1920, 1080,
                                           RK_FORMAT_YCbCr_420_SP);

im_rect bg_rect = {0, 0, 1920, 1080};
im_rect logo_rect = {10, 10, 200, 40};  /* 左上角 */

imblend(bg, dst, logo, logo_rect, bg_rect);
/* Alpha 混合: logo 的透明部分透出背景 */
```

---

## 九、RGA 性能对比

| 操作 | CPU 耗时 | RGA 耗时 | 加速比 |
|------|---------|---------|--------|
| NV12→RGB888 (1920x1080) | ~8ms | ~0.5ms | 16x |
| 缩放 1080p→720p | ~5ms | ~0.3ms | 17x |
| 旋转 90° (1080p) | ~6ms | ~0.4ms | 15x |
| Blending (1080p + 200x40 logo) | ~2ms | ~0.1ms | 20x |

> **结论**：RGA 硬件加速比 CPU 快 15-20 倍。在实时管线中 (30fps = 33ms/帧)，省下的时间可用于其他处理。

---

## 十、思考题

1. VPSS 和 RGA 都能做缩放，为什么管线内用 VPSS 而管线外用 RGA？如果用 RGA 替代 VPSS 做管线内缩放，延迟会增加多少？

2. EIS 防抖需要 IMU 陀螺仪数据和视频帧精确时间对齐。如果 IMU 采样率和帧率不同 (如 IMU 200Hz, 视频 30fps)，如何对齐？误差对防抖效果有什么影响？

3. RGA 的 Blending 需要 Alpha 通道。NV12 格式没有 Alpha 通道，如何用 RGA 在 NV12 帧上叠加 RGBA 水印？

4. 大疆 Osmo Action 的竖屏模式：如果 Sensor 输出是 4K 横屏 (3840x2160)，竖屏预览需要旋转+裁剪。用 RGA 如何实现？裁剪后的分辨率是多少？

5. 大疆的 "超采防抖" (Mavic 3) 利用 4K Sensor 读取全部像素，然后裁剪到 1080p 输出。这和传统 EIS 裁剪有什么区别？画质优势在哪里？

---

## 十一、踩坑记录

| 日期 | 问题 | 原因 | 解决方案 |
|------|------|------|----------|
| | imresize 返回 IM_STATUS_ERROR | 宽高不对齐 | RGA 要求宽高 4 字节对齐 |
| | 旋转后画面变形 | 宽高未互换 | 旋转 90/270 时 dst 宽高要互换 |
| | RGA 驱动未加载 | DTS 中 rga status=disabled | 板级 DTS 中启用 |
| | Blending 透明区域变黑 | Alpha 格式不对 | 用 BGRA_8888 (有 Alpha) |
| | EIS 画面抖动加剧 | IMU 时间戳不对齐 | 校准 IMU 和帧时间戳 |

---

## 十二、下阶段预告

阶段十二：**相机 Capstone — 完整相机管线项目**
- 从零搭建：Sensor → ISP → 3A → MPP → RGA → 文件/显示
- 完整管线代码 + 性能测量 + 设计文档

---

## 相关笔记

- [[MOC-嵌入式Linux]] — 学习路线总览
- [[bsp-v4l2-isp-pipeline]] — 阶段八：ISP 输出 (RGA 输入)
- [[bsp-mpp-codec-pipeline]] — 阶段十：MPP 编码 (OSD 叠加)
