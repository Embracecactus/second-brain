---
tags:
  - embedded-linux
  - bsp
  - interrupt
  - gic-400
  - irqchip
  - handle_irq
  - top-half
  - bottom-half
  - source-code
category: embedded-linux
created: 2026-06-23
updated: 2026-06-23
status: active
kernel: Linux 6.1.141
parent: bsp-interrupt-concurrency
---

# 中断处理完整路径 — 从硬件触发到驱动 handler

> **前置笔记**：[[bsp-interrupt-concurrency]] — 阶段三：中断处理 + 并发
> [[bsp-interrupt-gic-deep]] — GIC-400 驱动
>
> **核心文件**：`kernel/irq/chip.c`, `kernel/irq/handle.c`, `kernel/irq/manage.c`
> `arch/arm64/kernel/entry.S`, `drivers/irqchip/irq-gic.c`

---

## 一、全景路径图

```
[外设] GPIO pin 拉低 / SPI 总线中断信号
   │
   ▼
[GIC-400 Distributor]
   GICD 检测到中断 → 置 Pending → 检查优先级/目标 CPU
   │
   ▼
[GIC-400 CPU Interface]
   GICC 向 CPU 核发送 IRQ 信号
   │
   ▼
[ARM64 异常向量表]
   arch/arm64/kernel/entry.S → el1_irq
   │
   ▼
[arch/arm64/kernel/irq.c]
   handle_arch_irq(regs) = gic_handle_irq(regs)
   │
   ▼
[gic_handle_irq]  ← GIC 驱动
   readl(GICC_INTACK) → 获取 irqnr
   writel(GICC_EOI)   → EOImode=0 提前 EOI
   │
   ▼
[generic_handle_domain_irq]
   irq_find_mapping(domain, irqnr) → virq
   │
   ▼
[generic_handle_irq_desc]
   desc->handle_irq() = handle_fasteoi_irq()  ← 由 gic_irq_domain_map 设置
   │
   ▼
[handle_fasteoi_irq]  ← kernel/irq/chip.c
   raw_spin_lock(&desc->lock)
   handle_irq_event(desc)
   │
   ▼
[handle_irq_event → __handle_irq_event_percpu]
   action->handler(irq, dev_id)   ← ★ 驱动注册的 top half
   │                                IRQ_WAKE_THREAD? → 唤醒 threaded handler
   ▼
[cond_unmask_eoi_irq]
   chip->irq_unmask / chip->irq_eoi
   raw_spin_unlock(&desc->lock)
   │
   ▼
[中断返回]
   el1_irq → eret (返回被中断的用户/内核代码)
```

---

## 二、CPU 异常入口

### 2.1 ARM64 中断向量

```asm
// arch/arm64/kernel/entry.S

// 中断向量表 (vbar_el1 寄存器指向)
.align 11
ENTRY(vectors)
    kernel_ventry   1, sync_invalid        // Synchronous EL1t
    kernel_ventry   1, irq_invalid         // IRQ EL1t
    kernel_ventry   1, fiq_invalid         // FIQ EL1t
    kernel_ventry   1, error_invalid       // Error EL1t

    kernel_ventry   1, sync               // Synchronous EL1h
    kernel_ventry   1, irq                // ★ IRQ EL1h ← 外设中断走这里
    kernel_ventry   1, fiq_invalid        // FIQ EL1h
    kernel_ventry   1, error_invalid      // Error EL1h
    ...

el1_irq:
    // 保存现场 (x0-x29, sp_el0, pc, pstate)
    // 进入 irq 上下文
    irq_handler
    // 恢复现场
    eret
```

### 2.2 `irq_handler` 宏

```asm
// 展开为:
.macro irq_handler
    ldr	x1, =handle_arch_irq      // 加载函数指针
    mov	x0, sp                     // pt_regs
    blr	x1                         // → gic_handle_irq(regs)
.endm
```

### 2.3 `set_handle_irq()`

```c
// kernel/irq/handle.c
void (*handle_arch_irq)(struct pt_regs *) = default_handle_irq;

int __init set_handle_irq(void (*handle_irq)(struct pt_regs *))
{
    handle_arch_irq = handle_irq;
}

// GIC 驱动初始化时:
set_handle_irq(gic_handle_irq);
```

---

