---
tags:
  - embedded-linux
  - bsp
  - interrupt
  - gic
  - concurrency
  - spinlock
  - workqueue
  - rockchip
category: embedded-linux
created: 2026-06-22
updated: 2026-06-22
status: active
soc: Rockchip RV1126B
kernel: Linux 6.1.141
gic: ARM GIC-400 (GICv2)
---

# 阶段三：中断处理 + 并发

> **JD对标**：中断处理、稳定性提升
>
> 本章从 GIC 硬件中断控制器出发，理解 Linux 中断子系统的完整路径。核心是掌握 top half / bottom half 模型和并发原语，这是写稳定驱动的基础。

---

## 一、GIC-400 中断控制器

### 1.1 硬件架构

RV1126B 使用 ARM GIC-400 (GICv2 架构)：

```dts
/* rv1126b.dtsi */
gic: interrupt-controller@21201000 {
    compatible = "arm,gic-400";
    #interrupt-cells = <3>;
    interrupt-controller;
    reg = <0x21201000 0x1000>,    /* GICD: Distributor */
          <0x21202000 0x2000>,    /* GICC: CPU Interface */
          <0x21204000 0x2000>,    /* GICH: Virtualization */
          <0x21206000 0x2000>;    /* GICV: Virtual CPU Interface */
    interrupts = <GIC_PPI 9 (GIC_CPU_MASK_SIMPLE(4) | IRQ_TYPE_LEVEL_LOW)>;
};
```

### 1.2 GIC 内部结构

```
外设中断 (SPI 32~1019)
    │
    ▼
┌──────────────────────────┐
│    GICD (Distributor)     │
│  ├── 中断使能/禁用         │
│  ├── 优先级设置            │
│  ├── 中断分发到目标 CPU    │
│  └── Pending/Active 状态   │
└──────────┬───────────────┘
           │
    ┌──────┼──────┬──────┐
    ▼      ▼      ▼      ▼
  CPU0   CPU1   CPU2   CPU3
  GICC   GICC   GICC   GICC
```

| GIC 区域 | 基地址 | 作用 |
|---------|--------|------|
| GICD (Distributor) | 0x21201000 | 全局中断分发，控制 SPI/PPI 的使能、优先级、目标 CPU |
| GICC (CPU Interface) | 0x21202000 | 每个 CPU 独立，处理到达当前 CPU 的中断应答和 EOI |
| GICH (Hypervisor) | 0x21204000 | 虚拟化中断控制 (当前未使用) |
| GICV (Virtual CPU) | 0x21206000 | 虚拟 CPU 接口 (当前未使用) |

### 1.3 中断类型

| 类型 | 编号范围 | 说明 | RV1126B 示例 |
|------|---------|------|-------------|
| **SGI** (Software Generated) | 0~15 | 软件中断，CPU 间通信 (IPI) | `smp_call_function` |
| **PPI** (Private Peripheral) | 16~31 | 每 CPU 独有的私有中断 | CPU 本地定时器 (PPI 13/14)、GIC 维护中断 (PPI 9) |
| **SPI** (Shared Peripheral) | 32~1019 | 所有 CPU 共享的外设中断 | I2C/UART/SPI/GPIO 等所有外设 |

### 1.4 RV1126B 中断号分配

从 `rv1126b.dtsi` 提取的关键 SPI 中断号：

| 外设 | GIC SPI 编号 | 硬件中断号 (SPI+32) | 触发类型 |
|------|-------------|--------------------|---------|
| GPIO0 (pin 0-7) | 0~3 | 32~35 | LEVEL_HIGH |
| GPIO1 (pin 0-7) | 4~7 | 36~39 | LEVEL_HIGH |
| GPIO2 (pin 0-7) | 8~11 | 40~43 | LEVEL_HIGH |
| GPIO3 (pin 0-7) | 12~15 | 44~47 | LEVEL_HIGH |
| I2C0 | 48 | 80 | LEVEL_HIGH |
| I2C1 | 49 | 81 | LEVEL_HIGH |
| I2C2 | 50 | 82 | LEVEL_HIGH |
| I2C3 | 51 | 83 | LEVEL_HIGH |
| I2C4 | 52 | 84 | LEVEL_HIGH |
| I2C5 | 53 | 85 | LEVEL_HIGH |
| UART0 | 56 | 88 | LEVEL_HIGH |
| UART1 | 57 | 89 | LEVEL_HIGH |
| UART2~7 | 58~63 | 90~95 | LEVEL_HIGH |
| SPI0 | 192 | 224 | LEVEL_HIGH |
| SPI1 | 193 | 225 | LEVEL_HIGH |
| FIQ Debugger | 240 | 272 | LEVEL_HIGH |
| RTC | 215 | 247 | LEVEL_HIGH |
| USB2PHY | 32~35 | 64~67 | LEVEL_HIGH |

