---
tags:
  - embedded-linux
  - bsp
  - interrupt
  - gic-400
  - irqchip
  - source-code
  - registers
  - rockchip
category: embedded-linux
created: 2026-06-23
updated: 2026-06-23
status: active
soc: Rockchip RV1126B
kernel: Linux 6.1.141
parent: bsp-interrupt-concurrency
---

# GIC-400 驱动源码深挖

> **前置笔记**：[[bsp-interrupt-concurrency]] — 阶段三：中断处理 + 并发
> [[bsp-interrupt-irqdomain-deep]] — IRQ Domain 源码
>
> **核心文件**：`drivers/irqchip/irq-gic.c`, `include/linux/irqchip/arm-gic.h`

---

## 一、GIC-400 硬件寄存器

### 1.1 寄存器映射

```
RV1126B DTS:
gic: interrupt-controller@21201000 {
    reg = <0x21201000 0x1000>,    // GICD (Distributor)
          <0x21202000 0x2000>,    // GICC (CPU Interface)
          <0x21204000 0x2000>,    // GICH (Virtualization)
          <0x21206000 0x2000>;    // GICV (Virtual CPU)
};
```

### 1.2 GICD 寄存器 (Distributor)

```c
// include/linux/irqchip/arm-gic.h
#define GIC_DIST_CTRL           0x000   // 全局控制
#define GIC_DIST_CTR            0x004   // 类型: 支持多少中断线
#define GIC_DIST_IIDR           0x008   // GIC 实现者 ID

#define GIC_DIST_IGROUP         0x080   // 中断组 (Group 0/1)
#define GIC_DIST_ENABLE_SET     0x100   // ★ 使能: 写 1 对应 bit 使能
#define GIC_DIST_ENABLE_CLEAR   0x180   // ★ 禁能: 写 1 对应 bit 禁能
#define GIC_DIST_PENDING_SET    0x200   // ★ 软件触发中断 (置 pending)
#define GIC_DIST_PENDING_CLEAR  0x280   // 清除 pending
#define GIC_DIST_ACTIVE_SET     0x300   // 置 active
#define GIC_DIST_ACTIVE_CLEAR   0x380   // 清 active
#define GIC_DIST_PRI            0x400   // 优先级 (每中断 1 byte)
#define GIC_DIST_TARGET         0x800   // CPU 目标 (每中断 1 byte)
#define GIC_DIST_CONFIG         0xc00   // 配置: 电平/边沿触发
#define GIC_DIST_SOFTINT        0xf00   // SGI 软件触发寄存器

每个寄存器组按中断号索引:
  GIC_DIST_ENABLE_SET + (hwirq/32)*4
  GIC_DIST_CONFIG + (hwirq/16)*4
  GIC_DIST_PRI + hwirq
```

### 1.3 GICC 寄存器 (CPU Interface)

```c
#define GIC_CPU_CTRL            0x00    // CPU 接口控制
#define GIC_CPU_PRIMASK         0x04    // 优先级掩码
#define GIC_CPU_BINPOINT        0x08    // 优先级分组
#define GIC_CPU_INTACK          0x0c    // ★ 读取: 获取中断号 (IAR)
#define GIC_CPU_EOI             0x10    // ★ 写入: 中断结束 (EOI)
#define GIC_CPU_DEACTIVATE      0x1000  // EOImode=1 时去激活
#define GIC_CPU_RUNNINGPRI      0x14    // 当前 CPU 运行优先级
#define GIC_CPU_HIGHPRI         0x18    // 当前最高挂起中断优先级
#define GIC_CPU_IDENT           0xfc    // GIC 实现者 ID

#define GICC_IAR_INT_ID_MASK    0x3ff   // IAR 读出中断号掩码
#define GICC_INT_SPURIOUS       1023    // 伪中断 (无有效中断)
```

### 1.4 GICD_CTR — 类型寄存器

```c
// gic_init_bases() 中:
gic_irqs = readl_relaxed(gic_data_dist_base(gic) + GIC_DIST_CTR) & 0x1f;
gic_irqs = (gic_irqs + 1) * 32;     // 编码: 0x1f → 32*32=1024
                                      //         0x04 → 5*32=160 (RV1126B)
```

---

## 二、GIC 驱动初始化

### 2.1 `gic_of_init()` → `gic_init_bases()`

