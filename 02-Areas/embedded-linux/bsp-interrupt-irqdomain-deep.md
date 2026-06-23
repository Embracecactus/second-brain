---
tags:
  - embedded-linux
  - bsp
  - interrupt
  - irq-domain
  - irqchip
  - gic
  - source-code
  - rockchip
category: embedded-linux
created: 2026-06-23
updated: 2026-06-23
status: active
soc: Rockchip RV1126B
kernel: Linux 6.1.141
parent: bsp-interrupt-concurrency
---

# IRQ Domain 源码深挖 — hwirq → virq 映射

> **前置笔记**：[[bsp-interrupt-concurrency]] — 阶段三：中断处理 + 并发
>
> **核心文件**：`kernel/irq/irqdomain.c`, `include/linux/irqdomain.h`, `drivers/of/irq.c`

---

## 一、为什么需要 IRQ Domain？

### 1.1 问题

```
硬件中断号 (hwirq) vs Linux 中断号 (virq):

[硬件层]
  ┌──────────┐     ┌───────────┐     ┌──────────┐
  │ GIC-400  │     │ GPIO0     │     │ GPIO1    │
  │ hwirq 0~ │     │ hwirq 0~  │     │ hwirq 0~ │
  │     159  │     │      31   │     │      31  │
  └──────────┘     └───────────┘     └──────────┘
       │                    │               │
       ▼                    ▼               ▼
  ┌─────────────────────────────────────────────┐
  │ Linux 中断号空间 (virq)                      │
  │ NR_IRQS = 256 (平台相关)                     │
  │                                             │
  │ 问题: GIC SPI#45 = ? Linux 中断号?           │
  │       GPIO0#5   = ? Linux 中断号?            │
  │       GPIO1#12  = ? Linux 中断号?            │
  └─────────────────────────────────────────────┘

解决: IRQ Domain = hwirq → virq 映射层
```

---

## 二、核心数据结构

### 2.1 `struct irq_domain`

```c
// include/linux/irqdomain.h:164
struct irq_domain {
    struct list_head link;               // 全局 domain 链表
    const char *name;                    // 域名 (如 "GIC-400")
    const struct irq_domain_ops *ops;    // ★ 操作函数表
    void *host_data;                     // 控制器私有数据 (GIC 用)
    unsigned int flags;

    struct fwnode_handle *fwnode;        // ★ 指向 DT 节点
    enum irq_domain_bus_token bus_token;

    struct irq_domain *parent;           // ★ 层级 domain 的父节点

    /* 三种反向映射方式 (从 hwirq → virq) */
    irq_hw_number_t hwirq_max;           // 最大 hwirq 值
    unsigned int revmap_size;            // 线性映射大小
    struct radix_tree_root revmap_tree;   // 基数树 (非线性映射)
    struct irq_data __rcu *revmap[];     // 线性映射数组
};
```

### 2.2 `struct irq_domain_ops`

```c
// include/linux/irqdomain.h:107
struct irq_domain_ops {
    int (*match)(struct irq_domain *d, struct device_node *node,
                 enum irq_domain_bus_token bus_token);
    // 检查 domain 是否对应某个 DT 节点

    int (*map)(struct irq_domain *d, unsigned int virq,
               irq_hw_number_t hw);
    // ★ 映射创建时调用, 设置 irq_chip 和 handler

    void (*unmap)(struct irq_domain *d, unsigned int virq);

    int (*xlate)(struct irq_domain *d, struct device_node *node,
                 const u32 *intspec, unsigned int intsize,
                 unsigned long *out_hwirq, unsigned int *out_type);
    // ★ 解析 DTS interrupts 属性为 hwirq+type
};
```

### 2.3 三种反向映射