> **注意**：DTS 中写的是 `GIC_SPI 53`，表示 SPI 编号 53，实际硬件中断号 = 53 + 32 = 85。Linux 内核通过 irq_domain 映射为虚拟中断号 (virq)，开发者不直接接触硬件中断号。

### 1.5 中断号映射：hwirq → virq

```
硬件层:     GIC SPI 53 (I2C5)
                ↓ GIC driver irq_domain 翻译
内核层:     virq 53+32 = 85 (Linux 虚拟中断号)
                ↓ 驱动通过 platform_get_irq() 获取
驱动层:     irq = platform_get_irq(pdev, 0) → 返回 virq
                ↓ request_irq(irq, handler, ...)
注册:       handler 挂到 virq 上，中断到来时调用
```

---

## 二、Linux 中断子系统

### 2.1 中断处理路径

```
硬件中断到达 CPU
    ↓
异常向量 (arch/arm64/kernel/entry.S)
    ↓
gic_handle_irq()                    ← GIC 驱动入口
    ↓
handle_domain_irq()                 ← irq_domain 翻译
    ↓
generic_handle_irq_desc()           ← 通用中断处理
    ↓
handle_edge_irq / handle_level_irq  ← 按触发类型处理
    ↓
__handle_irq_event(desc)            ← 遍历 action 链表
    ↓
action->handler(irq, dev_id)        ← 调用你的中断处理函数
    ↓
  返回 IRQ_HANDLED / IRQ_NONE
```

### 2.2 request_irq vs request_threaded_irq

```c
/* 传统方式: 纯硬中断 (top half only) */
int request_irq(unsigned int irq, irq_handler_t handler,
                unsigned long flags, const char *name, void *dev);

/* 线程化中断: top half + bottom half (内核线程) */
int request_threaded_irq(unsigned int irq,
                         irq_handler_t handler,      /* top half */
                         irq_handler_t thread_fn,     /* bottom half (内核线程) */
                         unsigned long flags, const char *name, void *dev);
```

| 对比 | `request_irq` | `request_threaded_irq` |
|------|--------------|----------------------|
| handler 执行上下文 | 硬中断上下文 (原子) | top half 在硬中断, thread_fn 在内核线程 |
| 可否睡眠 | **绝对不能** | thread_fn 可以睡眠 |
| 可否执行耗时操作 | 不能 | 可以 (如 I2C 读取) |
| 延迟 | 最低 | 略高 (线程调度) |
| 适用场景 | 简单的清中断 + 唤醒 | 需要 I2C/SPI 通信的中断处理 |

### 2.3 IRQ 注册标志 (IRQF_*)

| 标志 | 说明 |
|------|------|
| `IRQF_SHARED` | 允许多个设备共享同一中断线 |
| `IRQF_TRIGGER_RISING` | 上升沿触发 |
| `IRQF_TRIGGER_FALLING` | 下降沿触发 |
| `IRQF_TRIGGER_HIGH` | 高电平触发 |
| `IRQF_TRIGGER_LOW` | 低电平触发 |
| `IRQF_ONESHOT` | 线程化中断中，硬中断结束后保持 IRQ 禁用直到线程完成 |
| `IRQF_NO_SUSPEND` | 系统休眠时不禁用此中断 |
| `IRQF_NO_THREAD` | 强制不线程化 (即使在 `PREEMPT_RT` 内核中) |
| `IRQF_NO_AUTOEN` | 注册后不自动使能，需手动 `enable_irq()` |

