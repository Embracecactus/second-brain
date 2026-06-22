---
tags:
  - embedded-linux
  - camera
  - isp
  - 3a
  - ae
  - awb
  - af
  - rkaiq
  - image-quality
  - rockchip
  - dji
category: embedded-linux
created: 2026-06-22
updated: 2026-06-22
status: active
soc: Rockchip RV1126B
kernel: Linux 6.1.141
isp: V35
rkaiq: camera_engine_rkaiq (71 algos)
---

# 阶段九：ISP 3A 算法 + RKAIQ 集成

> **大疆相机核心**：3A (AE/AWB/AF) 是相机画质的核心竞争力。大疆面试中 ISP/3A 理解深度直接区分候选人水平。
>
> 本章深入 ISP 处理流水线各模块，理解 3A 算法原理，学习 RKAIQ 架构和 65+ IQ 算法模块，最终能调参观察画质变化。

---

## 一、ISP 处理流水线

### 1.1 Raw → YUV 完整处理链

```
Sensor 输出 (Raw Bayer)
    │
    ▼
┌─────────────────────────────────────────────────┐
│                   ISP V35 流水线                  │
│                                                  │
│  1. BLC (黑电平校正)                              │
│     └─ 消除 Sensor 暗电流偏移                     │
│                                                  │
│  2. DPCC (坏点校正)                               │
│     └─ 检测并修正坏点                             │
│                                                  │
│  3. LSC (镜头阴影校正)                            │
│     └─ 补偿镜头边缘暗角                           │
│                                                  │
│  4. AWB (自动白平衡)                              │
│     └─ 估计光源色温, 调整 R/B 增益                │
│                                                  │
│  5. Demosaic (去马赛克)                           │
│     └─ Bayer → 全彩色 (插值)                      │
│                                                  │
│  6. CCM (色彩校正矩阵)                            │
│     └─ 精确色彩还原                               │
│                                                  │
│  7. Gamma (伽马校正)                              │
│     └─ 线性 → 非线性 (适配显示)                   │
│                                                  │
│  8. NR (降噪)                                     │
│     ├─ Bayer NR (Bayer 域降噪)                   │
│     ├─ TNR (时域降噪, 多帧融合)                    │
│     └─ CNR (色度降噪)                             │
│                                                  │
│  9. Sharpen (锐化)                                │
│     └─ 增强边缘细节                               │
│                                                  │
│  10. CSC (色彩空间转换)                           │
│      └─ RGB → YUV420 (NV12 输出)                 │
│                                                  │
└─────────────────────────────────────────────────┘
    │
    ▼
  YUV420 (NV12) → DDR → 编码/显示
```

### 1.2 各模块对画质的影响

| 模块 | 作用 | 调不好时的现象 | 大疆关注度 |
|------|------|--------------|-----------|
| BLC | 黑电平基准 | 暗部偏色/偏亮 | 高 |
| DPCC | 坏点 | 画面固定位置亮点 | 中 |
| LSC | 镜头暗角 | 四角发暗 | 高 (广角镜头) |
| AWB | 白平衡 | 整体偏黄/偏蓝 | 极高 |
| Demosaic | 去马赛克 | 伪彩色/摩尔纹 | 高 |
| CCM | 色彩还原 | 颜色不准 (红不红, 绿不绿) | 极高 |
| Gamma | 亮度映射 | 画面发灰/对比度不足 | 高 |
| NR | 降噪 | 噪点/涂抹感 | 极高 (夜景) |
| Sharpen | 锐化 | 太软/太锐(锯齿) | 高 |
| HDR/DRC | 动态范围 | 天空过曝/暗部死黑 | 极高 (航拍) |

---

## 二、3A 算法原理

### 2.1 AE (自动曝光)