```c
// irq_domain 有三种映射方式, 通过 flags 区分:

// 1. 线性映射 (Linear Revmap)
//    flags |= 0
//    revmap[] 数组, size = hwirq_max
//    以 hwirq 为索引, O(1) 查找
//    适合 hwirq 数量少且连续的场景 (如 GIC, hwirq=0~159)
//    分配: irq_domain_create_linear()

// 2. 基数树映射 (Radix Tree Revmap)
//    flags |= IRQ_DOMAIN_FLAG_NO_MAP
//    revmap_tree 基数树
//    适合 hwirq 数量大或稀疏的场景 (如 GPIO, hwirq=0~1000+)
//    分配: irq_domain_create_tree()

// 3. 无映射 (No Map / Nomap)
//    hwirq == virq 直接相等
//    flags |= IRQ_DOMAIN_FLAG_NOMAP
//    适合实际硬件中断号就是 Linux 中断号的场景 (少见)

// ★ 查找入口:
struct irq_data *irq_domain_get_irq_data(struct irq_domain *domain,
                                          irq_hw_number_t hwirq)
{
    if (hwirq < domain->revmap_size)
        // 线性查找
        return domain->revmap[hwirq];
    else
        // 基数树查找
        return radix_tree_lookup(&domain->revmap_tree, hwirq);
}
```

---

## 三、IRQ Domain 生命周期

### 3.1 创建

```c
// kernel/irq/irqdomain.c:245
struct irq_domain *__irq_domain_add(struct fwnode_handle *fwnode,
                                     unsigned int size,     // revmap_size
                                     irq_hw_number_t hwirq_max,
                                     int direct_max,
                                     const struct irq_domain_ops *ops,
                                     void *host_data)
{
    domain = kzalloc(struct_size(domain, revmap, size), GFP_KERNEL);
    domain->fwnode    = fwnode;
    domain->ops       = ops;
    domain->host_data = host_data;
    domain->hwirq_max = hwirq_max;
    domain->revmap_size = size;

    mutex_init(&domain->revmap_mutex);
    INIT_RADIX_TREE(&domain->revmap_tree, GFP_KERNEL);

    // 加入全局链表
    list_add(&domain->link, &irq_domain_list);
}

// GIC-400 创建 domain:
// drivers/irqchip/irq-gic.c
static int __init gic_of_init(struct device_node *node, ...)
{
    gic = kzalloc(sizeof(*gic), GFP_KERNEL);
    gic->domain = __irq_domain_add(of_node_to_fwnode(node),
                                    NR_GIC_SPI_IRQS,  // size = 160
                                    1024,              // hwirq_max
                                    0,
                                    &gic_irq_domain_ops,  // ★ GIC ops
                                    gic);
}
```

### 3.2 匹配 (DTS 中断控制器解析)

```c
// 从 DTS interrupts 属性找到对应的 irq_domain:

// 1. of_irq_parse_one()
//    drivers/of/irq.c:358
//    解析: interrupts = <GIC_SPI 45 IRQ_TYPE_LEVEL_HIGH>
//    输出: of_phandle_args {
//              np = &gic;           // 指向 interrupt-controller 节点
//              args = [0, 45, 4];   // GIC_SPI=0, hwirq=45, type=4
//          }

// 2. irq_find_matching_fwspec()
//    kernel/irq/irqdomain.c:423
//    遍历 irq_domain_list, 对每个 domain 调用:
//      domain->ops->match(domain, np, bus_token)
//    → GIC: gic_irq_domain_match() → 检查 np == gic->dev_node
//    → 找到正确的 irq_domain

// 3. irq_create_of_mapping()
//    kernel/irq/irqdomain.c:905
//    调用 domain->ops->xlate() 或 translate()
//    → GIC: gic_irq_domain_xlate()
//    → 将 intspec=[GIC_SPI, 45, 4]
//       → out_hwirq = 32 + 45 = 77  (SPI 偏移)
//       → out_type  = IRQ_TYPE_LEVEL_HIGH
```

### 3.3 映射创建