### 2.4 中断处理函数返回值

```c
irqreturn_t my_handler(int irq, void *dev_id)
{
    /* 处理中断... */
    return IRQ_HANDLED;     /* 中断已处理 */
    /* 或 */
    return IRQ_NONE;        /* 不是我的中断 (共享中断场景) */
    /* 或 */
    return IRQ_WAKE_THREAD; /* top half 返回，唤醒 thread_fn */
}
```

---

## 三、Top Half vs Bottom Half

### 3.1 为什么要分层

```
中断上下文 (top half):
  - 硬件中断关闭 (或当前中断线禁用)
  - 不能睡眠、不能调度
  - 必须尽快完成
  - 限制: 不能 mutex_lock、不能 kmalloc(GFP_KERNEL)、不能 copy_to_user

进程上下文 (bottom half):
  - 中断已重新使能
  - 可以睡眠、可以调度
  - 允许耗时操作
  - 可以 mutex_lock、kmalloc、I2C/SPI 通信
```

### 3.2 Bottom Half 三种机制

| 机制 | 执行上下文 | 可否睡眠 | 适用场景 | 推荐度 |
|------|-----------|---------|---------|--------|
| **workqueue** | 内核线程 (进程上下文) | 可以 | 需要睡眠/IO操作的处理 | 首选 |
| **threaded_irq** | 专用内核线程 | 可以 | 中断处理本身就耗时 | 推荐 |
| **tasklet** | 软中断 (原子上下文) | 不能 | 轻量级、不睡眠的延后处理 | 逐渐弃用 |

### 3.3 Workqueue 使用

```c
#include <linux/workqueue.h>

struct my_data {
    struct work_struct work;
    int irq_count;
};

/* bottom half 函数 (进程上下文, 可睡眠) */
static void my_work_handler(struct work_struct *work)
{
    struct my_data *data = container_of(work, struct my_data, work);

    /* 可以执行耗时操作: I2C 读取、mutex、kmalloc 等 */
    pr_info("processing interrupt in workqueue, count=%d\n", data->irq_count);
}

/* top half: 硬中断处理 (原子上下文, 不可睡眠) */
static irqreturn_t my_irq_handler(int irq, void *dev_id)
{
    struct my_data *data = dev_id;

    /* 1. 清中断 (读写硬件寄存器) */
    // iowrite32(0x01, base + IRQ_CLEAR_REG);

    /* 2. 调度 workqueue 做后续处理 */
    data->irq_count++;
    schedule_work(&data->work);

    return IRQ_HANDLED;
}

/* probe 中初始化 */
static int my_probe(struct platform_device *pdev)
{
    struct my_data *data;

    data = devm_kzalloc(&pdev->dev, sizeof(*data), GFP_KERNEL);
    INIT_WORK(&data->work, my_work_handler);

    return devm_request_irq(&pdev->dev, irq, my_irq_handler,
                            IRQF_TRIGGER_FALLING, "my-device", data);
}
```

### 3.4 Threaded IRQ 使用

```c
/* top half: 极简, 只做紧急判断 */
static irqreturn_t my_irq_top(int irq, void *dev_id)
{
    struct my_data *data = dev_id;

    /* 快速检查是否是我们的中断 */
    if (!(ioread32(data->base + IRQ_STATUS) & data->irq_mask))
        return IRQ_NONE;

    /* 清中断标志 */
    iowrite32(data->irq_mask, data->base + IRQ_CLEAR);

    /* 告诉内核: 唤醒线程处理 */
    return IRQ_WAKE_THREAD;
}

/* bottom half: 内核线程, 可睡眠 */
static irqreturn_t my_irq_thread(int irq, void *dev_id)
{
    struct my_data *data = dev_id;

    /* 可以执行 I2C 读取、mutex、sleep 等 */
    // i2c_transfer(data->client->adapter, &msg, 1);

    pr_info("threaded irq processed\n");
    return IRQ_HANDLED;
}

/* 注册 */
devm_request_threaded_irq(&pdev->dev, irq,
                          my_irq_top, my_irq_thread,
                          IRQF_TRIGGER_FALLING | IRQF_ONESHOT,
                          "my-device", data);
```