## 三、GIC 层: 从中断号到 virq

### 3.1 `gic_handle_irq()`

```c
// drivers/irqchip/irq-gic.c:370
static void __exception_irq_entry gic_handle_irq(struct pt_regs *regs)
{
    do {
        // 1. ★ 读取 GICC_INTACK: 获取当前最高优先级中断
        //    IAR = Interrupt Acknowledge Register
        //    读取会"确认"中断 — GIC 将中断状态从 Pending→Active
        irqstat = readl_relaxed(cpu_base + GIC_CPU_INTACK);
        irqnr = irqstat & GICC_IAR_INT_ID_MASK;
        //    低 10 位是中断号 (0~1023)
        //    1023 = 无有效中断 (spurious)

        if (unlikely(irqnr >= 1020))
            break;

        // 2. EOImode=0: 提前写 EOI
        //    (EOImode=0 时, EOI 只是标记中断已被接收)
        if (supports_deactivate)
            writel_relaxed(irqstat, cpu_base + GIC_CPU_EOI);

        // 3. ★★ 分发到 Linux 中断子系统
        generic_handle_domain_irq(gic->domain, irqnr);
        //     ↑ 这是最重要的跳转
    } while (1);
}
```

### 3.2 `generic_handle_domain_irq()` 调用链

```c
// include/linux/irqdesc.h
static inline int generic_handle_domain_irq(struct irq_domain *domain,
                                             unsigned int hwirq)
{
    return generic_handle_irq(irq_find_mapping(domain, hwirq));
    //                    ↑                   ↑
    //  1. irq_find_mapping: hwirq → virq      |
    //     → linear lookup: domain->revmap[hwirq]  ───┘
    //     → virq = irq_data->irq
    //  2. generic_handle_irq(virq)
}

// kernel/irq/irqdesc.h (internal)
int generic_handle_irq(unsigned int irq)
{
    struct irq_desc *desc = irq_to_desc(irq);
    //     ↑ irq 号 → irq_desc  (全局数组 irq_desc[NR_IRQS])
    return generic_handle_irq_desc(desc);
}

static inline int generic_handle_irq_desc(struct irq_desc *desc)
{
    // ★★ 调用 desc->handle_irq
    //    这个函数指针在 irq_domain_associate 时设置
    //    GIC: handle_fasteoi_irq
    return desc->handle_irq(desc);
}
```

---

## 四、Flow Handler 层

### 4.1 三种主要 flow handler

| handler | 触发类型 | GIC 使用 | 特点 |
|---------|---------|----------|------|
| `handle_fasteoi_irq` | 电平 (Level) | ✅ SPI/PPI | 硬件 EOI, 不需要 ack |
| `handle_edge_irq` | 边沿 (Edge) | ❌ | 需要 ack + keep masked |
| `handle_level_irq` | 电平 (旧) | ❌ | 需要 mask + ack + unmask |
| `handle_simple_irq` | 软件触发 | ❌ | 无硬件操作 |

### 4.2 `handle_fasteoi_irq()` — GIC 默认 handler

```c
// kernel/irq/chip.c:689
void handle_fasteoi_irq(struct irq_desc *desc)
{
    struct irq_chip *chip = desc->irq_data.chip;

    raw_spin_lock(&desc->lock);

    // 1. 检查是否可以运行 (避免 spurious 和 reentrant)
    if (!irq_may_run(desc))
        goto out;

    desc->istate &= ~(IRQS_REPLAY | IRQS_WAITING);

    // 2. ★ 如果中断被禁用或没有处理函数 → mask 并退出
    if (unlikely(!desc->action || irqd_irq_disabled(&desc->irq_data))) {
        mask_irq(desc);               // GIC: gic_mask_irq
        desc->istate |= IRQS_PENDING;
        goto out;
    }

    kstat_incr_irqs_this_cpu(desc);   // 统计

    // 3. ONESHOT 中断: 先 mask (防止嵌套)
    if (desc->istate & IRQS_ONESHOT)
        mask_irq(desc);

    // 4. ★★ 调用用户注册的 handler
    handle_irq_event(desc);

    // 5. 条件 unmask + eoi
    cond_unmask_eoi_irq(desc, chip);
    //   → 非 ONESHOT + 非边缘触发: 直接写 EOI
    //   → ONESHOT: 在 thread 完成后才 unmask

    raw_spin_unlock(&desc->lock);
    return;

out:
    // 如果中断不可运行, 仍然要写 EOI (通知 GIC 处理完成)
    if (!(chip->flags & IRQCHIP_EOI_IF_HANDLED))
        chip->irq_eoi(&desc->irq_data);
    raw_spin_unlock(&desc->lock);
}
```