```c
// kernel/irq/irqdomain.c:745
unsigned int irq_create_mapping_affinity(struct irq_domain *domain,
                                          irq_hw_number_t hwirq, ...)
{
    // 1. 检查是否已存在映射
    virq = irq_find_mapping(domain, hwirq);
    if (virq)
        return virq;

    // 2. ★ 从空闲池中分配一个 virq
    virq = irq_domain_alloc_descs(-1, 1, hwirq, nid, affinity);
    //    → alloc irq_desc (struct irq_desc 是每个 virq 的描述符)
    //    → 核心 IRQ 号分配!

    // 3. 关联 hwirq ↔ virq
    if (irq_domain_associate_locked(domain, virq, hwirq)) {
        irq_free_desc(virq);
        return 0;
    }

    return virq;
}

// irq_domain_associate_locked() 核心:
static int irq_domain_associate_locked(struct irq_domain *domain,
                                        unsigned int virq,
                                        irq_hw_number_t hwirq)
{
    struct irq_data *irq_data = irq_get_irq_data(virq);

    irq_data->hwirq  = hwirq;                // 设置硬件号
    irq_data->domain = domain;               // 设置所属 domain

    if (domain->ops->map)
        domain->ops->map(domain, virq, hwirq);
        // → GIC: gic_irq_domain_map()
        //   → irq_set_chip_data(virq, gic);
        //   → irq_set_chip_and_handler(virq, &gic_chip, handle_fasteoi_irq);

    // ★ 存入反向映射表
    irq_domain_set_mapping(domain, hwirq, irq_data);
    // → domain->revmap[hwirq] = irq_data   // 线性
    //   或 radix_tree_insert()               // 基数树
}
```

---

## 四、GIC-400 的 IRQ Domain 实例

### 4.1 GIC domain ops

```c
// drivers/irqchip/irq-gic.c
static const struct irq_domain_ops gic_irq_domain_ops = {
    .match  = gic_irq_domain_match,    // 匹配 DT 节点
    .map    = gic_irq_domain_map,      // 设置 irq_chip + handler
    .xlate  = gic_irq_domain_xlate,    // 解析 interrupts 属性
};

static int gic_irq_domain_map(struct irq_domain *d, unsigned int virq,
                               irq_hw_number_t hwirq)
{
    struct gic_chip_data *gic = d->host_data;

    // 根据 hwirq 类型设置不同 chip 和 handler:
    if (hwirq < 32) {
        // PPI (Per CPU Interrupt): 每个 CPU 私有
        irq_set_percpu_devid(virq);
        irq_set_chip_and_handler(virq, &gic_chip, handle_percpu_devid_irq);
    } else if (hwirq < GIC_PRIVATE_SIGNALS(32)) {  // SPI
        // SPI (Shared Peripheral Interrupt): 共享中断
        irq_set_chip_data(virq, gic);
        irq_set_chip_and_handler(virq, &gic_chip, handle_fasteoi_irq);
        //                             ↑ 使用 fasteoi 处理模型
    }

    irq_modify_status(virq, IRQ_LEVEL, IRQ_NOPROBE);
    //     ↑ 标记为电平触发
}
```

### 4.2 GIC xlate — DTS 中断号翻译

```c
// RV1126B DTS 中的中断声明:
// interrupts = <GIC_SPI 45 IRQ_TYPE_LEVEL_HIGH>;
//              = <0        45 4>;

GIC_SPI = 0    // SPI 类型
GIC_PPI = 1    // PPI 类型
GIC_SGI = 0    // SGI (实际和 SPI 同代码)

// xlate 函数:
static int gic_irq_domain_xlate(struct irq_domain *d,
                                 struct device_node *node,
                                 const u32 *intspec, unsigned int intsize,
                                 unsigned long *out_hwirq,
                                 unsigned int *out_type)
{
    if (intspec[0] == 0) {         // GIC_SPI
        *out_hwirq = 32 + intspec[1];   // ★ SPI 偏移: hwirq = 32 + 45 = 77
    } else if (intspec[0] == 1) { // GIC_PPI
        *out_hwirq = 16 + intspec[1];   // PPI 偏移: hwirq = 16 + n
    }
    *out_type = intspec[2] & IRQ_TYPE_SENSE_MASK;
}

// 因此:
// DTS: interrupts = <GIC_SPI 45 IRQ_TYPE_LEVEL_HIGH>
// → hwirq = 77
// 在 Linux 中 request_irq(77, handler) 实际上注册的是 GIC SPI #45
```

### 4.3 RV1126B 中断映射表