> **IRQF_ONESHOT 的作用**：在 top half 返回后、thread_fn 执行完毕前，保持中断线禁用。防止同一中断在 thread_fn 执行期间再次触发 top half，导致竞态。

---

## 四、并发原语

### 4.1 中断上下文 vs 进程上下文

| 上下文 | 如何进入 | 可否睡眠 | 可否调度 | 可否持有 mutex |
|--------|---------|---------|---------|--------------|
| 进程上下文 | 系统调用 / 内核线程 | 可以 | 可以 | 可以 |
| 软中断上下文 | tasklet / softirq | 不能 | 不能 | 不能 |
| 硬中断上下文 | IRQ handler | 不能 | 不能 | 不能 |

### 4.2 锁选择决策树

```
需要保护数据吗?
  └─ 是
     ├─ 中断上下文会访问吗?
     │   ├─ 是 → spinlock_irqsave (禁止中断 + 自旋锁)
     │   └─ 否
     │       ├─ 软中断/tasklet 会访问吗?
     │       │   ├─ 是 → spinlock_bh (禁止软中断 + 自旋锁)
     │       │   └─ 否 → mutex (可睡眠, 进程上下文专用)
     │       └─
     └─ 否 → 不需要锁
```

### 4.3 常用并发原语对比

| 原语 | 上下文要求 | 可否睡眠 | 开销 | 适用场景 |
|------|-----------|---------|------|---------|
| `spinlock_t` | 进程+中断 | 不能 | 最低 | 短临界区, 中断安全 |
| `spinlock_irqsave` | 进程+硬中断 | 不能 | 低 | 硬中断可能并发访问的数据 |
| `spinlock_bh` | 进程+软中断 | 不能 | 低 | tasklet/softirq 并发 |
| `struct mutex` | 仅进程 | 可以 | 中 | 大临界区, 可睡眠 |
| `struct rwlock_t` | 进程+中断 | 不能 | 低 | 读多写少 |
| `struct rw_semaphore` | 仅进程 | 可以 | 中 | 读多写少, 可睡眠 |
| `atomic_t` | 任意 | N/A | 最低 | 单个整数操作 |
| `struct completion` | 仅进程 | 可以 | 中 | 等待条件满足 (如 IRQ 完成) |

### 4.4 spinlock_irqsave 用法 (最常用)

```c
#include <linux/spinlock.h>

struct my_data {
    spinlock_t lock;
    int counter;
};

/* 进程上下文 + 中断上下文都可能访问 counter */
static void increment_counter(struct my_data *data)
{
    unsigned long flags;

    /* 保存当前中断状态 → 禁用中断 → 获取锁 */
    spin_lock_irqsave(&data->lock, flags);
    data->counter++;
    /* 释放锁 → 恢复中断状态 */
    spin_unlock_irqrestore(&data->lock, flags);
}

/* 如果确定当前不在中断上下文, 可以用更轻量的版本 */
static void increment_counter_process(struct my_data *data)
{
    spin_lock_bh(&data->lock);    /* 只禁软中断 */
    data->counter++;
    spin_unlock_bh(&data->lock);
}
```

### 4.5 mutex 用法

```c
#include <linux/mutex.h>

struct my_data {
    struct mutex mutex;
    struct expensive_struct *cache;
};

/* 进程上下文, 可以睡眠 (如等待 I2C) */
static struct expensive_struct *get_cache(struct my_data *data)
{
    mutex_lock(&data->mutex);
    if (!data->cache) {
        /* 可以在这里睡眠 (如 kmalloc(GFP_KERNEL) 或 I2C 读取) */
        data->cache = kmalloc(sizeof(*data->cache), GFP_KERNEL);
        populate_cache(data->cache);
    }
    mutex_unlock(&data->mutex);
    return data->cache;
}
```

### 4.6 completion 用法 (中断同步)