```c
// drivers/irqchip/irq-gic.c
static int __init gic_of_init(struct device_node *node,
                               struct device_node *parent)
{
    struct gic_chip_data *gic;

    gic = kzalloc(sizeof(*gic), GFP_KERNEL);

    // 1. ioremap GICD 和 GICC
    gic->raw_dist_base = of_iomap(node, 0);
    //                      → DTS reg = <0x21201000 0x1000>
    gic->raw_cpu_base  = of_iomap(node, 1);
    //                      → DTS reg = <0x21202000 0x2000>

    // 2. 检查 percpu_offset (非 banked GIC)
    of_property_read_u32(node, "cpu-offset", &gic->percpu_offset);

    // 3. 初始化
    ret = gic_init_bases(gic, of_node_to_fwnode(node));
}

static int gic_init_bases(struct gic_chip_data *gic,
                           struct fwnode_handle *handle)
{
    // 1. 设置 Dist/CPU base (banked alias)
    gic->dist_base.common_base = gic->raw_dist_base;
    gic->cpu_base.common_base  = gic->raw_cpu_base;

    // 2. ★ 读取 GICD_CTR, 获取硬件支持的中断数
    gic_irqs = readl_relaxed(dist_base + GIC_DIST_CTR) & 0x1f;
    gic_irqs = (gic_irqs + 1) * 32;
    //   RV1126B: 0x04 → 5 → (5+1)*32 = 160 个中断线
    if (gic_irqs > 1020) gic_irqs = 1020;
    gic->gic_irqs = gic_irqs;

    // 3. ★ 创建 IRQ Domain (线性映射)
    gic->domain = irq_domain_create_linear(handle, gic_irqs,
                                            &gic_irq_domain_hierarchy_ops,
                                            gic);

    // 4. 初始化 Distributor 硬件
    gic_dist_init(gic);
    //   → GICD_CTRL = enable (全局使能)
    //   → 设置 SPIs 为非安全组 (Group 1)
    //   → 设置所有 SPIs 优先级为 0xa0
    //   → 所有 SPIs 目标 CPU = CPU0

    // 5. 初始化 CPU Interface
    gic_cpu_init(gic);
    //   → GICC_CTRL = enable + FIQEn + EOImode
    //   → GICC_PRIMASK = 0xf0 (允许所有优先级)
    //   → 写 GICC_EOI 清空残留中断
}
```

### 2.2 `gic_dist_init()` — Distributor 硬件初始化

```c
static void gic_dist_init(struct gic_chip_data *gic)
{
    void __iomem *base = gic_data_dist_base(gic);
    unsigned int gic_irqs = gic->gic_irqs;
    u32 cpumask;
    int i;

    // 1. 禁用 Distributor
    writel_relaxed(GICD_DISABLE, base + GIC_DIST_CTRL);

    // 2. 所有 SPIs 设置为非安全 (Group 1)
    for (i = 32; i < gic_irqs; i += 32)
        writel_relaxed(~0, base + GIC_DIST_IGROUP + i / 4);

    // 3. 设置所有 SPIs 为电平触发 (GICD_CONFIG 每 2 bit 一个中断)
    for (i = 32; i < gic_irqs; i += 16)
        writel_relaxed(0, base + GIC_DIST_CONFIG + i / 4);
    //   GIC_DIST_CONFIG 每 2 bits 控制一个中断:
    //   00 = 电平触发, 01 = 边沿触发

    // 4. 设置优先级 (0xa0)
    for (i = 32; i < gic_irqs; i += 4)
        writel_relaxed(0xa0a0a0a0, base + GIC_DIST_PRI + i);

    // 5. 设置 CPU 目标 (所有 SPIs 发送到 CPU0)
    cpumask = 1 << 0;   // CPU0
    cpumask |= cpumask << 8;
    cpumask |= cpumask << 16;
    for (i = 32; i < gic_irqs; i += 4)
        writel_relaxed(cpumask, base + GIC_DIST_TARGET + i);

    // 6. 重新启用 Distributor
    writel_relaxed(GICD_ENABLE, base + GIC_DIST_CTRL);
}
```

---

## 三、中断操作函数

### 3.1 `gic_poke_irq()` — 底层寄存器操作

```c
// drivers/irqchip/irq-gic.c:187
static void gic_poke_irq(struct irq_data *d, u32 offset)
{
    u32 mask = 1 << (gic_irq(d) % 32);
    //     ↑ 中断号在 32-bit 寄存器中的 bit 位置

    writel_relaxed(mask,
        gic_dist_base(d) + offset + (gic_irq(d) / 32) * 4);
    //      寄存器基址      寄存器类型    寄存器编号 (每 32 中断一个)
    //
    // 例如: hwirq=77, offset=GIC_DIST_ENABLE_SET (0x100)
    //   77/32 = 2 (第 2 个 32-bit 寄存器)
    //   77%32 = 13 (第 13 bit)
    //   物理地址 = GICD_BASE + 0x100 + 2*4 = GICD_BASE + 0x108
    //   写入 bit13 = 1 → 使能 SPI #45
}
```