```
目标: 控制画面亮度到目标值 (不过曝/不欠曝)

控制量:
  - 曝光时间 (Exposure Time): Sensor 寄存器, 单位=行周期
  - 模拟增益 (Analog Gain): Sensor 内部放大器
  - 数字增益 (Digital Gain): ISP 内部乘法

反馈链:
  ISP stats (直方图/亮度统计) → AE 算法 → Sensor 曝光寄存器

AE 算法逻辑:
  1. 读取 ISP 统计的直方图/平均亮度
  2. 计算当前亮度 vs 目标亮度 (EV)
  3. 如果偏暗: 增加曝光时间 → 到极限后增加增益
  4. 如果偏亮: 减少增益 → 到极限后减少曝光时间
  5. 平滑过渡 (避免亮度突变)

权衡:
  - 曝光时间长 → 亮度高但运动模糊
  - 增益高 → 亮度高但噪点多
  - AE 算法在"亮度 vs 模糊 vs 噪点"间权衡
```

### 2.2 AWB (自动白平衡)

```
目标: 让白色物体在画面中看起来是白色 (不偏色)

原理:
  - 不同光源色温不同 (日光~5500K, 白炽灯~3000K, 阴天~6500K)
  - Sensor 的 RGB 响应不匹配人眼
  - 需要调整 R/G/B 三个通道的增益, 使白色=灰色 (R=G=B)

AWB 算法:
  1. 识别画面中的"灰色世界"或"白色区域"
  2. 计算当前 R/B 通道相对 G 的偏移
  3. 调整 R/B 增益使 R≈G≈B (在白色区域)
  4. 常用算法: Gray World, White Patch, 统计域分析

大疆挑战:
  - 航拍场景天空占比大 → 影响白平衡估计
  - 日出日落色温变化快 → 需要快速跟踪
  - 多光源混合 (自然光+人造光)
```

### 2.3 AF (自动对焦)

```
目标: 让主体清晰

类型:
  - Contrast Detection AF (对比度检测): 找对比度最大位置
  - Phase Detection AF (相位检测): 利用 PDAF 像素, 更快
  - Laser AF (激光测距): 主动测距, 最快但距离有限

AF 算法:
  1. 驱动 VCM (音圈马达) 移动镜头
  2. 在每个位置计算对焦值 (FV, Focus Value = 对比度)
  3. 找 FV 最大值位置 → 最佳对焦
  4. 持续跟踪 (如果主体移动)

RV1126B ISP V35:
  - 支持 PDAF (相位检测自动对焦)
  - ISP 输出 PDAF 统计数据
  - AF 算法在 RKAIQ 用户态运行
```

---

## 三、RKAIQ 架构

### 3.1 整体架构

```
┌──────────────────────────────────────────┐
│              RKAIQ 3A Server               │
│         (rkaiq_3A_server, 独立进程)         │
│                                            │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌──────────┐   │
│  │ AE  │ │ AWB │ │ AF  │ │ IQ 模块群  │   │
│  │算法 │ │算法 │ │算法 │ │ (65+ 个)  │   │
│  └──┬──┘ └──┬──┘ └──┬──┘ └─────┬────┘   │
│     │       │       │           │         │
│     └───────┴───────┴───────────┘         │
│                    │                       │
│              uAPI2 接口层                   │
│         (rk_aiq_user_api2_*)               │
└────────────────────┬─────────────────────┘
                     │
         V4L2 subdev ioctl + video node
                     │
┌────────────────────┴─────────────────────┐
│              Kernel ISP 驱动                │
│                                            │
│  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │ stats    │  │ params   │  │ capture │ │
│  │ video    │← │ video    │  │ video   │ │
│  │ (输出统计)│  │(输入参数)│  │(输出帧) │ │
│  └──────────┘  └──────────┘  └─────────┘ │
│                                            │
│  ISP V35 硬件 (BLC→LSC→AWB→...→CSC)      │
└──────────────────────────────────────────┘
```

### 3.2 RKAIQ 71 个算法模块

```
external/camera_engine_rkaiq/rkaiq/algos/
```