```c
#include <linux/completion.h>

struct my_data {
    struct completion done;
    void __iomem *base;
};

/* 进程上下文: 等待中断完成 */
static int wait_for_irq(struct my_data *data)
{
    /* 启动硬件操作 */
    iowrite32(START_CMD, data->base + CTRL);

    /* 等待中断通知完成, 最多等 1 秒 */
    if (!wait_for_completion_timeout(&data->done, msecs_to_jiffies(1000))) {
        dev_err(data->dev, "timeout waiting for IRQ\n");
        return -ETIMEDOUT;
    }
    return 0;
}

/* 中断上下文: 通知完成 */
static irqreturn_t my_irq_handler(int irq, void *dev_id)
{
    struct my_data *data = dev_id;
    complete(&data->done);     /* 唤醒等待者 */
    return IRQ_HANDLED;
}

/* probe 中初始化 */
init_completion(&data->done);
```

---

## 五、IRQ Domain 与中断级联

### 5.1 为什么需要 IRQ Domain

```
硬件层: GIC SPI 53 (硬件中断号)
    ↓ GIC 的 irq_domain 翻译
Linux 层: virq = 85 (Linux 虚拟中断号)
    ↓ 驱动通过 platform_get_irq() 获取
驱动层: request_irq(virq, handler, ...)
```

IRQ Domain 解决的问题：
- 多级中断控制器 (GIC → GPIO controller → 外设)
- 硬件中断号和 Linux 虚拟中断号分离
- 支持设备树中断描述符翻译

### 5.2 GPIO 中断级联

```
外设 (如按键)
  ↓ GPIO pin 电平变化
GPIO Controller (gpio0~7, 每组是一个 irq_domain)
  ↓ 级联到
GIC-400 (根 irq_domain)
  ↓ 送到 CPU
Linux 中断处理
```

```dts
/* gpio0 节点: 既是 GPIO 控制器, 又是中断控制器 */
gpio0: gpio@20600000 {
    compatible = "rockchip,gpio-bank";
    reg = <0x20600000 0x200>;
    interrupts = <GIC_SPI 0 IRQ_TYPE_LEVEL_HIGH>,   /* 级联到 GIC SPI 0 */
                 <GIC_SPI 1 IRQ_TYPE_LEVEL_HIGH>,
                 <GIC_SPI 2 IRQ_TYPE_LEVEL_HIGH>,
                 <GIC_SPI 3 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-controller;     /* 声明自己是中断控制器 */
    #interrupt-cells = <2>;   /* 子节点: <pin_number, trigger_type> */
};

/* 外设节点引用 GPIO 中断 */
my-button {
    compatible = "my,gpio-button";
    interrupt-parent = <&gpio0>;          /* 指向 GPIO0 控制器 */
    interrupts = <3 IRQ_TYPE_EDGE_FALLING>; /* GPIO0 pin 3, 下降沿 */
};
```

### 5.3 板端验证

```bash
# 查看中断号映射
cat /proc/interrupts
# 输出示例:
#            CPU0       CPU1       CPU2       CPU3
#  16:          0          0          0          0     GICv2  25 Level     arch_timer
#  85:        128          0          0          0     GICv2  53 Level     21140000.i2c
#  88:       1024          0          0          0     GICv2  56 Level     ttyFIQ0

# 查看中断控制器层级
cat /sys/kernel/debug/irq/irq_domains
# 预期: domain hierarchy (GIC → GPIO)

# 查看具体中断信息
cat /sys/kernel/debug/irq/irqs/85
# 预期: I2C5 的中断详细信息
```

---

## 六、实验 1：解析 GIC 中断分配

### 6.1 实验目标

从 DTS 和板端 `/proc/interrupts` 对照分析 RV1126B 的完整中断分配。

### 6.2 操作步骤

```bash
# 板端:
cat /proc/interrupts
# 记录每个中断的: virq, 硬件中断号, 触发类型, 设备名, 中断计数

# 查看当前中断亲和性 (哪些 CPU 在处理)
cat /proc/irq/85/smp_affinity
# 预期: f (所有 4 个 CPU)

# 统计各设备中断计数
cat /proc/interrupts | awk 'NR>1{print $NF, $(NF-1)}' | sort | head -20
```

### 6.3 分析问题

