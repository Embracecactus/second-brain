---
tags:
  - embedded-linux
  - bsp
  - bootloader
  - u-boot
  - boot-time
  - optimization
  - kernel
  - initcall
  - rockchip
  - rv1126b
category: embedded-linux
created: 2026-06-23
updated: 2026-06-23
status: active
soc: Rockchip RV1126B
kernel: Linux 6.1.141
parent: bsp-boot-flow
---

# U-Boot 启动速度优化 — 全链路分析与优化

> **前置笔记**：[[bsp-boot-flow]] — Boot Chain 全景 & 启动实验
>
> **前置笔记**：[[bsp-spl-fit]] — SPL FIT 解析耗时分析
>
> **前置笔记**：[[bsp-uboot-adaptation]] — 各阶段串口日志特征与计时

## 本文核心数据

| 指标 | 值 |
|------|------|
| Kernel Image (uncompressed) | **44 MB** |
| boot.img (FIT with kernel) | **46 MB** |
| U-Boot SPL | **251 KB** |
| U-Boot proper | **1.1 MB + 6.4 KB DTB** |
| U-Boot CONFIG_BOOTDELAY | **0** (已禁用) |
| U-Boot 解压算法 | **LZ4** (最快) |
| 内核抢占模型 | **PREEMPT_VOLUNTARY** |
| 目标总启动时间 | **< 2s** (运动相机) |
| 极限优化目标 | **< 1s** (DJI 竞品级) |

---

## 一、全链路时间模型

### 1.1 RV1126B 启动时间分解

上电到 login 的完整时间 = 各阶段串行之和：

```
t_total = t_bootrom + t_ddr + t_spl + t_uboot + t_kernel + t_rootfs

各阶段估算 (基于 SDK 数据和典型经验值):
t_bootrom  ≈  10-20ms   (BootROM 执行, 闭源, 不可优化)
t_ddr      ≈  80-120ms  (DDR 训练 1332MHz, 闭源, 不可优化)
t_spl      ≈  50-80ms   (SPL 加载 FIT + 验签 + 跳转 ATF)
t_uboot    ≈  200-400ms (U-Boot proper 初始化 + bootm)
t_kernel   ≈  500-800ms (Kernel probe + initcalls + 挂载 rootfs)
t_rootfs   ≈  500-1500ms(userspace 服务, systemd/BusyBox)
──────────────────────────────────────────
t_total    ≈  1.5-3.0s  (当前估算)
```

### 1.2 当前 sportcam 已知优化状态

| 优化点 | sportcam 状态 | 影响阶段 | 说明 |
|-------|-------------|---------|------|
| U-Boot 解压 | ✅ LZ4 | uboot→kernel | 最快解压算法 |
| autoboot 延迟 | ✅ BOOTDELAY=0 | uboot | 无等待时间 |
| 内核 defconfig | ❌ GZIP 默认 | kernel | Image 44MB, gzip ~11MB |
| SPL FIT 验签 | ❌ 未启用 | spl | 启用后 +~50ms |
| 内核压缩 | ❌ GZIP | kernel | LZ4 解压快 2x 但体积大 |
| 内核 initcall | ❌ 未分析 | kernel | 需测量后裁剪 |
| rootfs type | Buildroot ext4 | rootfs | 可换 squashfs/initramfs |
| systemd 并行 | N/A (BusyBox) | rootfs | Buildroot 串行执行 |

---

## 二、精确测量方法

### 2.1 硬件计时: U-Boot bootstage

`bootstage` 是 U-Boot 内置的启动计时框架，使用 SoC 的硬件定时器 (精度 μs 级):

```kconfig
# 在 defconfig 中启用
CONFIG_BOOTSTAGE=y                # 启用 bootstage
CONFIG_BOOTSTAGE_REPORT=y         # 启动后打印报告
CONFIG_BOOTSTAGE_USER_COUNT=20   # 自定义标记点数量
```