| 分类 | 模块 | 说明 |
|------|------|------|
| **3A 核心** | ae, awb, af | 自动曝光/白平衡/对焦 |
| **黑电平** | ablc, ablcV32 | 黑电平校正 |
| **坏点** | adpcc | 坏点检测校正 |
| **镜头阴影** | alsc | 镜头阴影校正 |
| **色彩** | accm, acsm, acgc, a3dlut | 色彩校正矩阵/色彩空间/伽马/3D LUT |
| **去马赛克** | adebayer, argbir | Demosaic 插值 |
| **降噪 (Bayer域)** | abayer2dnr2, abayertnr2, abayertnrV30 | Bayer 域空域+时域降噪 |
| **降噪 (YUV域)** | acnr, acnr2, arawnr, aynr, amfnr, auvnr | 色度/亮度降噪 |
| **锐化** | asharp, asharp3, asharpV33, asharpV34 | 多代锐化算法 |
| **HDR** | amerge, adrc, atmo, awdr | HDR 合并/动态范围压缩/色调映射 |
| **畸变校正** | aldc, aldch | 镜头畸变校正 (LDCH mesh) |
| **防抖** | aeis | 电子防抖 (EIS) |
| **AI 增强** | aiisp, aibnr, aie | AI ISP / AI 降噪 / AI 增强 |
| **其他** | afd, afec, again, agamma, aorb, asd, acac, acp | 闪烁检测/表情/增益/伽马/正交/场景检测/色彩增强 |

### 3.3 IQ 调校文件

```
external/camera_engine_rkaiq/rkaiq/iqfiles/
├── isp35/                    ← RV1126B ISP V35 专用
│   ├── ainr/                 ← AI 降噪参数
│   ├── airms/                ← AI 去马赛克参数
│   └── common/               ← 通用 IQ 参数 (JSON)
├── FEC_mesh_*                ← FEC 畸变校正网格
├── LDCH_mesh_*               ← LDCH 畸变校正网格
└── *.xml                     ← 旧格式 IQ 文件
```

> IQ 文件是每个 Sensor + 镜头组合的"画质调校配方"。大疆有专门的 IQ 工程师团队做调校，这是相机的核心竞争力。

### 3.4 3A Server 运行机制

```bash
# 板端: 3A server 作为独立进程运行
# 启动脚本: /etc/init.d/S40rkaiq_3A

# 运行流程:
# 1. rkipc 启动时调用 rk_aiq_uapi2_sysctl_init()
# 2. 加载 IQ 文件 (iqfiles/isp35/*.json)
# 3. rk_aiq_uapi2_sysctl_prepare() → 准备 3A 算法
# 4. rk_aiq_uapi2_sysctl_start() → 启动 3A 循环
# 5. 每帧: 读 ISP stats → AE/AWB 计算 → 写 ISP params
# 6. 停止: rk_aiq_uapi2_sysctl_stop()
```

---

## 四、rkipc 中的 RKAIQ 使用

### 4.1 rkipc ISP 初始化流程 (app/rkipc/common/isp/rv1126b/isp.c)

```c
/* rkipc main.c 中的相机初始化 */
RK_MPI_SYS_Init();              // MPP 系统初始化
rk_isp_init(0, "/etc/iqfiles"); // RKAIQ 3A 初始化, camera_id=0
rk_isp_set_frame_rate(0, 30);   // 设置帧率
rk_video_init();                // 视频管线初始化

/* isp.c 中 RKAIQ API 调用 */
rk_aiq_uapi2_sysctl_enumStaticMetasByPhyId(...)  // 枚举 sensor
rk_aiq_uapi2_sysctl_preInit(...)                  // 预初始化
rk_aiq_uapi2_sysctl_init(...)                     // 加载 IQ 文件
rk_aiq_uapi2_sysctl_prepare(...)                  // 准备 3A
rk_aiq_uapi2_sysctl_start(...)                    // 启动 3A
```

### 4.2 图像质量调整 API