1. 哪个设备的中断计数最高？为什么？
2. `ttyFIQ0` (串口控制台) 使用什么类型的中断？为什么不用标准 SPI？
3. GPIO 中断是如何级联到 GIC 的？

---

## 七、实验 2：注册 GPIO 中断 + workqueue

### 7.1 实验目标

编写驱动注册一个 GPIO 中断（按键或模拟），在 top half 中清中断，在 workqueue 中做延时处理。

### 7.2 设备树

```dts
/* 在 rv1126b-sportcam.dts 中添加 */
/ {
    my_irq_device: my-irq-device {
        compatible = "my,irq-demo";
        interrupt-parent = <&gpio0>;
        interrupts = <RK_PA0 IRQ_TYPE_EDGE_FALLING>;
        status = "okay";
    };
};
```

### 7.3 驱动源码 (irq_demo.c)

```c
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/of.h>
#include <linux/of_irq.h>
#include <linux/interrupt.h>
#include <linux/workqueue.h>

struct irq_demo_data {
    int irq;
    struct work_struct work;
    atomic_t irq_count;
};

static void irq_demo_work(struct work_struct *work)
{
    struct irq_demo_data *data =
        container_of(work, struct irq_demo_data, work);
    int count = atomic_inc_return(&data->irq_count);

    /* 进程上下文, 可以睡眠 */
    pr_info("irq_demo: processing in workqueue, count=%d\n", count);
    msleep(10);
    pr_info("irq_demo: workqueue done\n");
}

static irqreturn_t irq_demo_handler(int irq, void *dev_id)
{
    struct irq_demo_data *data = dev_id;

    /* top half: 极简, 只调度 work */
    pr_info("irq_demo: IRQ triggered!\n");
    schedule_work(&data->work);

    return IRQ_HANDLED;
}

static int irq_demo_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct irq_demo_data *data;
    int ret;

    data = devm_kzalloc(dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;

    data->irq = platform_get_irq(pdev, 0);
    if (data->irq < 0) {
        dev_err(dev, "failed to get IRQ: %d\n", data->irq);
        return data->irq;
    }
    dev_info(dev, "got IRQ %d\n", data->irq);

    INIT_WORK(&data->work, irq_demo_work);
    atomic_set(&data->irq_count, 0);

    ret = devm_request_irq(dev, data->irq, irq_demo_handler,
                           IRQF_TRIGGER_FALLING, "irq-demo", data);
    if (ret) {
        dev_err(dev, "failed to request IRQ: %d\n", ret);
        return ret;
    }

    platform_set_drvdata(pdev, data);
    dev_info(dev, "irq demo driver loaded, IRQ=%d\n", data->irq);
    return 0;
}

static int irq_demo_remove(struct platform_device *pdev)
{
    struct irq_demo_data *data = platform_get_drvdata(pdev);
    cancel_work_sync(&data->work);
    dev_info(&pdev->dev, "irq demo driver removed\n");
    return 0;
}

static const struct of_device_id irq_demo_match[] = {
    { .compatible = "my,irq-demo" },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, irq_demo_match);

static struct platform_driver irq_demo_driver = {
    .probe  = irq_demo_probe,
    .remove = irq_demo_remove,
    .driver = {
        .name = "irq-demo",
        .of_match_table = of_match_ptr(irq_demo_match),
    },
};

module_platform_driver(irq_demo_driver);
MODULE_LICENSE("GPL");
```

### 7.4 编译 & 测试

```bash
# 编译 (使用阶段二的 Makefile 模板)
make

# 部署
scp irq_demo.ko rooter@192.168.1.109:/tmp/

# 板端
sudo insmod /tmp/irq_demo.ko
# 预期: "irq demo driver loaded, IRQ=XXX"

# 触发中断 (用导线短接 GPIO 引脚到 GND, 或按板上的按键)
# 预期 dmesg:
#   "irq_demo: IRQ triggered!"
#   "irq_demo: processing in workqueue, count=1"
#   "irq_demo: workqueue done"

# 查看中断计数
cat /proc/interrupts | grep irq-demo

# 卸载
sudo rmmod irq_demo
```

---

## 八、实验 3：irqsoff tracer 测量中断关闭时长

### 8.1 实验目标

