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
// drivers/cpu/rockchip_amp.c — 从 amp 分区加载 AMP 固件
int amp_cpus_on(void)
{
    struct blk_desc *dev_desc;
    bootm_headers_t images;
    disk_partition_t part;
    void *hdr, *fit;
    int offset, cnt;
    int totalsize;
    int ret = 0;

    // 1. 获取块设备 (eMMC/SD)
    dev_desc = rockchip_get_bootdev();
    if (!dev_desc)
        return -EIO;

    // 2. 查找 amp 分区信息
    if (part_get_info_by_name(dev_desc, AMP_PART, &part) < 0)
        return -ENODEV;

    // 3. 读取 FIT 头部 (前 FIT_HEADER_SIZE 字节)
    hdr = memalign(ARCH_DMA_MINALIGN, FIT_HEADER_SIZE);
    if (!hdr)
        return -ENOMEM;

    offset = part.start;
    cnt = DIV_ROUND_UP(FIT_HEADER_SIZE, part.blksz);
    if (blk_dread(dev_desc, offset, cnt, hdr) != cnt) {
        ret = -EIO;
        goto out2;
    }

    // 4. 验证 FIT 格式 + 获取镜像总大小
    if (fdt_check_header(hdr)) {
        AMP_E("Not fit\n");
        ret = -EINVAL;
        goto out2;
    }
    if (fit_get_totalsize(hdr, &totalsize)) {
        AMP_E("No totalsize\n");
        ret = -EINVAL;
        goto out2;
    }

    // 5. 分配完整 FIT 缓冲区
    fit = memalign(ARCH_DMA_MINALIGN, ALIGN(totalsize, part.blksz));
    if (!fit) {
        printf("No memory\n");
        ret = -ENOMEM;
        goto out2;
    }

    // 6. 拷贝头部 + 读取剩余数据
    memcpy(fit, hdr, FIT_HEADER_SIZE);
    offset += cnt;
    cnt = DIV_ROUND_UP(totalsize, part.blksz) - cnt;
    if (blk_dread(dev_desc, offset, cnt, fit + FIT_HEADER_SIZE) != cnt) {
        ret = -EIO;
        goto out1;
    }

    // 7. 解析并分发 AMP 固件
    ret = parse_os_amp_dispatcher();
    if (ret < 0) {
        ret = -EINVAL;
        goto out1;
    }

    // 8. 提取 loadables 并加载到目标内存
    memset(&images, 0, sizeof(images));
    images.fit_uname_cfg = "conf";
    images.fit_hdr_os = fit;
    images.verify = 1;
    ret = boot_get_loadable(0, NULL, &images, IH_ARCH_DEFAULT, NULL, NULL);
    if (ret) {
        AMP_E("Load loadables, ret=%d\n", ret);
        goto out1;
    }
    flush_dcache_all();

    // 9. 唤醒所有 AMP 核
    ret = brought_up_all_amp(images.fit_hdr_os, images.fit_uname_cfg);
    if (ret)
        AMP_E("Brought up amps, ret=%d\n", ret);

    // ★★ 注意: 这里存在一个 bug — 加载过程中 images 已被
    // boot_get_loadable() 内部修改, 但 brought_up_all_amp()
    // 依赖 images.fit_hdr_os 指向有效内存 (即 fit).
    // 如果 loadable 提取破坏了 images 结构体成员, brought_up_all_amp
    // 使用的 fit_hdr_os 可能已是悬空指针.

out1:
    free(fit);
out2:
    free(hdr);

    return ret;
}
```

| 阶段 | 操作 | 说明 |
|------|------|------|
| 1~2 | 获取设备 + 分区 | `rockchip_get_bootdev()` 获取 eMMC/SD, `AMP_PART` 是 `"amp"` 分区名 |
| 3~4 | 读 FIT 头 + 解析 | 先读头部验证 FIT 格式, `fit_get_totalsize` 获取总大小 |
| 5~6 | 分配 + 读完整镜像 | 按 `totalsize` 对齐 `blksz` 分配, 头部拷贝 + 剩余部分读取 |
| 7~8 | parse + loadables | `parse_os_amp_dispatcher` 解析镜像, `boot_get_loadable` 将 loadable 节加载到 `load` 地址 |
| 9 | 唤醒 AMP 核 | `brought_up_all_amp` 配置 mailbox/共享内存, 释放 RISC-V 核复位 |

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

# Linux 侧 AMP 状态查询:
cat /sys/rk_amp/boot_cpu     # 显示 AMP 帮助信息
echo status 0 | sudo tee /sys/rk_amp/boot_cpu   # 查询 CPU0 AMP 状态 → dmesg
echo on 0 | sudo tee /sys/rk_amp/boot_cpu        # 使能 CPU0 AMP 核

# 注意: 写入 sysfs 需用 sudo tee 而非 sudo echo >,
#       因为 > 重定向由当前 shell 执行, sudo 只影响 echo 命令.
#       也可用 sudo -i 进入 root shell 后直接 echo.

dmesg | grep -i amp         # 查看 AMP 驱动日志
# 预期输出:
#   cpu[0] amp is disabled (0)    # AMP 未使能
#   cpu[0] amp is enabled (1)     # AMP 已使能
#   cpu[0] amp is on (2)          # AMP 核正在运行
#   cpu[0] is unavailable         # AMP 核未就绪 (MCU 固件未加载)

# 注意: AMP 驱动暴露的唯一 sysfs 接口是 /sys/rk_amp/boot_cpu (sysfs),
#       不支持 debugfs 或单独的 mailbox/status 文件.
#       更多调试需通过 rockchip_amp.c 中的 pr_info 输出查看 dmesg.
```