### 3.2 `gic_mask_irq()` — 禁能中断

```c
// drivers/irqchip/irq-gic.c:199
static void gic_mask_irq(struct irq_data *d)
{
    gic_poke_irq(d, GIC_DIST_ENABLE_CLEAR);
    // 写 1 到 GICD_ENABLE_CLEAR[hwirq] = 清除使能位
    // → Distributor 不再向 CPU 发送此中断
}
```

### 3.3 `gic_unmask_irq()` — 使能中断

```c
// drivers/irqchip/irq-gic.c:227
static void gic_unmask_irq(struct irq_data *d)
{
    gic_poke_irq(d, GIC_DIST_ENABLE_SET);
    // 写 1 到 GICD_ENABLE_SET[hwirq] = 设置使能位
    // → Distributor 开始向 CPU 发送此中断
}
```

### 3.4 `gic_eoi_irq()` — 中断结束

```c
// drivers/irqchip/irq-gic.c:236
static void gic_eoi_irq(struct irq_data *d)
{
    u32 hwirq = gic_irq(d);

    // SGI (<16) 需要特殊处理: 写入原始 IAR 值 (含 CPU 源)
    if (hwirq < 16)
        hwirq = this_cpu_read(sgi_intid);

    // 写入 GICC_EOI = 告知 GIC 中断处理完成
    writel_relaxed(hwirq, gic_cpu_base(d) + GIC_CPU_EOI);
}
```

### 3.5 `gic_set_type()` — 设置触发方式

```c
// drivers/irqchip/irq-gic.c:322
static int gic_set_type(struct irq_data *d, unsigned int type)
{
    // 约束:
    //   SGI (<16): 只能是边沿触发
    //   PPI (16-31): 需要硬件支持
    //   SPI (>=32): 仅支持电平高和边沿上升

    if (gicirq < 16)
        return type != IRQ_TYPE_EDGE_RISING ? -EINVAL : 0;
    if (gicirq >= 32 && type != IRQ_TYPE_LEVEL_HIGH
                     && type != IRQ_TYPE_EDGE_RISING)
        return -EINVAL;

    // 写 GICD_CONFIG 寄存器 (每 2 bits 一个中断)
    ret = gic_configure_irq(gicirq, type, base + GIC_DIST_CONFIG, NULL);
}
```

---

## 四、中断处理流程

### 4.1 `gic_handle_irq()` — 中断处理入口

```c
// drivers/irqchip/irq-gic.c:370
static void __exception_irq_entry gic_handle_irq(struct pt_regs *regs)
{
    u32 irqstat, irqnr;

    do {
        // ★ 1. 读取 GICC_INTACK = 获取当前最高优先级中断
        //     这个操作会"确认"中断 (acknowledge)
        irqstat = readl_relaxed(cpu_base + GIC_CPU_INTACK);
        irqnr = irqstat & GICC_IAR_INT_ID_MASK;
        //     GICC_IAR 低 10 位 = 中断号
        //     1023 = spurious (无有效中断)

        if (unlikely(irqnr >= 1020))
            break;

        // ★ 2. EOImode=0: 立即写 EOI (只表示收到, 不等处理完)
        //     EOImode=1: 实际在 irq handler 完成后写 DEACTIVATE
        if (supports_deactivate)
            writel_relaxed(irqstat, cpu_base + GIC_CPU_EOI);

        // ★ 3. 分发到 Linux 中断子系统
        generic_handle_domain_irq(gic->domain, irqnr);
        //     → irq_find_mapping(domain, irqnr)  ← 从 domain 找映射
        //     → desc->handle_irq() = handle_fasteoi_irq
        //       → 调用注册的 action handler
    } while (1);
}
```

### 4.2 完整中断响应路径

