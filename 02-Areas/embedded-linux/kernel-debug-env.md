---
tags:
  - embedded-linux
  - kernel
  - debug
  - ftrace
  - perf
  - lockdep
  - ebpf
category: embedded-linux
created: 2026-06-17
updated: 2026-06-17
status: active
sdk: atk_dlrv1126b_linux6.1_sdk_release_v1.2.1_20260327
kernel: Linux 6.1.141
board: RV1126B SportCam (ATK SDK)
toolchain: gcc-arm-10.3-2021.07 (aarch64-none-linux-gnu-gcc 10.3.1)
---

# 内核 Debug 环境搭建

## 目标

在 RV1126B 上搭建完整的内核调试工具链，为后续驱动开发、性能分析、实时性优化打好基础。

---

## 一、环境基线

### 1.1 工具链

SDK 自带交叉编译工具链，不在系统默认 PATH 中，需要临时添加：

```bash
export PATH=$PWD/prebuilts/gcc/linux-x86/aarch64/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin:$PATH
aarch64-none-linux-gnu-gcc --version
```

也可以加到 `~/.bashrc` 里省得每次重敲。

### 1.2 采集结果

| 项目 | 值 |
|------|-----|
| GCC | aarch64-none-linux-gnu-gcc 10.3.1 20210621 |
| 内核 | 6.1.141 |
| SDK | atk_dlrv1126b_linux6.1_sdk_release_v1.2.1_20260327 |

**命令：**

```bash
head -5 kernel/Makefile | grep -E 'VERSION|PATCHLEVEL|SUBLEVEL'
grep -E '^CONFIG_(FTRACE|FUNCTION_TRACER|DYNAMIC_FTRACE|STACK_TRACER|IRQSOFF_TRACER|...)=' kernel/.config
```

### 1.3 初始 debug 选项

| 选项 | 初始状态 | 说明 |
|------|----------|------|
| CONFIG_FTRACE | ✅ | Ftrace 框架总开关 |
| CONFIG_FUNCTION_TRACER | ✅ | 函数级追踪 |
| CONFIG_DYNAMIC_FTRACE | ✅ | 运行时开关 ftrace |
| CONFIG_DYNAMIC_DEBUG | ✅ | 动态 printk 开关 |
| CONFIG_DEBUG_FS | ✅ | debugfs 文件系统 |
| CONFIG_PERF_EVENTS | ✅ | Perf 事件 |
| CONFIG_DEBUG_INFO | ✅ | 调试符号 |
| CONFIG_UPROBES | ✅ | 用户态探针 |
| CONFIG_LOCKDEP_SUPPORT | ✅ | Lockdep 基础设施（但 LOCKDEP 未开） |
| CONFIG_FUNCTION_GRAPH_TRACER | ❌ | 函数调用图 |
| CONFIG_STACK_TRACER | ❌ | 内核栈回溯 |
| CONFIG_IRQSOFF_TRACER | ❌ | 关中断时间追踪 |
| CONFIG_PREEMPT_TRACER | ❌ | 抢占延迟追踪 |
| CONFIG_SCHED_TRACER | ❌ | 调度追踪 |
| CONFIG_LOCKDEP | ❌ | 锁依赖验证 |
| CONFIG_PROVE_LOCKING | ❌ | 锁使用正确性证明 |
| CONFIG_BPF_SYSCALL | ❌ | eBPF 系统调用 |
| CONFIG_LATENCYTOP | ❌ | 延迟分析 |
| CONFIG_DEBUG_VM | ❌ | 内存调试 |
| CONFIG_KMEMLEAK | ❌ | 内存泄漏检测 |

---

## 二、创建 debug 配置片段

### 2.1 文件位置

> ⚠️ **实操修正**：配置片段放在 `kernel-6.1/arch/arm64/configs/` 目录下。
>
> SDK 中已有的示例 `rockchip_amp.config` 也在这个目录，不要放到 `kernel/configs/`，那不是这个 SDK 的惯例。