启用后，U-Boot 命令行执行:

```bash
=> bootstage report
```

预期输出:

```
Timer summary in microseconds (16 entries):
       Mark    Elapsed  Stage Name
          0          0  reset
     52,345     52,345  board_init_f
    123,456     71,111  board_init_r
    201,234     77,778  id=60
    234,567     33,333  main_loop
    345,678    111,111  bootm_start
    456,789    111,111  bootm_load_os
    567,890    111,101  start_kernel
─────────────────────────────────────────
Total: 567,890 us (0.57s) before kernel
```

### 2.2 内核 initcall 计时

```bash
# bootargs 添加:
# initcall_debug loglevel=8

# 启动后提取 TOP 20 耗时 initcall
dmesg | grep "initcall.*returned.*after" | \
    sed 's/.*after \([0-9]*\) usecs.*/\1/;s/.*initcall/deleted/' | \
    awk '{print $1, $NF}' | sort -rn | head -20

# 分析脚本
python3 -c "
import re
calls = []
with open('/tmp/dmesg.log') as f:
    for line in f:
        m = re.search(r'initcall (.+?)\+.*returned (\d+) after (\d+) usecs', line)
        if m:
            calls.append((m.group(1), int(m.group(3))))
calls.sort(key=lambda x: -x[1])
print(f'{\"initcall\":<50} {\"time (ms)\":>10}')
print('-' * 62)
for name, us in calls[:20]:
    print(f'{name:<50} {us/1000:>10.2f}')
total = sum(us for _, us in calls) / 1000
print(f'{\"TOTAL initcall\":<50} {total:>10.2f}')
"
```

### 2.3 内核启动时间 line-by-line 分析

```bash
#!/bin/bash
# 全链路时间线分析脚本

LOG=$1

echo "=== 全链路时间线 ==="
echo ""

# BootROM/DDR (无日志)
echo "上电:          0 ms"

# SPL 开始
spl_start=$(grep -m1 'U-Boot SPL board init' "$LOG" | awk -F'[\\[\\]]' '{print $2}')
echo "SPL 开始:      ${spl_start:-???} ms"

# FIT 加载
fit_start=$(grep -m1 'Trying fit image' "$LOG" | awk -F'[\\[\\]]' '{print $2}')
echo "FIT 加载:      ${fit_start:-???} ms"

# U-Boot 版本
uboot_time=$(grep -m1 '^U-Boot 20' "$LOG" | awk -F'[\\[\\]]' '{print $2}')
echo "U-Boot:        ${uboot_time:-???} ms"

# bootm 加载 kernel
kernel_load=$(grep -m1 '## Loading kernel from FIT' "$LOG" | awk -F'[\\[\\]]' '{print $2}')
echo "bootm 加载:    ${kernel_load:-???} ms"

# Starting kernel (U-Boot 最后一条)
kernel_start_uboot=$(grep -m1 'Starting kernel' "$LOG" | awk -F'[\\[\\]]' '{print $2}')
echo "→ Kernel:      ${kernel_start_uboot:-???} ms"

# Kernel 第一条 (0.000000)
echo ""
echo "--- Kernel 内部 ---"

# Freeing unused memory (Kernel 完成)
kernel_done=$(grep -m1 'Freeing unused kernel' "$LOG" | awk '{print $1}' | tr -d '[]')
echo "Kernel 完成:   ${kernel_done}s"

# init 启动
init_start=$(grep -m1 'Run /sbin/init' "$LOG" | awk '{print $1}' | tr -d '[]')
echo "Init 启动:     ${init_start}s"

# login
login_time=$(grep -m1 -E 'login:|Welcome to' "$LOG" | awk '{print $1}' | tr -d '[]')
echo "登录就绪:     ${login_time}s"

echo ""
echo "=== 耗时汇总 ==="
echo "SPL → U-Boot:        $(echo ${uboot_time} - ${spl_start} | bc 2>/dev/null) ms"
echo "U-Boot → Kernel:     $(echo ${kernel_start_uboot} - ${uboot_time} | bc 2>/dev/null) ms"
echo "Kernel total:        ${kernel_done}s"
echo "Userspace:           $(echo ${login_time} - ${init_start} | bc 2>/dev/null) s"
```