```c
/* 亮度/对比度/饱和度/锐度/色调 */
rk_aiq_uapi2_setBrightness(ctx, value);    // -100~100
rk_aiq_uapi2_setContrast(ctx, value);
rk_aiq_uapi2_setSaturation(ctx, value);
rk_aiq_uapi2_setSharpness(ctx, value);
rk_aiq_uapi2_setHue(ctx, value);

/* AE 控制 */
rk_aiq_uapi2_setExpMode(ctx, AUTO/MANUAL); // 自动/手动曝光
rk_aiq_uapi2_setExpManualTime(ctx, us);    // 手动曝光时间
rk_aiq_uapi2_setExpManualGain(ctx, gain);  // 手动增益
rk_aiq_uapi2_setFrameRate(ctx, fps);       // 帧率

/* AWB 控制 */
rk_aiq_uapi2_setWBMode(ctx, AUTO/MANUAL);  // 自动/手动白平衡
rk_aiq_uapi2_setMWBGain(ctx, r, g, b);    // 手动白平衡增益

/* 降噪 */
rk_aiq_user_api2_aibnr_SetAttrib(ctx, attr); // AI Bayer 降噪
rk_aiq_user_api2_btnr_SetAttrib(ctx, attr);  // Bayer 时域降噪

/* HDR/去雾 */
rk_aiq_uapi2_setDehazeEnable(ctx, on);     // 去雾
rk_aiq_uapi2_setMDehazeStrth(ctx, level);  // 去雾强度
rk_aiq_uapi2_setDarkAreaBoostStrth(ctx, v);// 暗部增强

/* 色彩 */
rk_aiq_uapi2_setColorMode(ctx, mode);      // 色彩模式
rk_aiq_uapi2_setColorSpace(ctx, cs);       // 色彩空间
```

---

## 五、实验 1：启动 RKAIQ 3A Server

### 5.1 实验目标

在板端启动 RKAIQ 3A server，验证 3A 算法运行。

### 5.2 操作步骤

```bash
# 板端:

# 1. 检查 IQ 文件
ls /etc/iqfiles/
# 预期: isp35/ 目录 + JSON 文件

# 2. 检查 3A server
ls /usr/bin/rkaiq_3A_server
# 或检查 init 脚本
cat /etc/init.d/S40rkaiq_3A

# 3. 启动 3A server (如果未自动启动)
sudo /usr/bin/rkaiq_3A_server &

# 4. 检查运行状态
ps | grep rkaiq
# 预期: rkaiq_3A_server 进程存在

# 5. 检查 ISP stats/params 节点是否活跃
# (需要 rkipc 或相机应用正在运行)
v4l2-ctl -d /dev/video12 --stream-mmap --stream-count=5
# 预期: stats 节点能采集到数据
```

---

## 六、实验 2：调整 AE 参数观察效果

### 6.1 实验目标

通过 rkipc 或自定义程序调整 AE 参数，观察画面亮度变化。

### 6.2 操作步骤

```bash
# 方法 1: 通过 rkipc 配置文件
# 编辑 /etc/rkipc.ini
# [isp]
# isp.ae.exp_mode = auto          # 自动曝光
# isp.ae.exp_time = 10000         # 手动曝光时间 (us)
# isp.ae.exp_gain = 4.0           # 手动增益
# 修改后重启 rkipc

# 方法 2: 通过 v4l2-ctl 直接控制 sensor subdev
# 设置手动曝光
v4l2-ctl -d /dev/v4l-subdev0 \
    --set-ctrl=exposure=5000     # 曝光时间 5000us
v4l2-ctl -d /dev/v4l-subdev0 \
    --set-ctrl=gain=200          # 增益 2.0x

# 方法 3: 通过 RKAIQ API (需要写程序)
# rk_aiq_uapi2_setExpMode(ctx, MANUAL);
# rk_aiq_uapi2_setExpManualTime(ctx, 5000);
# rk_aiq_uapi2_setExpManualGain(ctx, 200);
```

### 6.3 验证