```
kernel-6.1/arch/arm64/configs/rockchip_debug.config
```

### 2.2 文件内容

```bash
# ========== Ftrace 增强 ==========
CONFIG_FUNCTION_GRAPH_TRACER=y
CONFIG_STACK_TRACER=y
CONFIG_IRQSOFF_TRACER=y
CONFIG_PREEMPT_TRACER=y
CONFIG_SCHED_TRACER=y
CONFIG_FTRACE_SYSCALLS=y
CONFIG_TRACER_SNAPSHOT=y
CONFIG_TRACER_SNAPSHOT_PER_CPU_SNAPSHOT=y
CONFIG_HIST_TRIGGERS=y

# ========== Lockdep 锁调试 ==========
CONFIG_LOCKDEP=y
CONFIG_LOCK_STAT=y
CONFIG_PROVE_LOCKING=y
CONFIG_DEBUG_ATOMIC_SLEEP=y

# ========== 内存调试 ==========
CONFIG_DEBUG_VM=y
CONFIG_DEBUG_VM_VMACACHE=y
CONFIG_DEBUG_VM_RB=y
CONFIG_DEBUG_VM_PGFLAGS=y
# KMEMLEAK 有显著性能开销，初次构建可先注释
# CONFIG_DEBUG_KMEMLEAK=y

# ========== DebugFS + Dynamic Debug ==========
CONFIG_DEBUG_FS=y
CONFIG_DYNAMIC_DEBUG=y
CONFIG_GENERIC_IRQ_DEBUGFS=y

# ========== Perf 增强 ==========
CONFIG_PERF_EVENTS=y
CONFIG_HW_PERF_EVENTS=y

# ========== 延迟追踪 ==========
CONFIG_LATENCYTOP=y

# ========== 关闭 debug info 缩减（保留完整符号）==========
# CONFIG_DEBUG_INFO_REDUCED is not set

# ========== eBPF 基础 ==========
CONFIG_BPF=y
CONFIG_BPF_SYSCALL=y
CONFIG_BPF_JIT=y
CONFIG_HAVE_EBPF_JIT=y
```

### 2.3 配置项速查

| 配置 | 学习重点 | 面试价值 |
|------|----------|----------|
| FUNCTION_GRAPH_TRACER | 看内核函数调用链 | "如何分析性能？" |
| IRQSOFF_TRACER | 找最大关中断时间 | "中断下半部机制？" |
| LOCKDEP / PROVE_LOCKING | 死锁检测 | "spinlock 能 sleep 吗？" |
| DYNAMIC_DEBUG | 不重编内核开调试 | 实用技巧 |
| BPF_SYSCALL | eBPF 观测内核 | 大势所趋 |

---

## 三、构建内核

### 3.1 关键知识点

> ⚠️ **核心注意**：`./build.sh kernel` 会从 `output/.config` 中读取 `RK_KERNEL_CFG` 和 `RK_KERNEL_CFG_FRAGMENTS`，然后**重新执行 defconfig**。所以直接在 `kernel-6.1/` 下手动 `merge_config.sh` 修改的 `.config` 会被覆盖，debug 选项不会被编译进去。
>
> **正确做法**：把 `rockchip_debug.config` 作为 fragment 注册到 SDK 的配置系统中。

### 3.2 构建步骤

#### 步骤一：选择板级配置

```bash
cd <SDK_TOP>
make rv1126b_sportcam_defconfig
```

#### 步骤二：将 debug 配置片段注册到 SDK

编辑 `output/.config`，找到 `RK_KERNEL_CFG_FRAGMENTS`，修改为：

```
RK_KERNEL_CFG_FRAGMENTS="rockchip_amp.config rockchip_debug.config"
```

也可以用 sed 命令直接改：

```bash
sed -i 's/RK_KERNEL_CFG_FRAGMENTS="rockchip_amp.config"/RK_KERNEL_CFG_FRAGMENTS="rockchip_amp.config rockchip_debug.config"/' output/.config
```

#### 步骤三：编译

```bash
./build.sh kernel
```