### 2.4 Ftrace 函数级启动追踪

```bash
# bootargs 添加:
# trace_event=initcall:* trace_buf_size=64M ftrace=function_graph
# trace_options=funcgraph-abstime funcgraph-proc

# 启动后查看:
cat /sys/kernel/tracing/trace > /tmp/boot_ftrace.log

# 提取所有 initcall 调用链
grep "initcall" /tmp/boot_ftrace.log | head -100

# 分析函数调用耗时
cat /tmp/boot_ftrace.log | \
    grep -oP '\)\s*\+\s*\K[0-9.]+(?=\s+usec)' | \
    awk '{sum+=$1; n++; if($1>max) max=$1} END \
    {print "Total: " sum/1000 " ms, Avg: " sum/n/1000 " ms, Max: " max/1000 " ms"}'
```

---

## 三、U-Boot 阶段优化

### 3.1 SPL 阶段

SPL 阶段可优化的部分：

```kconfig
# 减少 SPL 设备扫描
# 只保留启动必需的设备:
CONFIG_SPL_DM=y
# CONFIG_SPL_DM_KEYBOARD is not set    # 不需要键盘
# CONFIG_SPL_DM_VIDEO is not set        # 不需要显示
CONFIG_SPL_DM_MMC=y                     # 需要 eMMC
CONFIG_SPL_DM_SERIAL=y                  # 需要串口
CONFIG_SPL_DM_I2C=y                     # 需要 PMIC
CONFIG_SPL_DM_RTC=y                     # 需要 RTC (可选)
```

`u-boot,dm-spl` DTS 标记控制 SPL 扫描的设备：

```dts
// arch/arm/dts/rv1126b-u-boot.dtsi
// 只在 SPL 阶段标记必要的设备
&cru      { u-boot,dm-spl; };   // 时钟
&grf      { u-boot,dm-spl; };   // 寄存器
&uart0    { u-boot,dm-spl; };   // 串口
&emmc     { u-boot,dm-spl; };   // eMMC
&pinctrl  { u-boot,dm-spl; };   // 引脚

// 以下不要标记 u-boot,dm-spl，SPL 阶段不需要:
// &usbdrd      { /* SPL 不需要 USB */ };
// &i2c1        { /* SPL 不需要其他 I2C */ };
// &sdmmc       { /* SPL 不需要 SD 卡 */ };
```

### 3.2 U-Boot proper 阶段

**环境变量加载优化:**

```kconfig
# 默认: 从 MMC 读取完整环境变量 (~8KB)
# 如果环境变量不经常修改，可预编译到 U-Boot 二进制中
CONFIG_ENV_IS_NOWHERE=y              # 无环境变量存储 (最快)
# 或
CONFIG_ENV_IS_IN_MMC=y               # MMC 环境 (默认)
CONFIG_ENV_SIZE=0x2000               # 8KB, 减少读取量
# 禁用环境变量冗余备份:
# CONFIG_ENV_IS_IN_MMC_REDUNDANT is not set
```

**禁用不需要的驱动:**

```kconfig
# 相机不需要的功能
# CONFIG_USB is not set                # 如果启动阶段不需要 USB
# CONFIG_DM_USB is not set             # 禁用 USB DM
# CONFIG_USB_STORAGE is not set        # 禁用 U 盘
# CONFIG_VIDEO is not set              # 禁用显示输出 (节省 ~30ms)
# CONFIG_DM_VIDEO is not set
# CONFIG_CMD_NET is not set            # 禁用网络命令
# CONFIG_CMD_PING is not set
# CONFIG_CMD_DHCP is not set
```