```bash
# 抓取不同曝光参数下的帧, 对比亮度
# 自动曝光
v4l2-ctl -d /dev/video10 --stream-mmap --stream-count=1 --stream-to=/tmp/auto.nv12

# 手动短曝光 (暗)
v4l2-ctl -d /dev/v4l-subdev0 --set-ctrl=exposure=1000
v4l2-ctl -d /dev/video10 --stream-mmap --stream-count=1 --stream-to=/tmp/dark.nv12

# 手动长曝光 (亮)
v4l2-ctl -d /dev/v4l-subdev0 --set-ctrl=exposure=30000
v4l2-ctl -d /dev/video10 --stream-mmap --stream-count=1 --stream-to=/tmp/bright.nv12

# 对比三帧的亮度 (用 Python 计算平均 Y)
python3 -c "
import struct
for name in ['auto', 'dark', 'bright']:
    with open(f'/tmp/{name}.nv12', 'rb') as f:
        data = f.read(1920*1080)  # Y 平面
        avg = sum(data) / len(data)
        print(f'{name}: avg_Y = {avg:.1f}')
"
```

---

## 七、实验 3：分析 ISP Stats 数据

### 7.1 实验目标

读取 ISP stats 节点的数据，理解 3A 算法的输入。

### 7.2 操作步骤

```bash
# ISP stats 节点输出的是 C 结构体 (不是图像)
# 结构定义在: kernel-6.1/include/uapi/linux/rk-isp3x-config.h
# struct rkisp3x_isp_stat

# 读取 stats
v4l2-ctl -d /dev/video12 --stream-mmap --stream-count=1 --stream-to=/tmp/stats.bin

# 分析 (需要解析 C 结构体)
# 关键字段:
# - ae.meas: AE 统计 (直方图, 各区域亮度)
# - awb.meas: AWB 统计 (R/G/B 通道均值)
# - af.meas: AF 统计 (对焦值 FV)
# - hist.meas: 直方图
```

### 7.3 stats 数据结构

```c
/* kernel-6.1/include/uapi/linux/rk-isp3x-config.h */
struct rkisp3x_isp_stat {
    __u32 frame_id;
    struct rkisp3x_isp_ae_stat ae;    // AE 统计
    struct rkisp3x_isp_awb_stat awb;  // AWB 统计
    struct rkisp3x_isp_af_stat af;    // AF 统计
    struct rkisp3x_isp_hist_stat hist;// 直方图
    ...
};

/* AE 统计: 分区域亮度 (如 15×15 网格) */
/* AWB 统计: R/G/B 通道在白点的统计 */
/* AF 统计: 对焦值 (对比度) 随位置变化 */
```

---

## 八、实验 4：IQ 文件调参

### 8.1 实验目标

修改 IQ JSON 文件中的降噪/锐化参数，观察画质变化。

### 8.2 操作步骤

```bash
# 1. 备份原始 IQ 文件
cp /etc/iqfiles/isp35/common/*.json /tmp/iq_backup/

# 2. 查看可调参数
ls /etc/iqfiles/isp35/common/
# 预期: AINR.json, ANR.json, Sharp.json, Gamma.json, CCM.json...

# 3. 调整降噪强度 (以 AINR.json 为例)
# 找到 strength/gain 等参数, 适当增大或减小
# 注意: JSON 格式必须正确, 否则 3A server 加载失败

# 4. 重启 3A server + rkipc
sudo killall rkaiq_3A_server rkipc
sudo /usr/bin/rkaiq_3A_server &
sleep 1
sudo /usr/bin/rkipc &

# 5. 对比画质
# 降噪强度低: 噪点明显但细节保留
# 降噪强度高: 画面干净但涂抹感强
# 锐化强度低: 画面柔和
# 锐化强度高: 边缘锐利但可能有锯齿

# 6. 恢复
cp /tmp/iq_backup/*.json /etc/iqfiles/isp35/common/
```

> **大疆 IQ 调校**：大疆有专门的 IQ 团队，对每个 Sensor + 镜头组合做数周的调校。参数间相互影响（降噪强→锐化需求变），需要反复实测。