`build.sh` 会自动完成：
1. 从 `rv1126b_sportcam_defconfig` 生成基础 `.config`
2. 逐一合并 `rockchip_amp.config` 和 `rockchip_debug.config` 两个 fragment
3. 解决依赖（`olddefconfig`）
4. 编译 Image
5. 编译 DTB
6. 打包 boot.img（FIT 格式）

#### 步骤四：验证

```bash
grep CONFIG_FUNCTION_GRAPH_TRACER kernel-6.1/.config
grep CONFIG_LOCKDEP kernel-6.1/.config
```

应为 `=y`。

#### 产物

```
output/firmware/boot.img   # 完整 FIT 镜像（kernel + dtb + resource）
kernel-6.1/arch/arm64/boot/Image
kernel-6.1/arch/arm64/boot/dts/rockchip/rv1126b-sportcam.dtb
```

### 3.2 编译结果

| 指标 | 值 |
|------|-----|
| 编译结果 | ✅ 通过 |
| Image 大小 | ~38 MB（debug info 导致比 release 大很多） |
| boot.img 大小 | ~39 MB |
| DTB | `rv1126b-sportcam.dtb` (170 KB) |
| FIT 打包文件 | `output/firmware/boot.img` |

### 3.3 产物路径说明

```
kernel-6.1/arch/arm64/boot/
├── Image              # 内核镜像（未经压缩）
├── Image.lz4          # LZ4 压缩后的内核
└── dts/rockchip/
    └── rv1126b-sportcam.dtb

output/firmware/
└── boot.img           # FIT 格式的完整启动镜像（含 kernel + dtb + resource）
```

---

## 四、部署

### 4.1 网络连接

```bash
# 主机配置静态 IP（与板子同网段）
sudo ip addr add 192.168.1.100/24 dev ethX
sudo ip link set ethX up
```

### 4.2 部署方式

| 方式 | 适用场景 |
|------|----------|
| TFTP | 开发调试最快，每次启动从网络加载 |
| scp | 已有系统，替换 /boot 下的 Image |
| 烧录 | 首次或全量更新 |

---

## 五、板端验证结果

### 5.1 部署方式

通过 SSH 连接板子（`rooter@192.168.1.109`），使用 `dd` 直接写 boot 分区：

```bash
# 分区布局（/dev/block/by-name/）
boot     → /dev/mmcblk0p3   # 内核分区，64 MB
rootfs   → /dev/mmcblk0p7   # 文件系统，不动
oem      → /dev/mmcblk0p8
userdata → /dev/mmcblk0p9

# 备份旧内核
sudo dd if=/dev/mmcblk0p3 of=/tmp/boot_backup.img bs=1M status=progress

# 写入新内核（先 scp 传到 /tmp/）
sudo dd if=/tmp/boot.img of=/dev/mmcblk0p3 bs=1M status=progress

# 重启
sudo reboot
```

### 5.2 内核版本确认

```bash
dmesg | grep "Linux version"
# → Linux version 6.1.141 (aarch64-none-linux-gnu-gcc 10.3.1 20210621)
```

### 5.3 Ftrace

```bash
sudo cat /sys/kernel/tracing/available_tracers
# → blk function_graph wakeup_dl wakeup_rt wakeup irqsoff function nop
```

可见 tracer：`function_graph` ✓、`irqsoff` ✓、`wakeup` ✓，但 `preemptoff` 未出现（需要在配置中进一步开启 `PREEMPT_TRACER`）。

### 5.4 Lockdep

```bash
sudo cat /proc/lockdep
# → all lock classes: ...（正常输出锁依赖信息，不报错）
```

Lockdep 正常工作。

### 5.5 Perf

> ⏳ 暂未安装。板子无网络，待有网后 `sudo apt install linux-perf` 或从内核源码编译。

### 5.6 注意事项

- `/sys/kernel/tracing/` 和 `/proc/lockdep` 等 debug 文件需要 `sudo` 或 root 权限才能读取
- tracefs 在系统启动时已自动挂载（`mount` 显示已存在）