### 3.3 bootm 加速

```kconfig
# 跳过不必要的解压
CONFIG_BOOTM_OPTIMIZE_SIZE=n          # 关闭 bootm 大小优化

# 减少 bootm 过程中的 log 输出
# CONFIG_LOG is not set

# 如果不需要 FIT 验签 (非安全产品):
# CONFIG_FIT_SIGNATURE is not set      # 跳过签名验证, 节省 ~50ms
```

### 3.4 实测优化预期

| 优化 | 预期收益 | 风险 |
|------|---------|------|
| 减少 SPL DM 设备 | -10~20ms | 启动设备不可用 |
| 禁用 USB | -20~50ms | USB 启动后不可用 |
| 禁用 NET | -10~30ms | 网络不可用 |
| 跳过签名 | -50~100ms | 无安全保护 |
| 内置环境变量 | -5~10ms | 修改 env 需重编译 |
| **U-Boot 总优化** | **-100~200ms** | |

---

## 四、Kernel 阶段优化

### 4.1 内核压缩算法决策

```bash
# 当前: 默认 GZIP
# CONFIG_KERNEL_GZIP=y        # Image 44MB → ~11MB, 解压慢 ~500ms
# 选项:
CONFIG_KERNEL_LZ4=y            # Image 44MB → ~17MB, 解压快 ~200ms
# CONFIG_KERNEL_LZO=y          # Image 44MB → ~14MB, 解压中等 ~350ms
# CONFIG_KERNEL_XZ=y          # Image 44MB → ~8MB,  解压慢 ~800ms
```

| 算法 | 压缩后大小 | 解压速度 | 总时间 (包含 IO + 解压) | 备注 |
|------|-----------|---------|----------------------|------|
| **GZIP** | ~11 MB | 基准 1x | 基准 | 当前默认 |
| **LZ4** | ~17 MB | **~2x** | **-200~300ms** | sportcam U-Boot 已支持 |
| **LZO** | ~14 MB | ~1.5x | -100~200ms | 平衡 |
| **XZ** | ~8 MB | ~0.5x | +300~500ms | 不适合启动 |

> **选择建议**: LZ4 解压最快，虽然镜像大 50% 但从 eMMC 读取更快 (eMMC HS400 约 200MB/s，额外 6MB 只需 ~30ms)。**GZIP → LZ4 净收益约 200ms。**

### 4.2 驱动 probe 优化

**方案 A: 延迟非关键驱动为模块 (Deferred Probe)**

```bash
# 从 dmesg initcall_debug 找出最耗时的驱动
dmesg | grep "initcall" | sort -rn -t' ' -k5 | head -10
# 预期 TOP 耗时:
# 50-100ms:  rkisp (ISP 相机)
# 30-50ms:   rknpu (NPU)
# 20-30ms:   mpp_service
# 10-20ms:   dwc3 (USB 3.0)
# 10-20ms:   rk8xx (PMIC)

# 将这些驱动改为模块 (.ko):
# CONFIG_VIDEO_ROCKCHIP_ISP=m       # ISP 驱动
# CONFIG_RKNPU=m                     # NPU 驱动
# CONFIG_ROCKCHIP_MPP_SERVICE=m      # MPP 编解码
```

| 驱动 | 改为模块收益 | 影响 | 按需加载方式 |
|------|------------|------|------------|
| ISP (rkisp) | -50~100ms | 启动后不能立即录像 | `modprobe rockchip-isp` |
| NPU (rknpu) | -30~50ms | AI 功能不可用 | `modprobe rknpu` |
| MPP (mpp_service) | -20~30ms | 无法编解码 | `modprobe mpp_service` |
| USB DWC3 | -10~20ms | 无 USB 功能 | `modprobe dwc3` |

**方案 B: 异步 probe**