```
[硬件] 外设触发中断信号
    ↓
[GIC] GICD 检测到电平/边沿
    → 置 Pending 位
    → 根据 Target 和目标 CPU 的 Running Priority
    → 向目标 CPU 发送 IRQ 信号
    ↓
[CPU] 异常向量表
    → arch/arm64/kernel/entry.S
    → el1_irq → irq_handler
    → handle_arch_irq() = gic_handle_irq
    ↓
[gic_handle_irq]
    → readl(GICC_INTACK)     ← 获取中断号
    → writel(GICC_EOI)       ← EOImode=0: 立即 EOI
    → generic_handle_domain_irq(domain, irqnr)
      → irq_find_mapping(domain, irqnr)
      → desc->handle_irq()
        = handle_fasteoi_irq()
    ↓
[handle_fasteoi_irq]
    → desc->irq_data.chip->irq_ack()        ← (GIC 不需要 ack)
    → handle_irq_event(desc)
      → __handle_irq_event_percpu()
        → action->handler(irq, action->dev_id)  ← ★ 驱动注册的 handler
        → (返回 IRQ_WAKE_THREAD 则唤醒 threaded handler)
    → cond_unmask_eoi_irq(desc, chip)
      → chip->irq_unmask()   ← 如果边缘触发才 unmask
      → chip->irq_eoi()      ← EOImode=1: 写 DEACTIVATE
    ↓
[驱动中断处理结束]
```

---

## 五、中断亲和性 (Affinity)

### 5.1 GICD_TARGET 寄存器

```c
// 每个 SPI 有一个字节的 Target 寄存器
// 1 bit = 1 个 CPU 核
// bit0=CPU0, bit1=CPU1, ...

// gic_set_affinity:
static int gic_set_affinity(struct irq_data *d, const struct cpumask *mask_val,
                             bool force)
{
    // 写 GICD_TARGET + hwirq, 设置目标 CPU
    // 实现 SMP 中断分发
}
```

### 5.2 查看和设置

```bash
# 查看中断亲和性:
cat /proc/irq/77/smp_affinity
# 0001  → CPU0 独占
# 000f  → 4 个 CPU 都接收

# 设置中断亲和性 (将 I2C5 中断绑定到 CPU2):
echo 04 > /proc/irq/77/smp_affinity
# → GIC 将 SPI#45 路由到 CPU2
```

---

## 六、GIC 驱动关键操作速查

| 操作 | 函数 | 硬件操作 |
|------|------|---------|
| 使能中断 | `gic_unmask_irq` | 写 `GICD_ENABLE_SET + (hw/32)*4, 1 << (hw%32)` |
| 禁能中断 | `gic_mask_irq` | 写 `GICD_ENABLE_CLEAR + (hw/32)*4, 1 << (hw%32)` |
| 中断结束 | `gic_eoi_irq` | 写 `GICC_EOI, hwirq` |
| 设置触发 | `gic_set_type` | 写 `GICD_CONFIG + (hw/16)*4` (2 bits per IRQ) |
| 读取中断 | `gic_handle_irq` | 读 `GICC_INTACK` |
| 设置优先级 | — | 写 `GICD_PRI + hwirq` (1 byte per IRQ) |
| 设置目标 CPU | `gic_set_affinity` | 写 `GICD_TARGET + hwirq` (1 byte per IRQ) |
| 软件触发 SGI | — | 写 `GICD_SOFTINT` |
| 全局使能 GICD | `gic_dist_init` | 写 `GICD_CTRL, 1` |
| 全局使能 GICC | `gic_cpu_init` | 写 `GIC_CPU_CTRL, 0x1e0` |

---

## 七、思考题

1. **EOI 时机**：`gic_handle_irq()` 中在调用 `generic_handle_domain_irq()` 之前就写了 EOI。这意味着中断还未处理完时 GIC 就可以接收新的中断了。为什么这样设计？有什么风险？

2. **GICD_ENABLE_CLEAR vs GICD_ENABLE_SET**：为什么 GIC 设计了两组寄存器（一组写 1 使能，一组写 1 禁能），而不是直接用一个读写寄存器？这样设计的好处是什么？

3. **SGI 特殊处理**：`gic_eoi_irq()` 中 SGI 需要写回 `this_cpu_read(sgi_intid)` 而不是 hwirq，为什么？

4. **AMP 中断共享**：`gic_mask_irq()` 和 `gic_unmask_irq()` 中有 `#ifdef CONFIG_ROCKCHIP_AMP`，检查 `rockchip_amp_check_amp_irq()`。为什么 AMP 模式下要跳过某些中断的使能/禁能操作？(提示：MCU 也在使用 GIC)

---

## 相关笔记

- [[bsp-interrupt-concurrency]] — 阶段三：中断处理 + 并发
- [[bsp-interrupt-irqdomain-deep]] — IRQ Domain 源码深挖
- [[bsp-uboot-amp]] — AMP Boot (MCU 共享 GIC 中断)
- [[MOC-嵌入式Linux]] — 嵌入式 Linux 学习路线 MOC