### 6.1 sysfs 接口源码

此接口实现在 `drivers/soc/rockchip/rockchip_amp.c`：

```c
// rockchip_amp.c:117-133 — boot_cpu show 实现
static ssize_t boot_cpu_show(struct device *dev,
                             struct device_attribute *attr,
                             char *buf)
{
    char *str = buf;

    str += sprintf(str, "cpu on/off:\n");
    str += sprintf(str,
        "         echo on/off [cpu id] > /sys/rk_amp/boot_cpu\n");
    str += sprintf(str, "get cpu on/off status:\n");
    str += sprintf(str,
        "         echo status [cpu id] > /sys/rk_amp/boot_cpu\n");
    if (str != buf)
        *(str - 1) = '\n';

    return (str - buf);
}

// rockchip_amp.c:224-226 — 属性数组
static struct device_attribute rk_amp_attrs[] = {
    __ATTR(boot_cpu, 0664, boot_cpu_show, boot_cpu_store),
};

// rockchip_amp.c:716 — kobject 创建 (sysfs 目录)
rk_amp_kobj = kobject_create_and_add("rk_amp", NULL);
// → /sys/rk_amp/ 下挂 boot_cpu 属性

// rockchip_amp.c:135-150 — store 最终调用 ATF SMC 查询状态
static void cpu_status_print(unsigned long cpu_id, struct arm_smccc_res *res)
{
    if (res->a1 == AMP_CPU_STATUS_AMP_DIS)
        pr_info("cpu[%lx] amp is disabled (%ld)\n", cpu_id, res->a1);
    else if (res->a1 == AMP_CPU_STATUS_EN)
        pr_info("cpu[%lx] amp is enabled (%ld)\n", cpu_id, res->a1);
    else if (res->a1 == AMP_CPU_STATUS_ON)
        pr_info("cpu[%lx] amp is on (%ld)\n", cpu_id, res->a1);
}
```

`boot_cpu_store()` 通过 `arm_smccc_smc` 调用 ATF SMC 指令，由 ATF 固件操作 RISC-V MCU 核的电源状态——这解释了为什么 AMP 电源管理是 ATF 层的职责。`cpu[0] is unavailable` 表示 MCU 固件 (rtt.bin) 未加载到 `0x48c02000`。

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