```c
// 驱动代码中, 将 probe 改为异步:
// 修改前: static int rkisp_probe(struct platform_device *pdev)
// 修改后:
static int rkisp_probe(struct platform_device *pdev)
{
    // ... 快速返回, 实际初始化延后
    return 0;
}

static int __init rkisp_init(void)
{
    return platform_driver_register(&rkisp_driver);
}
// 或使用 deferred probe 机制:
// echo -1 > /sys/bus/platform/drivers/rkisp/bind
```

### 4.3 内核配置裁剪

```kconfig
# 运动相机不需要的功能 (可节省 50-100ms)
# CONFIG_NET_9P is not set          # 9P 虚拟文件系统
# CONFIG_NFS_FS is not set          # NFS 挂载
# CONFIG_CIFS is not set            # CIFS/SMB
# CONFIG_IPV6 is not set            # IPv6 (如果不需要)
# CONFIG_WIRELESS is not set        # 无线 (如果不需要)
# CONFIG_BT is not set              # 蓝牙
# CONFIG_SOUND is not set           # ALSA (如果不需要)
# CONFIG_DRM is not set             # DRM 显示
# CONFIG_V4L_PLATFORM_DRIVERS is not set  # 平台 V4L2 驱动

# 减少 probe 的设备数量
# 在 DTS 中禁用不需要的节点:
// rv1126b-sportcam.dts
// &sdmmc { status = "disabled"; };      // 不需要 SD 卡
// &wdt  { status = "disabled"; };       // 不需要看门狗
// &can0 { status = "disabled"; };       // 不需要 CAN
// &i2c3 { status = "disabled"; };       // 不需要的 I2C 总线
```

### 4.4 内核启动参数优化

```bash
# 当前 bootargs (见 bsp-boot-flow.md):
# net.ifnames=0 earlycon=uart8250,mmio32,0x20810000
# console=ttyFIQ0 rw root=PARTUUID=614e0000-0000
# rootfstype=ext4 rootwait snd_soc_core.prealloc_buffer_size_kbytes=16
# coherent_pool=32K

# 优化后:
# rootfstype=ext4 rootwait       ← 保留
# no_console_suspend             ← 调试时添加
# quiet                          ← # 减少日志输出 (节省 ~10ms IO)
# loglevel=3                     ← # 仅显示 KERN_ERR 及以上
# udev.children-max=1            ← # 限制 udev 并发, 减少启动压力
# initcall_blacklist=clk_disable_unused  ← 跳过不需要的 initcall
```

---

## 五、Rootfs 阶段优化

### 5.1 Buildroot BusyBox init

当前 sportcam SDK 默认使用 Buildroot:

```bash
# rcS 遍历 S?? 脚本, 串行执行
# 总耗时取决于最慢的服务链

# 优化手段:
# 1. 合并多个脚本为一个 (减少 fork 开销)
# 2. 并行启动独立服务 (在 rcS 中使用 & 后台执行)
cat /etc/init.d/rcS
#!/bin/sh
# 串行启动依赖链
/etc/init.d/S01syslogd start
/etc/init.d/S10udev start       # udev 完成后才能创建设备节点
# 并行启动独立服务
/etc/init.d/S30dbus start &
/etc/init.d/S40network start &
/etc/init.d/S50sshd start &
wait                            # 等待后台任务完成
/etc/init.d/S99camera start     # 依赖网络? 等待 wait 后
```

### 5.2 Initramfs 方案

对于快速启动, Initramfs (内存文件系统) 可大幅减少 rootfs 加载时间:

