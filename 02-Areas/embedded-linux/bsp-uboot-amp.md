---
tags:
  - embedded-linux
  - bsp
  - bootloader
  - u-boot
  - amp
  - risc-v
  - mcu
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

# U-Boot AMP Boot 详解

> **前置笔记**：[[bsp-boot-flow]] — AMP 架构概览
>
> **核心代码**：`drivers/cpu/rockchip_amp.c`, `include/amp.h`, `drivers/cpu/amp.its`

---

## 一、AMP 架构回顾

RV1126B 的 AMP (Asymmetric Multi-Processing) 架构：

```
┌──────────────────────────────────────────────────────┐
│ SoC RV1126B                                          │
│                                                      │
│  ┌─────────────────────────┐  ┌──────────────────┐  │
│  │ Cortex-A53 ×4 (Linux)   │  │ RISC-V MCU       │  │
│  │                         │  │ (RT-Thread/HAL)   │  │
│  │  权责: 网络, 存储,      │  │  权责: 实时控制,  │  │
│  │  图像处理, 复杂逻辑      │  │  传感器, 电机     │  │
│  └──────────┬──────────────┘  └────────┬─────────┘  │
│             │                          │             │
│             └──────────┬───────────────┘             │
│                        │                             │
│                 Mailbox ×8                           │
│            Shared Memory (RPMSG)                    │
│            Hardware Lock (hwlock)                   │
└──────────────────────────────────────────────────────┘
```

---

## 二、AMP 固件格式

### 2.1 `amp_mcu.its` 结构

```dts
// device/rockchip/.chips/rv1126b/amp_mcu.its
/ {
    images {
        hpmcu {
            data = /incbin/("rtt.bin");       // RT-Thread 固件二进制
            type = "standalone";                // 独立固件 (非 Linux)
            arch = "arm";                       // 实际为 RISC-V, 但 U-Boot 限制了 arch 值
            load = <0x48c02000>;                // ★ MCU 固件加载地址
            udelay = <10000>;                   // 加载后等待 10ms
        };
    };
    share {
        rpmsg_base = <0x48c3c000>;              // ★ 共享内存基址
        rpmsg_size = <0x00040000>;              // ★ 共享内存大小 (256KB)
    };
    configurations {
        conf {
            loadables = "hpmcu";                // 标记为 loadable 固件
            rollback-index = <0x0>;
            signature { ... };
        };
    };
};
```

### 2.2 内存布局

```
RV1126B AMP 内存映射:

0x40000000 ┌────────────────────────┐
           │ Linux + U-Boot        │  DDR 起始, 主核使用
           │ (Cortex-A53)          │
           │                      │
0x48c00000 ├────────────────────────┤
           │ MCU 固件 (RISC-V)     │  ← amp_mcu.its 中 load = 0x48c02000
           │ RT-Thread / HAL       │     rtt.bin 从这里加载执行
0x48c3c000 ├────────────────────────┤
           │ RPMSG 共享内存        │  ← share.rpmsg_base
           │ (256KB)               │     Mailbox 通信数据区
0x48c7c000 ├────────────────────────┤
           │ ...                   │
0xc0000000 └────────────────────────┘  DDR 结束
```

---

## 三、AMP 启动流程

### 3.1 U-Boot 中的 AMP 加载

```
U-Boot board_init_r() → board_late_init()
  │
  └─ amp_cpus_on()                ← include/amp.h, drivers/cpu/rockchip_amp.c
       │
       ├─ 1. 读取 DTS /rockchip-amp 节点
       │    ├─ 解析 amp-cpus (哪些 CPU 核给 MCU)
       │    └─ 解析共享内存配置
       │
       ├─ 2. 从 amp 分区加载 amp.img
       │    │  mmc dev 0
       │    │  load mmc 0:4 0x42000000 amp.img   ← 分区 4 = amp 分区
       │    │
       │    └─ 解析 amp.img (FIT 格式)
       │         ├─ images/hpmcu → data = rtt.bin
       │         ├─ load = 0x48c02000
       │         └─ 将 rtt.bin 复制到 0x48c02000
       │
       ├─ 3. 配置 mailbox 中断
       │    │  GICD 配置: 为 MCU 分配中断向量
       │    │
       │    └─ 配置共享内存地址到 MCU 可见的地址空间
       │
       ├─ 4. 释放 MCU 复位
       │    │  写寄存器释放 RISC-V 核复位
       │    │
       │    └─ MCU 从 0x48c02000 开始执行
       │
       └─ 5. 返回 Linux 启动
            └─ Linux 启动后通过 mailbox 驱动与 MCU 通信
```

### 3.2 关键函数 `amp_cpus_on()`

```c
// drivers/cpu/rockchip_amp.c
int amp_cpus_on(void)
{
    // 1. 检测 DTS 中是否启用了 AMP
    //    /rockchip-amp { status = "okay"; };

    // 2. 从 amp 分区加载 FIT 镜像
    bootm_find_images();              // 定位 amp 分区
    fit_get_data(amp_fit, "hpmcu");   // 提取 rtt.bin 数据

    // 3. 将 MCU 固件复制到指定地址
    memcpy((void *)0x48c02000, rtt_bin, rtt_size);

    // 4. 初始化共享内存
    memset((void *)rpmsg_base, 0, rpmsg_size);

    // 5. 配置 GIC: SPI 中断路由到 MCU
    gicd_writel(..., GICD_ISENABLER);

    // 6. 释放 MCU 复位
    writel(0, MCU_RST_CTRL);          // 释放 RISC-V 核

    // 7. 等待 MCU 启动
    udelay(10000);                    // amp_mcu.its 中的 udelay

    return 0;
}
```