---

## 五、Action 分发层

### 5.1 `handle_irq_event()`

```c
// kernel/irq/handle.c:202
irqreturn_t handle_irq_event(struct irq_desc *desc)
{
    irqreturn_t ret;

    desc->istate &= ~IRQS_PENDING;
    irqd_set(&desc->irq_data, IRQD_IRQ_INPROGRESS);

    // ★ 解锁 desc->lock, 允许其他 CPU 在其他中断上操作
    raw_spin_unlock(&desc->lock);

    // ★★ 执行所有注册的 handler
    ret = handle_irq_event_percpu(desc);

    raw_spin_lock(&desc->lock);
    irqd_clear(&desc->irq_data, IRQD_IRQ_INPROGRESS);

    return ret;
}
```

### 5.2 `__handle_irq_event_percpu()` — 最终分发

```c
// kernel/irq/handle.c:139
irqreturn_t __handle_irq_event_percpu(struct irq_desc *desc)
{
    irqreturn_t retval = IRQ_NONE;
    struct irqaction *action;

    // ★ 遍历 action 链表 (共享中断可以多个 handler)
    for_each_action_of_desc(desc, action) {

        trace_irq_handler_entry(irq, action);

        // ★★★ 调用驱动的中断处理函数!
        res = action->handler(irq, action->dev_id);

        // 检查: handler 必须禁中断 (IRQF_DISABLED 已废弃)
        if (WARN_ONCE(!irqs_disabled(), ...))
            local_irq_disable();

        switch (res) {
        case IRQ_WAKE_THREAD:
            // ★ top half 返回 WAKE_THREAD → 唤醒 threaded handler
            if (unlikely(!action->thread_fn)) {
                warn_no_thread(irq, action);
                break;
            }
            __irq_wake_thread(desc, action);
            //   → desc->threads_online++
            //   → wake_up_process(action->thread)
            //   → kthread 开始执行 action->thread_fn(irq, dev_id)
            break;

        case IRQ_HANDLED:
            // 中断已处理
            break;

        case IRQ_NONE:
            // 不是本驱动的中断 (共享中断)
            break;
        }

        retval |= res;
    }

    return retval;
}
```

### 5.3 Threaded IRQ 机制

```c
// request_threaded_irq 注册时:
int request_threaded_irq(unsigned int irq, irq_handler_t handler,
                          irq_handler_t thread_fn, unsigned long flags,
                          const char *name, void *dev_id)
{
    // 1. 分配 action
    action = kzalloc(sizeof(*action), GFP_KERNEL);
    action->handler  = handler;     // top half (hardirq context)
    action->thread_fn = thread_fn;  // bottom half (process context)
    action->name      = name;
    action->dev_id    = dev_id;

    // 2. 如果 handler==NULL, 用 irq_default_primary_handler
    //    → 返回 IRQ_WAKE_THREAD, 立即唤醒 thread

    // 3. 注册
    __setup_irq(irq, action, flags, NULL);

    // ★ 4. 创建内核线程
    if (thread_fn) {
        action->thread = kthread_create(irq_thread, action, "irq/%d-%s", irq, name);
        // → `ps | grep irq` 可以看到: irq/77-i2c_rk3x
    }
}

// irq_thread — 内核线程主函数:
static int irq_thread(void *data)
{
    struct irqaction *action = data;

    while (!irq_wait_for_interrupt(action)) {
        // ★ 等待: 被 __irq_wake_thread() 唤醒
        //   → set_current_state(TASK_INTERRUPTIBLE)
        //   → schedule()
        //   → 被 wake_up_process() 唤醒

        // ★ 运行 bottom half:
        action->thread_fn(action->irq, action->dev_id);

        // ★ 通知完成:
        irq_finalize_oneshot(desc, action);
        //   → 如果 IRQF_ONESHOT, 这里才 unmask
    }
}
```

---

## 六、request_irq 注册流程