```bash
# 在 kernel defconfig 中启用:
CONFIG_BLK_DEV_INITRD=y
CONFIG_INITRAMFS_SOURCE="rootfs.cpio.gz"

# 构建最小 initramfs (仅包含 busybox + 必要的 .ko)
mkdir -p initramfs_root/{bin,dev,etc,lib,proc,sys}
cp /path/to/busybox initramfs_root/bin/
# ... 添加必要的驱动模块
cd initramfs_root && find . | cpio -H newc -o | gzip > ../initramfs.cpio.gz

# 嵌入内核 Image
# 在 kernel defconfig 中:
CONFIG_INITRAMFS_SOURCE="/path/to/initramfs.cpio.gz"

# 预期收益: rootfs 加载时间从 ~500ms → ~100ms
# 缺点是 rootfs 大小受限于内核编译时的嵌入大小
```

### 5.3 BoardRoot (Ubuntu) systemd 优化

如果使用 BoardRoot Ubuntu rootfs:

```bash
# 1. 分析 systemd 启动耗时
systemd-analyze blame

# 2. 禁用不需要的服务
systemctl disable bluetooth.service
systemctl disable NetworkManager-wait-online.service  # 常见瓶颈
systemctl disable connman.service
systemctl mask systemd-networkd-wait-online.service

# 3. 缩短服务超时时间
# /etc/systemd/system.conf:
DefaultTimeoutStartSec=5s
DefaultTimeoutStopSec=5s

# 4. 并行化服务依赖
# /etc/systemd/system/multi-user.target.wants/
# 移除依赖冲突的 .wants 链接

# 5. 使用 systemd-analyze plot 可视化瓶颈
systemd-analyze plot > boot.svg
```

### 5.4 极端优化: squashfs + dm-verity

```bash
# squashfs 只读文件系统 (不需要 fsck, 快速挂载)
# + dm-verity 完整性保护

# 构建命令:
mksquashfs rootfs/ rootfs.squashfs -comp lz4 -b 256K
# LZ4 压缩 + 256KB 块 → 最快挂载速度

# 对比:
# ext4: 挂载需 journal replay (~100-300ms)
# squashfs: 挂载 < 10ms
# initramfs: 内核已经解压到内存, 瞬时
```

---

## 六、Compression 详测

### 6.1 U-Boot FIT 压缩算法

U-Boot 中 FIT 镜像各段的压缩方式在 ITS 中定义:

```dts
// boot.its
images {
    kernel {
        data = /incbin/("@KERNEL_IMG@");
        compression = "none";          // ★ 内核自身压缩, 不是 FIT 处理
        // 可选: "gzip", "lz4", "lzo", "none"
    };
};
```

Rockchip 的打包脚本使用 `mkimage`:

```bash
# mkimage 支持:
# -c none    不压缩
# -c gzip    gzip 压缩
# -c lzma    lzma 压缩
# -c lz4     lz4 压缩

# 当前 sportcam 使用 "none" + 内核自压缩
# 如果 FIT 层也处理压缩, 可双重压缩但收益不大
```

### 6.2 内核自解压对比

```bash
# 编译内核时选择压缩算法:
# kernel-6.1/arch/arm64/configs/rv1126b_sportcam_defconfig

CONFIG_KERNEL_GZIP=y    # 约 11MB, 解压 ~500ms
# CONFIG_KERNEL_LZ4=y   # 约 17MB, 解压 ~200ms, 推荐
# CONFIG_KERNEL_LZO=y   # 约 14MB, 解压 ~350ms

# 实测: Image 44MB
# GZIP:  44MB → ~11MB (compressed IO: ~55ms @200MB/s + 解压 ~500ms)
# LZ4:   44MB → ~17MB (compressed IO: ~85ms @200MB/s + 解压 ~200ms)
#        LZ4 净收益 = (55+500) - (85+200) = ~270ms
```

---

## 七、RV1126B 具体优化清单

### 7.1 按优先级排序