```
中断类型       DTS 写法           hwirq    virq (线性)
─────────────────────────────────────────────────────
SGI #0~15     (软件中断, 不配 DTS)  0~15    0~15
PPI #0~15     (每 CPU 定时器等)     16~31   16~31   (GICC_PPI)
SPI #0~31     interrupts = <0 0 4>  32~63   32~63
SPI #32       interrupts = <0 32 4> 64      64
SPI #33       interrupts = <0 33 4> 65      65
SPI #45       interrupts = <0 45 4> 77      77      ← I2C5 中断
SPI #46       interrupts = <0 46 4> 78      78
...
SPI #127      interrupts = <0 127 4> 159   159

GPIO0#5       (gpio_to_irq)         GIC SPI 专用号  动态分配
```

---

## 五、完整映射流程：从 DTS 到 request_irq

```c
// 驱动 probe 中:
int irq = platform_get_irq(pdev, 0);
//           ↓
// 1. of_irq_parse_one(dev->of_node, 0, &oirq)
//    → 解析 DTS: interrupts = <GIC_SPI 45 IRQ_TYPE_LEVEL_HIGH>
//    → oirq.np = <&gic>, oirq.args = [0, 45, 4]

// 2. irq_create_of_mapping(&oirq)
//    ↓
// 2a. irq_find_matching_fwspec(fwspec, DOMAIN_BUS_WIRED)
//     → 遍历 irq_domain_list, 匹配到 GIC domain
//
// 2b. domain->ops->xlate(d, node, intspec, ...)
//     = gic_irq_domain_xlate()
//     → out_hwirq = 32 + 45 = 77
//     → out_type  = IRQ_TYPE_LEVEL_HIGH
//
// 2c. irq_create_mapping(domain, 77)
//     → irq_find_mapping(domain, 77)  ← 检查是否已有映射
//     → irq_domain_alloc_descs()      ← 分配 virq
//     → irq_domain_associate_locked()
//       → irq_data->hwirq = 77
//       → irq_data->domain = domain
//       → domain->ops->map(d, virq, 77)
//         = gic_irq_domain_map()
//         → irq_set_chip(virq, &gic_chip)
//         → irq_set_handler(virq, handle_fasteoi_irq)
//     → domain->revmap[77] = irq_data  (线性映射)
//
// 返回 virq = 77

// 3. request_irq(77, handler, ...)
//    → irq_to_desc(77) → struct irq_desc
//    → desc->handle_irq = handle_fasteoi_irq  (在 map 阶段设置)
//    → desc->irq_data.chip = &gic_chip
//    → chip->irq_request_resources()
//    → chip->irq_startup() / irq_enable()
//      → writel(1 << (hwirq % 32), GICD_ISENABLER + (hwirq/32)*4)
```

---

## 六、层级 IRQ Domain (Hierarchy)

### 6.1 问题

```
GPIO 中断级联:

  GPIO0 控制器           GIC-400
  ┌──────────┐          ┌──────────┐
  │ pin 0~31 │──SPI#64──→│ SPI      │
  │   (hwirq)│          │ handler  │
  └──────────┘          └──────────┘

  DTS:
  gpio0: gpio@... {
      interrupt-controller;
      #interrupt-cells = <2>;
      interrupts = <GIC_SPI 64 IRQ_TYPE_LEVEL_HIGH>;
                 // ↑ GPIO0 控制器本身的中断 (连接 GIC SPI#64)
  };

  sensor: sensor@... {
      interrupt-parent = <&gpio0>;   // 不是 &gic!
      interrupts = <5 IRQ_TYPE_LEVEL_LOW>;
                  // ↑ GPIO0 pin 5
  };

问题: sensor 的中断先经过 GPIO0 控制器, 再经过 GIC
      → 需要两级 irq_domain 链
```

### 6.2 层级映射