### 6.1 `request_irq()` → `request_threaded_irq()`

```c
// include/linux/interrupt.h
#define request_irq(irq, handler, flags, name, dev) \
    request_threaded_irq(irq, handler, NULL, flags, name, dev)
//                                    ↑  thread_fn = NULL → 纯 top half

// 或带 threaded 的:
request_threaded_irq(irq, my_top_half, my_bottom_half,
                     IRQF_TRIGGER_RISING, "mydev", dev);
```

### 6.2 `__setup_irq()` — 关键步骤

```c
// kernel/irq/manage.c
static int __setup_irq(unsigned int irq, struct irqaction *act,
                        unsigned long flags, struct irqaction **last)
{
    struct irq_desc *desc = irq_to_desc(irq);

    // 1. 检查中断是否可用 (未被 NMI 等占用)
    // 2. 检查共享中断兼容性

    // 3. 分配 thread (如果有 thread_fn)
    if (act->thread_fn && !act->thread) {
        act->thread = kthread_create(irq_thread, act, "irq/%d-%s", irq, act->name);
    }

    // 4. 挂入 action 链表 (desc->action 链表)
    old = desc->action;
    if (old) {
        // 共享中断: 追加到链表尾部
    } else {
        desc->action = act;     // 第一个 handler
    }

    // 5. ★ 如果是第一个 action, 调用 irq_chip 启动中断
    if (!old) {
        // 调用 irq_domain->ops->map 中设置的 startup
        // → chip->irq_startup(desc->irq_data)
        //   → gic_unmask_irq()  ← 使能 GICD_ENABLE_SET
        // → desc->handle_irq = handle_fasteoi_irq
    }

    // 6. 设置中断类型 (触发方式)
    if (act->flags & IRQF_TRIGGER_MASK) {
        ret = __irq_set_trigger(desc, act->flags & IRQF_TRIGGER_MASK);
        // → chip->irq_set_type = gic_set_type()
        //   写 GICD_CONFIG 寄存器
    }

    // 7. 激活线程
    if (act->thread)
        wake_up_process(act->thread);
}
```

---

## 七、完整路径总结

```
[外设]
   ↓ 硬件信号
GIC-400 Distributor + CPU Interface
   ↓ GICC_INTACK 读取
gic_handle_irq()                    ← irq-gic.c
   ↓ generic_handle_domain_irq()
generic_handle_irq(virq)            ← irqdesc.h
   ↓ irq_to_desc → handle_irq
handle_fasteoi_irq(desc)            ← chip.c
   ↓
handle_irq_event(desc)              ← handle.c
   ↓
__handle_irq_event_percpu(desc)
   ↓
action->handler(irq, dev_id)       ← ★ 你的 top half!
   │                        返回 IRQ_WAKE_THREAD?
   ↓ YES                          ↓ NO
action->thread_fn(irq, dev_id)    ← ★ 你的 bottom half!
```

---

## 八、思考题

1. **EOI 提前**：`gic_handle_irq()` 中调用 `generic_handle_domain_irq()` 之前就写 GICC_EOI。这意味着在驱动的 top half 还没开始执行时，GIC 就已经认为中断处理结束了。如果同一中断再次触发，会怎么样？

2. **共享中断的重入**：`__handle_irq_event_percpu()` 遍历 action 链表时，每个 handler 都调用完之后才退出。如果某个共享中断的 handler 返回 IRQ_NONE，内核会怎么处理？

3. **ONESHOT 标记**：`handle_fasteoi_irq()` 中如果 `IRQS_ONESHOT` 被设置，会在调用 handler 之前 mask 中断。`IRQF_ONESHOT` 标志通常用于什么场景？为什么 threaded handler 需要这个标志？

4. **中断嵌套**：`handle_irq_event()` 中 `raw_spin_unlock(&desc->lock)` 释放了锁，此时同一个中断能再次被触发吗？如果能，会发生什么？

---

## 相关笔记

- [[bsp-interrupt-concurrency]] — 阶段三：中断处理 + 并发
- [[bsp-interrupt-irqdomain-deep]] — IRQ Domain 源码
- [[bsp-interrupt-gic-deep]] — GIC-400 驱动源码
- [[MOC-嵌入式Linux]] — 嵌入式 Linux 学习路线 MOC