用 Ftrace `irqsoff` tracer 测量中断被禁用的最大时长，评估驱动对系统实时性的影响。

### 8.2 操作步骤

```bash
# 板端:
# 启用 irqsoff tracer (追踪中断关闭最长时间)
echo irqsoff | sudo tee /sys/kernel/tracing/current_tracer

# 查看当前最大中断关闭时长
cat /sys/kernel/tracing/tracing_max_latency
# 预期: 初始值很小 (~100us)

# 运行一些操作 (加载驱动、读写文件)
sudo insmod /tmp/irq_demo.ko
dd if=/dev/urandom of=/tmp/test bs=1M count=10

# 重新查看最大延迟
cat /sys/kernel/tracing/tracing_max_latency
# 预期: 可能增加到几百 us

# 查看延迟发生的函数调用
cat /sys/kernel/tracing/trace
# 预期: 显示中断关闭时的函数调用链和延迟柱状图

# 重置
echo 0 | sudo tee /sys/kernel/tracing/tracing_max_latency

# 关闭
echo nop | sudo tee /sys/kernel/tracing/current_tracer
```

### 8.3 预期输出格式

```
# tracer: irqsoff
# irqsoff latency trace v1.1.5
# latency: 235 us
# (函数调用链, 显示哪些函数导致中断长时间关闭)
# 例如: _raw_spin_lock_irqsave → ... → spin_unlock_irqrestore
```

---

## 九、实验 4：对比 request_irq vs request_threaded_irq

### 9.1 实验目标

同一个 GPIO 中断，分别用 `request_irq` 和 `request_threaded_irq` 注册，用 Ftrace 测量两种方式的延迟差异。

### 9.2 实验方法

```bash
# 1. 用 request_irq 版本注册中断
#    在 handler 中只做: iowrite32(清中断) + complete()
#    在进程上下文中测量 complete() 到来的时间

# 2. 用 request_threaded_irq 版本注册中断
#    top half: 清中断 + return IRQ_WAKE_THREAD
#    thread_fn: complete()
#    在进程上下文中测量 complete() 到来的时间

# 3. 用 Ftrace function_graph 追踪两者
echo function_graph | sudo tee /sys/kernel/tracing/current_tracer
echo '*handle_irq*|*thread_fn*|*complete*' | sudo tee /sys/kernel/tracing/set_ftrace_filter

# 4. 对比延迟
# 预期: request_irq 延迟 < 10us
# 预期: request_threaded_irq 延迟 ~50-200us (线程调度开销)
```

### 9.3 延迟对比表

| 方式 | 延迟 (估算) | 原因 |
|------|------------|------|
| `request_irq` | < 10us | 直接在硬中断上下文执行 |
| `request_threaded_irq` | 50~200us | 需要唤醒内核线程 + 调度 |
| `workqueue (schedule_work)` | 100~500us | 加入系统工作队列 + 调度 |

> **结论**：如果中断处理只需要清中断 + 通知，用 `request_irq`。如果需要 I2C/SPI 通信等耗时操作，用 `request_threaded_irq`。

---

## 十、实验 5：Lockdep 捕获并发 bug

### 10.1 实验目标

故意在驱动中制造一个死锁场景，用 Lockdep 捕获。

### 10.2 故意制造死锁

```c
/* 错误示例: 在持有 spinlock 时调用可能睡眠的函数 */
static void buggy_handler(struct work_struct *work)
{
    struct my_data *data = container_of(work, struct my_data, work);
    unsigned long flags;

    spin_lock_irqsave(&data->lock, flags);

    /* BUG: mutex_lock 可能睡眠, 在 spinlock 中不允许 */
    mutex_lock(&data->mutex);   /* ← Lockdep 会在这里报错 */

    /* ... */

    mutex_unlock(&data->mutex);
    spin_unlock_irqrestore(&data->lock, flags);
}
```

### 10.3 Lockdep 输出

```bash
# 确保 debug 内核已启用 Lockdep
zcat /proc/config.gz | grep LOCKDEP
# 预期: CONFIG_LOCKDEP=y

# 加载有 bug 的驱动
sudo insmod /tmp/buggy_driver.ko

# dmesg 中预期看到:
# ============================================
# WARNING: possible recursive locking detected
# ...
#     may be causing this lockup.
# ============================================
```