---

## 六、常用操作 CheatSheet

```bash
# ==================== Ftrace ====================
echo function_graph > /sys/kernel/tracing/current_tracer
echo nop > /sys/kernel/tracing/current_tracer
echo do_sys_open > /sys/kernel/tracing/set_graph_function
cat /sys/kernel/tracing/trace > /tmp/trace.log
echo > /sys/kernel/tracing/trace                        # 清缓冲区

# ==================== Dynamic Debug ====================
echo 'file v4l2-* +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file xxx.c -p' > /sys/kernel/debug/dynamic_debug/control

# ==================== Perf ====================
perf stat -e cycles,instructions,cache-misses sleep 1
perf record -F 99 -a -g -- sleep 10
perf report

# ==================== Lockdep ====================
cat /proc/lockdep
cat /proc/lock_stat

# ==================== 内存 ====================
cat /sys/kernel/debug/cma/*
cat /proc/meminfo
```

---

## 七、踩坑记录

| 日期 | 问题 | 原因 | 解决方案 |
|------|------|------|----------|
| 2026-06-17 | `aarch64-none-linux-gnu-gcc: command not found` | SDK 工具链不在系统 PATH | `export PATH=$PWD/prebuilts/gcc/linux-x86/aarch64/gcc-arm-10.3-.../bin:$PATH` |
| 2026-06-17 | 配置片段放到了 `kernel/configs/` 但 SDK 不识别 | 路径错误，SDK 的配置片段在 `arch/arm64/configs/` | 移到 `arch/arm64/configs/rockchip_debug.config` |
| 2026-06-17 | `make dtbs` 报错 `No rule to make target '...mipinodisplay.dtb'` | 内核 Makefile 引用了不存在的 DTB 文件 | 在 `arch/arm64/boot/dts/rockchip/Makefile` 中注释掉对应行，改用 `./build.sh kernel` 编译 |
| 2026-06-17 | `./build.sh kernel` 编出来的 Image 没有 debug 选项 | `build.sh` 从 `output/.config` 重新执行 defconfig，覆盖了手动 merge 的结果 | 修改 `RK_KERNEL_CFG_FRAGMENTS` 加入 `rockchip_debug.config` |
| 2026-06-17 | `merge_config.sh -O .config` 报错 | `-O` 参数应接目录而非文件名 | 改成 `-O .` |
| 2026-06-17 | 编译过程弹出 `BPF_JIT_ALWAYS_ON` 交互提示 | `merge_config` 后新依赖未自动处理 | 先执行 `make olddefconfig` 再编译 |
| 2026-06-17 | Image 达 38 MB | CONFIG_DEBUG_INFO 保留完整符号导致 | 正常现象，release 版本会小很多 |
| 2026-06-17 | `available_tracers` 中缺少 `preemptoff` | `CONFIG_PREEMPT_TRACER` 未生效（可能被依赖选项阻止） | 后续排查 |
| 2026-06-17 | 写 boot 分区需要 sudo 权限 | 块设备 root 所有 | 全程加 `sudo` |
| 2026-06-17 | 板子无网络，perf 无法安装 | 开发环境限制 | 暂用 ftrace 替代，有网后补 `apt install linux-perf` |
| 2026-06-17 | `/sys/kernel/tracing` 和 `/proc/lockdep` 普通用户无权读取 | 权限限制 | 加 `sudo` |

---

## 八、下阶段预告

阶段二：**V4L2 + ISP 驱动深度**
1. 手动写 V4L2 程序抓一帧图像
2. 用 Ftrace 追踪 ISP 驱动路径
3. 用 Dynamic Debug 打开 RKAIQ 调试
4. 用 Perf 分析摄像头管线性能瓶颈

---

## 相关笔记

- [[MOC-嵌入式Linux]] — 嵌入式 Linux 学习地图
- [[rv1126b]] — RV1126B 运动相机项目
- [[BoardRoot-嵌入式Linux厂商适配框架知识库]]