### 3.3 Linux 侧的 AMP 初始化

```dts
// kernel DTS: rv1126b-sportcam.dts
/ {
    rockchip-amp {
        compatible = "rockchip,amp";
        status = "okay";

        amp-cpus = <&mcu>;           // RISC-V MCU

        // 共享内存区域
        amp-rpmsg {
            reg = <0x0 0x48c3c000 0x0 0x40000>;  // 256KB
        };

        // Mailbox 通道
        amp-mailbox {
            // 8 个 mailbox 通道
            mboxes = <&mailbox 0>, <&mailbox 1>, ...;
        };
    };
};
```

---

## 四、AMP 通信机制

### 4.1 Mailbox

```
RV1126B 有 8 个硬件 Mailbox 通道:

           ┌──────────────────────────┐
           │        Mailbox           │
           │  ┌────┐  ┌────┐         │
Linux ─────┼─┤Ch 0├──┤Ch 1├─────────┼── MCU
           │  ├────┤  ├────┤         │
           │  │Ch 2│  │Ch 3│         │
           │  ├────┤  ├────┤         │
           │  │Ch 4│  │Ch 5│         │
           │  ├────┤  ├────┤         │
           │  │Ch 6│  │Ch 7│         │
           │  └────┘  └────┘         │
           └──────────────────────────┘
使用方式:
  - Linux 写 mailbox 寄存器 → MCU 接收中断
  - MCU 写 mailbox 寄存器 → Linux 接收中断
  - 数据通过 RPMSG 共享内存传输
```

### 4.2 RPMSG 协议

```
┌─────────────────────────────────────────┐
│ RPMSG 共享内存 (0x48c3c000, 256KB)      │
├─────────────────────────────────────────┤
│ 头部 (64 bytes):                        │
│   src_cpu, dst_cpu, len, flags, ...     │
├─────────────────────────────────────────┤
│ 数据负载 (可变长度):                     │
│   struct rpmsg_data {                   │
│       u32 cmd;       // 命令 ID        │
│       u32 seq;       // 序列号         │
│       u8  payload[]; // 数据           │
│   };                                    │
├─────────────────────────────────────────┤
│ ...                                      │
│                                         │
│ 尾部: status, CRC (可选)                │
└─────────────────────────────────────────┘
```

### 4.3 Hardware Spinlock

```c
// 防止 Linux 和 MCU 同时访问共享资源的硬件锁
// rv1126b-hwspinlock 驱动

// Linux 侧:
hwlock = hwspin_lock_request();
hwspin_lock_timeout(hwlock, 1000);
// 访问共享内存
hwspin_unlock(hwlock);

// MCU 侧同样使用硬件 spinlock
// → 保证访问互斥
```

---

## 五、AMP 配置文件

`sportcam` 的 AMP 配置：

```bash
# device/rockchip/.chips/rv1126b/rv1126b_sportcam_defconfig
RK_AMP=y
RK_AMP_RISCV=y                              # RISC-V MCU
RK_AMP_RTT_TARGET="rv1126b-mcu"             # RT-Thread 目标
RK_AMP_FIT_ITS="amp_mcu.its"                # FIT 描述文件
```

U-Boot defconfig 中的 AMP 配置：

```kconfig
# u-boot/configs/rk-amp.config (fragment)
CONFIG_AMP=y
CONFIG_CMD_AMP=y
CONFIG_ROCKCHIP_AMP=y
CONFIG_AMP_DTB=y
```

---

## 六、AMP 调试命令

```bash
# U-Boot shell 中:
=> amp list                  # 列出所有 AMP 固件
=> amp load 0 0x48c02000     # 手动加载 AMP 固件到地址
=> amp start 0               # 启动 AMP 核

# 板端 Linux 中查看 AMP 状态:
cat /sys/kernel/debug/amp/status    # AMP 状态
cat /sys/kernel/debug/amp/mailbox   # Mailbox 状态
dmesg | grep amp                    # AMP 驱动日志

# RPMSG 通信测试:
# Linux 侧:
echo "ping" > /dev/rpmsg0
cat /dev/rpmsg0                     # 预期: "pong"

# MCU 侧 (RT-Thread):
# msh > amp_ping
# 预期: RPMSG ping-pong 测试通过
```

---

## 七、思考题

1. **AMP 启动顺序**：U-Boot 在加载 Linux 之前就启动了 MCU。如果 MCU 在 Linux 完全启动前就开始工作，但 Linux 需要重新配置部分硬件寄存器，如何避免状态冲突？

2. **内存隔离**：Linux 和 MCU 共享 DDR 空间，如何防止 Linux 错误地覆盖 MCU 的代码或数据段？

3. **DJI 对标**：大疆无人机的飞行控制器通常运行 RTOS，而图传/相机运行 Linux。AMP 架构如何映射到大疆的"Linux + RTOS"双系统架构？mailbox 和 RPMSG 的职责分别对应什么？

---

## 相关笔记

- [[bsp-boot-flow]] — AMP 架构概览
- [[bsp-uboot-adaptation]] — AMP 启动配置
- [[bsp-uboot-rktools]] — amp.img 打包
- [[MOC-嵌入式Linux]] — 嵌入式 Linux 学习路线 MOC