### 10.4 正确做法

```c
/* 正确: 先释放 spinlock, 再获取 mutex */
static void correct_handler(struct work_struct *work)
{
    struct my_data *data = container_of(work, struct my_data, work);
    unsigned long flags;

    spin_lock_irqsave(&data->lock, flags);
    data->counter++;
    spin_unlock_irqrestore(&data->lock, flags);

    /* spinlock 释放后, 可以安全使用 mutex */
    mutex_lock(&data->mutex);
    /* ... */
    mutex_unlock(&data->mutex);
}
```

---

## 十一、思考题

1. GIC-400 中 SPI 中断 53 (I2C5) 到达 GICD 后，GICD 如何决定发给哪个 CPU？如果 4 个 CPU 都空闲，中断会被发给哪个？如何改变这个行为？

2. `request_threaded_irq` 中 `IRQF_ONESHOT` 的作用是"在 thread_fn 执行完毕前保持中断线禁用"。如果不加这个标志，什么场景下会出问题？

3. 在硬中断处理函数 (top half) 中，以下哪些操作是安全的，哪些会导致崩溃？
   - `ioread32()` / `iowrite32()`
   - `spin_lock_irqsave()`
   - `mutex_lock()`
   - `kmalloc(GFP_ATOMIC)`
   - `kmalloc(GFP_KERNEL)`
   - `schedule_work()`
   - `complete()`
   - `printk()`
   - `copy_to_user()`

4. `spinlock_irqsave` 和 `spinlock_bh` 的区别是什么？什么场景下用前者，什么场景下用后者？

5. 如果一个 GPIO 中断的处理需要通过 I2C 读取传感器数据（每次约 2ms），你会选择 workqueue 还是 threaded_irq？为什么？如果用户要求中断响应延迟不超过 1ms，你的方案是否满足？

---

## 十二、踩坑记录

| 日期 | 问题 | 原因 | 解决方案 |
|------|------|------|----------|
| | request_irq 返回 -EBUSY | 中断号已被占用 | `cat /proc/interrupts` 检查冲突，或加 `IRQF_SHARED` |
| | 中断 handler 不被调用 | DTS 中 interrupt-parent 指向错误 | 确认 GPIO 控制器引用正确 |
| | 中断频繁触发导致系统卡死 | 边沿中断未清中断标志 | 在 handler 中清硬件中断标志 |
| | workqueue handler 中 mutex 死锁 | workqueue 中持锁后调用可能阻塞的函数 | 缩小锁范围，避免在锁内睡眠 |
| | threaded_irq 的 thread_fn 不执行 | top half 未返回 IRQ_WAKE_THREAD | 检查 top half 返回值 |
| | Lockdep 报 "possible deadlock" | 嵌套锁顺序不一致 | 统一锁获取顺序 |

---

## 十三、下阶段预告

阶段四：**外设驱动 — I2C + SPI + UART**
- I2C 子系统：adapter (`i2c-rk3x.c`) + client driver + `i2c_transfer`
- SPI 子系统：master (`spi-rockchip.c`) + device driver + `spi_sync`
- UART 子系统：8250/DesignWare APB UART (`8250_dw.c`)
- 写完整的 I2C client driver 读取 RK801 PMIC 寄存器
- Ftrace 追踪 `i2c_transfer` 完整调用链

---

## 相关笔记

- [[MOC-嵌入式Linux]] — 学习路线总览
- [[bsp-boot-flow]] — 阶段一：Bootloader + 启动流程
- [[bsp-device-model-dtb]] — 阶段二：设备模型 + 设备树
- [[bsp-interrupt-irqdomain-deep]] — IRQ Domain 源码深挖
- [[bsp-interrupt-gic-deep]] — GIC-400 驱动源码深挖
- [[bsp-interrupt-fullpath-deep]] — 中断处理完整路径
- [[bsp-lockdep-deep]] — Lockdep 原理与源码
- [[kernel-debug-env]] — 附录A：内核调试环境 (Ftrace / Lockdep 使用)