| 优先级 | 优化手段 | 阶段 | 预期收益 | 实现难度 | 风险 |
|--------|---------|------|---------|---------|------|
| 🔴 P0 | kernel GZIP → LZ4 | kernel | -270ms | 低 | 无 |
| 🔴 P0 | bootargs `quiet loglevel=3` | kernel | -10ms | 低 | 调试困难 |
| 🔴 P0 | 禁用不需要的 initcall | kernel | -50~200ms | 中 | 功能缺失 |
| 🟡 P1 | 延迟 ISP/NPU 为模块 | kernel | -50~150ms | 低 | 启动后功能需手动加载 |
| 🟡 P1 | 裁剪 kernel defconfig | kernel | -50~100ms | 低 | 功能可能缺失 |
| 🟡 P1 | 禁用 U-Boot USB/NET | uboot | -30~80ms | 低 | 启动后不可用 |
| 🟡 P1 | U-Boot bootstage 测量 | uboot | 0 (测量) | 低 | 无 |
| 🟢 P2 | 精简 rootfs / rcS 并行 | rootfs | -100~500ms | 中 | 服务依赖 |
| 🟢 P2 | systemd 优化 (如适用) | rootfs | -200~500ms | 中 | 依赖冲突 |
| 🟢 P2 | swap to squashfs | rootfs | -50~200ms | 中 | 只读 |
| ⚪ P3 | Initramfs | kernel | -100~300ms | 高 | rootfs 嵌入内核 |
| ⚪ P3 | SPL DM 裁剪 | spl | -10~20ms | 低 | 低收益 |

### 7.2 sportcam 优化目标

```
当前估算:   上电 → login = ~2.0s
P0 优化后:  ~1.7s   (kernel 压缩 + 日志裁剪)
P1 优化后:  ~1.5s   (模块延迟 + 裁剪)
P2 优化后:  ~1.2s   (rootfs 优化)
P3 优化后:  ~1.0s   (极限优化)

DJI 竞品参考: 从按下电源到图传画面 ≈ 1-2s
运动相机参考: 从按下电源到开始录像 ≈ 1.5-3s
```

---

## 八、优化实验

### 8.1 实验 1: 基准测量

```bash
# 1. 启用 bootstage
cd $SDK_ROOT/u-boot
cat >> configs/rv1126b_sportcam_defconfig << 'EOF'
CONFIG_BOOTSTAGE=y
CONFIG_BOOTSTAGE_REPORT=y
CONFIG_BOOTSTAGE_USER_COUNT=20
EOF

# 2. 添加 initcall_debug
# 修改 DTS chosen.bootargs, 追加: initcall_debug loglevel=8

# 3. 编译部署
cd $SDK_ROOT
./build.sh loader
./build.sh kernel

# 4. 板端重启, 抓日志
# 预期输出:
#   U-Boot: bootstage report
#   Kernel: initcall debug 每行带耗时
```

### 8.2 实验 2: 内核 LZ4 压缩

```bash
# 1. 修改 kernel defconfig
cd $SDK_ROOT
cat >> kernel-6.1/arch/arm64/configs/rv1126b_sportcam_defconfig << 'EOF'
CONFIG_KERNEL_GZIP=n
CONFIG_KERNEL_LZ4=y
EOF

# 2. 重新编译内核
./build.sh kernel

# 3. 对比 boot.img 大小
ls -lh kernel-6.1/arch/arm64/boot/Image* kernel-6.1/boot.img

# 4. 部署并测量启动时间
sudo dd if=kernel-6.1/boot.img of=/dev/mmcblk0p3 bs=512
# 对比 Freeing unused kernel memory 的时间戳
```

### 8.3 实验 3: 驱动模块化

```bash
# 1. 在 kernel defconfig 中将 ISP 改为模块
cd $SDK_ROOT
echo "CONFIG_VIDEO_ROCKCHIP_ISP=m" >> kernel-6.1/arch/arm64/configs/rv1126b_sportcam_defconfig

# 2. 重新编译
./build.sh kernel
./build.sh kernel-modules     # 编译模块

# 3. 部署模块到 board
scp kernel-6.1/drivers/media/platform/rockchip/isp/rockchip-isp.ko rooter@192.168.1.109:/lib/modules/

# 4. 板端: 启动后手动加载
insmod /lib/modules/rockchip-isp.ko

# 5. 对比 dmesg 中 ISP 相关时间戳
# 优化前: ISP probe 在内核启动期间
# 优化后: ISP probe 在 userspace modprobe 后
```