---

## 九、实验 5：多帧 HDR 对比

### 9.1 实验目标

理解 HDR (高动态范围) 的原理和效果。

### 9.2 HDR 原理

```
短曝光帧: 天空细节好, 地面全黑
长曝光帧: 地面细节好, 天空过曝

HDR 合并:
  取短帧的亮区 + 长帧的暗区 → 合成一张高动态范围图

ISP V35 支持:
  - 硬件 HDR (Sensor 输出长短交替帧)
  - DRC (动态范围压缩): 单帧提亮暗部
  - Amerge: 多帧 HDR 合并
```

### 9.3 操作步骤

```bash
# 通过 RKAIQ 启用 HDR
# rk_aiq_uapi2_setAeExpSepStep(ctx, 2);  // 曝光分离步进
# rk_aiq_uapi2_setHDR(ctx, HDR_ENABLE);

# 对比:
# 1. HDR 关闭: 拍逆光场景, 天空过曝/地面死黑
# 2. HDR 开启: 拍同场景, 天空有细节/地面也可见

# 通过 DRC 也可单帧提亮暗部
# rk_aiq_uapi2_setDarkAreaBoostStrth(ctx, 50);
```

---

## 十、思考题

1. AE 算法在"亮度 vs 模糊 vs 噪点"间权衡。如果用户要求在 30fps 下拍运动物体（不能有运动模糊），AE 应该优先调整哪个参数？会有什么副作用？

2. AWB 的 Gray World 算法假设画面平均颜色是灰色。大疆航拍场景中天空占比常超过 50%，这会对 Gray World 算法造成什么问题？如何解决？

3. ISP stats 和 params 节点是每帧交互的。如果 3A 算法处理延迟超过 1 帧时间（33ms@30fps），会发生什么？如何保证 3A 的实时性？

4. RKAIQ 有 71 个算法模块，模块间有依赖关系（如降噪影响锐化参数）。如果你是 IQ 调校工程师，你会按什么顺序调校？为什么？

5. 大疆 Mavic 3 的哈苏相机支持 14 档动态范围。从 ISP 角度看，14 档动态范围需要 Sensor 输出多少 bit 的 Raw 数据？ISP 的 HDR 合并需要几帧？

---

## 十一、踩坑记录

| 日期 | 问题 | 原因 | 解决方案 |
|------|------|------|----------|
| | 3A server 启动失败 | IQ 文件缺失或格式错 | 检查 /etc/iqfiles/isp35/ JSON 文件 |
| | 画面偏暗/偏亮 | AE 未收敛或手动曝光设置不当 | 检查 ExpMode 和 ExpTime/Gain |
| | 画面偏色 | AWB 未收敛或 IQ 文件 CCM 不对 | 检查 WBMode, 对比 IQ 文件 |
| | 画面噪点多 | 降噪参数太低或增益太高 | 调整 AINR.json, 降低增益 |
| | 画面涂抹感 | 降噪太强 | 降低 NR strength |
| | 闪烁 | 灯光 50/60Hz 干扰 | 启用 AFD 闪烁检测 |
| | IQ 文件修改后无效 | 3A server 未重启 | killall + 重启 rkaiq_3A_server |

---

## 十二、下阶段预告

阶段十：**MPP 硬件编解码管线**
- MPP 架构：MPI 接口 + mpp_service + VEPU511
- Rockit 框架：RK_MPI_VI → VENC → VO 管线
- 编码流程 + H.265 参数调优
- 相机录像管线：ISP → VPSS → MPP → H.265 文件

---

## 相关笔记

- [[MOC-嵌入式Linux]] — 学习路线总览
- [[bsp-csi-sensor-driver]] — 阶段七：Sensor 驱动 (3A 控制 Sensor)
- [[bsp-v4l2-isp-pipeline]] — 阶段八：ISP Pipeline (stats/params 节点)
- [[mpp-hardware-codec]] — 附录C：MPP 编码实验 (MPP 基础)