```
irq_domain 链:
                    xlate → hwirq:5
  sensor ─────────→ GPIO domain ──→ GIC domain
                     parent           parent
                     ↓                ↓
                   virq:N           hwirq:64
                    (GPIO)           (GIC SPI)

1. sensor 的 interrupts 引用 gpio0
   → of_irq_parse_one() 解析为:
     oirq.np = <&gpio0>, args = [5, 8]  // pin5, low-level

2. irq_find_matching_fwspec() → GPIO domain

3. GPIO domain->ops->xlate() → hwirq = 5 (GPIO pin number)

4. irq_create_mapping(gpio_domain, 5)
   → 分配 virq
   → gpio_domain->ops->map()
     → 设置 irq_chip = gpio_chip (不是 gic_chip!)
     → 设置 handler = gpio_irq_handler
   → ★ 内部调用 irq_create_mapping(gic_domain, 64)
      → GIC domain 为 GPIO0 分配第 2 层 virq
      → GPIO0 的 GIC 中断线

5. 最终:
   request_irq(virq_N, sensor_handler)
   → 中断触发: GIC → GPIO_domain → sensor_handler
               (hwirq:64)  (hwirq:5)
```

### 6.3 层级 API

```c
// 创建层级 domain (GPIO domain 以 GIC domain 为 parent):
gpio_domain = irq_domain_add_linear(gpio_fwnode, 32,
                                     &gpio_irq_domain_ops, gpio_chip);
irq_domain_set_hierarchy(gpio_domain, gic_domain);

// 或:
gpio_domain = irq_domain_create_hierarchy(gic_domain, 0,
                                           32, fwnode, &gpio_ops, chip);
```

---

## 七、中断号映射关系表

```bash
# 查看系统所有 IRQ 映射:
cat /proc/interrupts
#            CPU0 CPU1 CPU2 CPU3
#  16:       435    0    0    0     GIC-400  29  timer      ← PPI
#  29:        13    0    0    0     GIC-400  61  arch_timer
#  45:         0    0    0    0     GIC-400  77  ff3e0000.i2c  ← SPI
#  46:         0    0    0    0     GIC-400  78  ff3d0000.spi
#  50:         0    0    0    0     GIC-400  82  ff240000.dma-controller
#  ...
#  GIC-400 后面的数字是 hwirq
#  virq 等于第一列

# 查看 irq_domain 信息:
ls /sys/kernel/debug/irq/domains/
# GIC-400  gpio0  gpio1  ...
cat /sys/kernel/debug/irq/domains/GIC-400
# name: GIC-400
# size: 0
# mapped: 48
# flags: 0x00000001

# 查看特定 virq 的 domain 信息:
cat /sys/kernel/debug/irq/irqs/77
# handler:  handle_fasteoi_irq
# device:   ff3e0000.i2c
# domain:   GIC-400
# hwirq:    0x4d           ← 77 = 0x4d  (32 + 45)
# chip:     GIC-400
# action:   i2c_rk3x
```

---

## 八、思考题

1. **hwirq 与 virq 的关系**：在 RV1126B 上, GIC SPI #45 对应的 hwirq 是 77, virq 也是 77。这是巧合吗？如果系统中有多个中断控制器, hwirq 会冲突吗？

2. **线性 vs 基数树**：GIC-400 使用线性映射, GPIO 控制器使用基数树映射。为什么？一个 GPIO 控制器有 32 个 pin, 使用线性映射也可以, 为什么 GPIO 选择基数树？

3. **层级 domain 的 xlate**：在 GPIO 层级 domain 中, `xlate()` 返回的 hwirq 是 GPIO pin 号还是 GIC 的硬件中断号？`irq_create_mapping()` 是在哪个 domain 上分配 virq？

4. **`domain->map()` 调用时机**：`irq_domain_associate_locked()` 中调用 `domain->ops->map()`, 这个函数可能访问硬件寄存器 (如配置 GICD_ICFGR)。如果这个映射是在中断被 request 之前做的, 那什么时候真正使能这个中断？

---

## 相关笔记

- [[bsp-interrupt-concurrency]] — 阶段三：中断处理 + 并发
- [[bsp-device-model-dtb]] — DTS interrupts 属性
- [[bsp-device-model-dtb-unflatten-deep]] — DTB 中断解析
- [[MOC-嵌入式Linux]] — 嵌入式 Linux 学习路线 MOC