### 8.4 实验 4: systemd 优化

```bash
# 仅用于 BoardRoot Ubuntu rootfs

# 板端:
# 1. 查看启动瓶颈
systemd-analyze blame

# 2. 禁用等待网络
systemctl mask NetworkManager-wait-online.service

# 3. 禁用蓝牙
systemctl disable bluetooth.service

# 4. 再次测量
systemd-analyze
# 预期: init 时间减少 200-500ms
```

### 8.5 实验 5: 全链路优化组合

```bash
# 综合优化步骤:

# Step 1: Kernel LZ4 压缩
# Step 2: 禁用不需要的内核驱动 (CAN, WIRELESS, BT 等)
# Step 3: 延迟 ISP/NPU 为模块
# Step 4: bootargs 添加 quiet loglevel=3
# Step 5: U-Boot 禁用 USB/NET DM
# Step 6: U-Boot BOOTDELAY=0 (已设置)
# Step 7: 精简 rootfs rcS

# 优化前后对比:
# 优化前: ~2.0s
# 优化后: ~1.2-1.5s (预期)

# 输出优化报告:
echo "=== 优化前后对比 ==="
echo "阶段               优化前     优化后"
echo "SPL                ~60ms     ~60ms   (不变)"
echo "U-Boot             ~300ms    ~200ms  (裁剪 DM)"
echo "Kernel 解压        ~550ms    ~280ms  (GZIP→LZ4)"
echo "Kernel initcall    ~300ms    ~150ms  (模块化)"
echo "Rootfs             ~500ms    ~300ms  (rcS 并行)"
echo "-------------------------------"
echo "总计               ~1710ms   ~990ms"
```

---

## 九、思考题

1. **压缩权衡**：LZ4 比 GZIP 解压快 2x 但镜像大 50%。从 eMMC (HS400, 200MB/s) 读取时哪种算法总时间更短？从 SPI NAND (~40MB/s) 呢？

2. **bootstage vs initcall_debug**：U-Boot 的 bootstage 是否直接测量了 SPI flash 读取 FIT 的时间？initcall_debug 测量的时间包含 IO 等待吗？

3. **initramfs vs ext4**：initramfs 从内核嵌入到解压到内存是内核初始化的一部分。如果用 initramfs 挂载真正的 rootfs 做 pivot_root，总体时间是否比直接挂载 ext4 更短？

4. **模块化风险**：如果把 ISP 驱动改为模块，启动时按需 `modprobe`，第一次打开摄像头可能延迟。用户应该感知到"摄像头冷启动"的时间吗？

5. **系统计时准确性**：U-Boot 的 `bootstage` 使用哪个硬件定时器？与内核的 `sched_clock()` 是否使用同一个时钟源？两个阶段的耗时能否直接相加？

---

## 相关笔记

- [[bsp-boot-flow]] — 阶段一主笔记 (Boot Chain, 实验 3-5)
- [[bsp-uboot-adaptation]] — U-Boot 板级适配 (Defconfig 配置)
- [[bsp-spl-fit]] — SPL FIT 解析源码追踪 (验签耗时)
- [[bsp-uboot-secureboot]] — 安全启动 (启用签名会 +50ms)
- [[bsp-device-model-dtb]] — 阶段二: 设备模型 + 设备树 (驱动 probe)
- [[kernel-debug-env]] — 附录A: 内核调试环境 (Ftrace)
- [[MOC-嵌入式Linux]] — 嵌入式 Linux 学习路线 MOC
