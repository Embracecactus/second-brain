---
tags:
  - embedded-linux
  - bsp
  - device-model
  - platform-bus
  - kernel-core
  - source-code
  - rockchip
category: embedded-linux
created: 2026-06-23
updated: 2026-06-23
status: active
soc: Rockchip RV1126B
kernel: Linux 6.1.141
parent: bsp-device-model-dtb
---

# Platform Bus 源码追溯

> **前置笔记**：[[bsp-device-model-dtb]] — 阶段二：设备模型 + 设备树
>
> **核心文件**：`drivers/base/platform.c`, `drivers/of/platform.c`, `drivers/base/dd.c`, `include/linux/platform_device.h`

---

## 一、从 DTS 到 platform_device

### 1.1 入口：`of_platform_default_populate_init()`

```c
// drivers/of/platform.c:517
static int __init of_platform_default_populate_init(void)
{
    // 1. 检查 DTB 是否已展开
    if (!of_have_populated_dt())
        return -ENODEV;

    // 2. 对 PPC 特殊处理 (跳过)

    // 3. 非 PPC: from 设备树根节点开始 populate
    of_platform_default_populate(NULL, NULL, NULL);
    //                                ↑      ↑
    //                          matches    lookup
    //                          = NULL     = NULL
    //  → 使用 of_default_bus_match_table
}
```

### 1.2 `of_platform_default_populate()` → `of_platform_populate()`

```c
// drivers/of/platform.c:497
int of_platform_default_populate(struct device_node *root, ...)
{
    return of_platform_populate(root, of_default_bus_match_table, lookup, parent);
    //                                       ↑
    //  匹配表: 只有 compatible = "simple-bus", "simple-mfd", "arm,amba-bus" 的节点
    //         才会递归创建子设备
}

// drivers/of/platform.c:465
int of_platform_populate(struct device_node *root, matches, lookup, parent)
{
    // 根节点 = "/"
    for_each_child_of_node(root, child) {
        rc = of_platform_bus_create(child, matches, lookup, parent, true);
        //                                                  ↑  strict=true
        //  要求每个节点必须有 compatible 属性, 否则跳过
    }
}
```

### 1.3 核心递归：`of_platform_bus_create()`

```c
// drivers/of/platform.c:343
static int of_platform_bus_create(struct device_node *bus, matches, lookup, parent, strict)
{
    // 1. strict 模式: 跳过没有 compatible 的节点
    if (strict && !of_get_property(bus, "compatible", NULL))
        return 0;

    // 2. 跳过特殊节点 (如 operating-points-v2)
    if (of_match_node(of_skipped_node_table, bus))
        return 0;

    // 3. 防止重复创建 (OF_POPULATED_BUS flag)
    if (of_node_check_flag(bus, OF_POPULATED_BUS))
        return 0;

    // 4. 为当前节点创建 platform_device
    dev = of_platform_device_create_pdata(bus, bus_id, platform_data, parent);
    if (!dev || !of_match_node(matches, bus))
        return 0;    // 不匹配 simple-bus → 不递归子节点

    // 5. ★ 递归: 只有匹配 simple-bus/mfd 的节点才创建子设备
    for_each_child_of_node(bus, child) {
        rc = of_platform_bus_create(child, matches, lookup, &dev->dev, strict);
    }
    of_node_set_flag(bus, OF_POPULATED_BUS);
}
```

### 1.4 `of_platform_device_create_pdata()` — 真正的创建

```c
// drivers/of/platform.c:167
static struct platform_device *of_platform_device_create_pdata(np, bus_id, platform_data, parent)
{
    // 1. 检查设备是否启用 (status = "okay")
    if (!of_device_is_available(np))
        return NULL;

    // 2. 防止重复创建 (OF_POPULATED flag)
    if (of_node_test_and_set_flag(np, OF_POPULATED))
        return NULL;

    // 3. ★ 核心: 分配 platform_device 并解析资源
    dev = of_device_alloc(np, bus_id, parent);

    dev->dev.coherent_dma_mask = DMA_BIT_MASK(32);
    dev->dev.bus = &platform_bus_type;    // ← 绑定到 platform 总线

    // 4. ★ 注册到设备模型
    if (of_device_add(dev) != 0) {
        platform_device_put(dev);
        goto err_clear_flag;
    }
}
```

### 1.5 `of_device_alloc()` — 资源解析

```c
// drivers/of/platform.c:112
struct platform_device *of_device_alloc(struct device_node *np, bus_id, parent)
{
    struct platform_device *dev;

    dev = platform_device_alloc("", PLATFORM_DEVID_NONE);
    //                    ↑  name 为空, 后面通过 bus_id 或地址命名

    // 1. 统计并解析 reg 资源
    while (of_address_to_resource(np, num_reg, &temp_res) == 0)
        num_reg++;

    if (num_reg) {
        res = kcalloc(num_reg, sizeof(*res), GFP_KERNEL);
        dev->num_resources = num_reg;
        dev->resource = res;
        for (i = 0; i < num_reg; i++)
            of_address_to_resource(np, i, res);
            //  → 将 DTS reg 属性 → struct resource (IORESOURCE_MEM/IO)
    }

    // 2. 关联 device_node
    dev->dev.of_node = of_node_get(np);
    dev->dev.parent = parent ?: &platform_bus;

    // 3. ★ 生成唯一名称
    if (bus_id)
        dev_set_name(&dev->dev, "%s", bus_id);
    else
        of_device_make_bus_id(&dev->dev);
    //     → 优先用 地址.掩码.节点名 格式
    //       如: "ff3e0000.i2c", "ff3d0000.spi"

    return dev;
}
```

### 1.6 `of_device_make_bus_id()` — 命名规则

```c
// drivers/of/platform.c:74
static void of_device_make_bus_id(struct device *dev)
{
    struct device_node *node = dev->of_node;
    // 优先: 从 reg 获取地址
    //   get: reg 属性的地址值
    //   format: "%llx.%pOFn" → "ff3e0000.i2c"
    //
    // 次优: 逐级向上用父节点名做前缀
    //   node = node->parent 循环
    //   format: "%s"
}
```

**平台设备命名示例**（从 RV1126B DTS 解析结果）：

| 设备树节点 | 生成的 platform_device 名 |
|-----------|------------------------|
| `i2c5: i2c@ff3e0000` | `ff3e0000.i2c` |
| `spi2: spi@ff3d0000` | `ff3d0000.spi` |
| `uart0: serial@ff390000` | `ff390000.serial` |
| `dmac: dma-controller@ff240000` | `ff240000.dma-controller` |

---

## 二、platform_bus 初始化

### 2.1 全局实例

```c
// drivers/base/platform.c:42
struct device platform_bus = {
    .init_name = "platform",
};

// 这是所有 platform_device 的虚拟父设备
// 在 /sys/devices/platform/ 可见
```

### 2.2 总线注册

```c
// drivers/base/platform.c:1513
int __init platform_bus_init(void)
{
    // 1. 注册 "platform" 设备本身
    error = device_register(&platform_bus);   // → /sys/devices/platform

    // 2. 注册 platform 总线类型
    error = bus_register(&platform_bus_type); // → /sys/bus/platform

    // 3. 监听 DTS overlay 通知 (动态设备树支持)
    of_platform_register_reconfig_notifier();
}
```

### 2.3 `platform_bus_type` 定义

```c
// drivers/base/platform.c:1478
struct bus_type platform_bus_type = {
    .name      = "platform",
    .dev_groups = platform_dev_groups,
    .match     = platform_match,        // ★ 匹配算法
    .uevent    = platform_uevent,       // 热插拔事件
    .probe     = platform_probe,        // ★ probe 调用
    .remove    = platform_remove,
    .shutdown  = platform_shutdown,
    .dma_configure = platform_dma_configure,
    .dma_cleanup   = platform_dma_cleanup,
    .pm        = &platform_dev_pm_ops,
};
```

---

## 三、platform_driver 注册

### 3.1 `platform_driver_register()` 宏

```c
// include/linux/platform_device.h:244
#define platform_driver_register(drv) \
    __platform_driver_register(drv, THIS_MODULE)
```

### 3.2 `__platform_driver_register()`

```c
// drivers/base/platform.c:861
int __platform_driver_register(struct platform_driver *drv, struct module *owner)
{
    drv->driver.owner = owner;
    drv->driver.bus   = &platform_bus_type;   // ← 绑定到 platform 总线

    return driver_register(&drv->driver);
    //       ↓
    //   drivers/base/driver.c
    //   → bus_add_driver() → klist_add_tail(&priv->knode_bus, ...)
    //   → driver_attach(drv)  ← ★ 触发已注册设备的匹配
}
```

### 3.3 `module_platform_driver()` 宏展开

```c
// 宏: module_platform_driver(xxx_driver)
// 展开为:
static int __init xxx_driver_init(void)
{
    return platform_driver_register(&xxx_driver);
}
module_init(xxx_driver_init);

static void __exit xxx_driver_exit(void)
{
    platform_driver_unregister(&xxx_driver);
}
module_exit(xxx_driver_exit);
```

---

## 四、匹配算法全解

### 4.1 `platform_match()` — 四种匹配方式

```c
// drivers/base/platform.c:1331
static int platform_match(struct device *dev, struct device_driver *drv)
{
    struct platform_device *pdev = to_platform_device(dev);
    struct platform_driver *pdrv = to_platform_driver(drv);

    // 1. ★ driver_override: 强制绑定
    if (pdev->driver_override)
        return !strcmp(pdev->driver_override, drv->name);

    // 2. ★ OF 匹配: of_match_table → DTS compatible
    //    这是最常用的方式
    if (of_driver_match_device(dev, drv))
        return 1;

    // 3. ACPI 匹配 (嵌入式基本不用)
    if (acpi_driver_match_device(dev, drv))
        return 1;

    // 4. id_table 匹配: platform_driver.id_table → pdev->name
    if (pdrv->id_table)
        return platform_match_id(pdrv->id_table, pdev) != NULL;

    // 5. ★ 回退: 直接比较 name 字符串
    return (strcmp(pdev->name, drv->name) == 0);
}
```

### 4.2 OF 匹配细节

```c
// drivers/of/device.c
int of_driver_match_device(struct device *dev, struct device_driver *drv)
{
    // 设备没有 of_node → 无法匹配
    if (!dev->of_node)
        return 0;

    // 用 drv->of_match_table 和 dev->of_node->compatible 做比较
    return of_match_device(drv->of_match_table, dev) != NULL;
}

// drivers/of/base.c
const struct of_device_id *of_match_device(const struct of_device_id *matches,
                                          const struct device *dev)
{
    if (!matches)
        return NULL;

    // 遍历 of_match_table 里的所有 compatible
    // 调用 of_match_node() 与设备节点的 compatible 属性对比
    return of_match_node(matches, dev->of_node);
}
```

### 4.3 四种匹配方式优先级

```
platform_match() 调用顺序

  ┌─ pdev->driver_override 设置了吗?
  │   ├─ YES → 强制绑定到指定驱动
  │   └─ NO  → 继续
  │
  ├─ DTS compatible 匹配?
  │   ├─ YES → ★ 现代 BSP 的标准路径
  │   └─ NO  → 继续
  │
  ├─ ACPI 匹配?
  │   └─ (嵌入式极少用)
  │
  ├─ id_table 匹配?
  │   ├─ YES → 用于旧式/非 DTS 设备
  │   └─ NO  → 继续
  │
  └─ name 字符串匹配 (最后回退)
```

---

## 五、Probe 全路径

### 5.1 触发点：`driver_attach()`

```
  驱动注册 (module_init → platform_driver_register)
       │
       ▼
   driver_register()          drivers/base/driver.c
       │
       ▼
   bus_add_driver()           drivers/base/bus.c
       │  用 bus_type.match 遍历已注册的 device 列表
       │  对每个 device 调用 __driver_attach()
       ▼
   __driver_attach()          drivers/base/dd.c
       │
       ├─ device_is_bound(dev)? → 跳过
       ├─ driver_match_device(dev, drv) → bus->match(dev, drv)
       └─ 匹配成功 → driver_probe_device(dev, drv)
```

### 5.2 `driver_probe_device()`

```c
// drivers/base/dd.c:472
int driver_probe_device(struct device_driver *drv, struct device *dev)
{
    int ret = 0;

    if (!device_is_registered(dev))
        return -ENODEV;

    // 1. 检查设备-驱动依赖 (fw_devlink)
    //    如果 supplier 还没 probe → -EPROBE_DEFER
    ret = device_links_check_suppliers(dev);
    if (ret == -EPROBE_DEFER)
        return ret;

    // 2. 检查 atomic 上下文中是否允许 probe
    if (!drv->allow_probe_in_atomic && !probe_hold) {
        // 实际工作推迟到 probe workqueue

        //   ★ 默认: probe 在进程上下文执行
    }

    // 3. 调用 really_probe()
    ret = really_probe(dev, drv);
}
```

### 5.3 `really_probe()` — probe 核心

```c
// drivers/base/dd.c:584
static int really_probe(struct device *dev, struct device_driver *drv)
{
    // 1. 全局推迟? (设备初始化未完成时)
    if (defer_all_probes)
        return -EPROBE_DEFER;

    // 2. 依赖检查 (device links)
    link_ret = device_links_check_suppliers(dev);
    if (link_ret == -EPROBE_DEFER)
        return link_ret;

    // 3. 设置 dev->driver
    dev->driver = drv;

    // 4. pinctrl 绑定 (如果 DTS 有 pinctrl-* 属性)
    ret = pinctrl_bind_pins(dev);
    //    → DTS: pinctrl-names = "default";
    //           pinctrl-0 = <&i2c5_pins>;

    // 5. DMA 配置 (from DTS dma-ranges / dma-coherent)
    if (dev->bus->dma_configure)
        ret = dev->bus->dma_configure(dev);

    // 6. ★ 创建 sysfs 接口
    ret = driver_sysfs_add(dev);
    //    → /sys/devices/platform/xxx/driver

    // 7. PM domain activate
    if (dev->pm_domain && dev->pm_domain->activate)
        ret = dev->pm_domain->activate(dev);

    // 8. ★★ 调用 bus->probe (即 platform_probe)
    ret = call_driver_probe(dev, drv);
    //         ↓
    //   platform_probe(_dev)

    // 9. Probe 成功 → driver_bound()
    if (!ret) {
        driver_bound(dev);
        //   → klist_add_tail(&dev->knode_driver, ...)
        //   → 触发 device_bind_driver() 通知
    }
}
```

### 5.4 `platform_probe()` — bus 层的 probe 包装

```c
// drivers/base/platform.c:1375
static int platform_probe(struct device *_dev)
{
    struct platform_driver *drv = to_platform_driver(_dev->driver);
    struct platform_device *dev = to_platform_device(_dev);

    // 1. 检查是否是 __init probe (不可重入)
    if (unlikely(drv->probe == platform_probe_fail))
        return -ENXIO;

    // 2. ★ 从 DTS 初始化时钟 (clk = devm_clk_get)
    ret = of_clk_set_defaults(_dev->of_node, false);
    //    → 解析 DTS clock-names / clocks 属性
    //    → 对 assigned-clocks 配置频率/父时钟

    // 3. ★ 连接 PM domain
    ret = dev_pm_domain_attach(_dev, true);
    //    → 解析 DTS power-domains 属性
    //    → dev->pm_domain = &genpd->domain

    // 4. ★★ 调用驱动自己的 probe 函数
    if (drv->probe) {
        ret = drv->probe(dev);
        //   → your_driver_probe(struct platform_device *pdev)
        if (ret)
            dev_pm_domain_detach(_dev, true);
    }

    // 5. 防止 deferred probe? (平台驱动专用)
    if (drv->prevent_deferred_probe && ret == -EPROBE_DEFER) {
        ret = -ENXIO;
    }

    return ret;
}
```

### 5.5 `call_driver_probe()` → `really_probe()` → `driver_bound()`

```c
static void driver_bound(struct device *dev)
{
    // 1. 加入驱动设备链表
    klist_add_tail(&dev->knode_driver, &dev->driver->p->klist_devices);

    // 2. 触发 device_bind_driver 通知链
    blocking_notifier_call_chain(&dev->bus->p->bus_notifier,
                                 BUS_NOTIFY_BOUND_DRIVER, dev);

    // 3. 触发供应商 consumer 检查
    device_links_driver_bound(dev);

    // 4. 移除 deferred probe 列表
    if (dev->needs_deferred_probe)
        driver_deferred_probe_del(dev);

    // 5. sysfs 创建
    sysfs_create_link(&dev->driver->p->kobj, &dev->kobj,
                      kobject_name(&dev->kobj));
    sysfs_create_link(&dev->kobj, &dev->driver->p->kobj, "driver");
}
```

---

## 六、完整调用链（附代码行号）

```
DTS → platform_device
---------------------

unflatten_device_tree()                    // drivers/of/fdt.c
  → of_scan_flat_dt()
  → of_platform_default_populate_init()    // drivers/of/platform.c:517
    → of_platform_default_populate()       // drivers/of/platform.c:497
      → of_platform_populate()             // drivers/of/platform.c:465
        → of_platform_bus_create()         // drivers/of/platform.c:343
          → of_platform_device_create_pdata()  // drivers/of/platform.c:167
            → of_device_alloc()            // drivers/of/platform.c:112
              → platform_device_alloc()
              → of_address_to_resource()   // 解析 reg → resource[]
              → of_device_make_bus_id()    // 命名 "ff3e0000.i2c"
            → of_device_add()              // drivers/of/platform.c:190
              → device_add()               // drivers/base/core.c
        → (递归子节点: simple-bus 匹配)

Driver 注册 → Probe
--------------------

platform_driver_register(drv)              // include/linux/platform_device.h:244
  → __platform_driver_register()           // drivers/base/platform.c:861
    → driver_register()                    // drivers/base/driver.c
      → bus_add_driver()                   // drivers/base/bus.c
        → driver_attach(drv)               // drivers/base/dd.c
          → __driver_attach()              // drivers/base/dd.c
            → driver_match_device()        // drivers/base/base.h
              → bus->match() = platform_match()  // platform.c:1331
                → of_driver_match_device()  // 匹配 DTS compatible
            → driver_probe_device()         // drivers/base/dd.c:472
              → really_probe()             // drivers/base/dd.c:584
                → pinctrl_bind_pins()
                → dev->bus->dma_configure()
                → driver_sysfs_add()
                → call_driver_probe()      // drivers/base/dd.c
                  → bus->probe() = platform_probe()  // platform.c:1375
                    → of_clk_set_defaults()
                    → dev_pm_domain_attach()
                    → drv->probe(dev)       // ★ 你的 probe 函数!
                → driver_bound()           // sysfs 链接创建
```

---

## 七、资源管理

### 7.1 DTS 属性 → platform_device 资源映射

| DTS 属性 | platform_device 字段 | 获取 API |
|---------|---------------------|---------|
| `reg = <0x1 0x2>` | `resource[]` (IORESOURCE_MEM) | `platform_get_resource(pdev, IORESOURCE_MEM, 0)` |
| `interrupts = <GIC_SPI 45>` | `resource[]` (IORESOURCE_IRQ) | `platform_get_irq(pdev, 0)` |
| `clocks = <&cru CLK_I2C5>` | 不直接存储 | `devm_clk_get(dev, "bus")` |
| `dmas = <&dmac 0 1>` | 不直接存储 | `dma_request_slave_channel(dev, "tx")` |
| `pinctrl-0 = <&i2c5_pins>` | 不直接存储 | `pinctrl_bind_pins(dev)` (自动) |

### 7.2 `platform_get_resource()` — 资源查询源码

```c
// drivers/base/platform.c:55
struct resource *platform_get_resource(struct platform_device *dev,
                                       unsigned int type, unsigned int num)
{
    // 遍历 dev->resource[] 数组, 按 type 匹配
    // 支持 IORESOURCE_MEM, IORESOURCE_IO, IORESOURCE_IRQ, ...
    for (i = 0; i < dev->num_resources; i++) {
        struct resource *r = &dev->resource[i];
        if (type == resource_type(r) && num-- == 0)
            return r;
    }
    return NULL;
}
```

### 7.3 `devm_` 自动管理

```c
// 为什么推荐使用 devm_ 系列 API?

// 非 devm 写法: 手动管理
static int my_probe(struct platform_device *pdev)
{
    clk = clk_get(dev, "bus");
    irq = platform_get_irq(pdev, 0);
    ret = request_irq(irq, handler, 0, "mydev", dev);
    // 如果忘记释放 → 资源泄漏
}

static int my_remove(struct platform_device *pdev)
{
    clk_put(clk);               // 必须手动
    free_irq(irq, dev);         // 必须手动
}

// devm 写法: 自动释放
static int my_probe(struct platform_device *pdev)
{
    clk = devm_clk_get(dev, "bus");   // devm_ = device-managed
    irq = platform_get_irq(pdev, 0);
    ret = devm_request_irq(dev, irq, handler, 0, "mydev", dev);
    // probe 失败或 remove 时自动释放!
}
```

`devm_` 实现原理:

```c
// devm_ 注册的资源挂在 dev->devres_head 链表上
// 设备释放时 (device_release 或 probe 失败):
//   devres_release_all(dev)
//     → 遍历 devres_head, 按注册顺序逆序释放
```

---

## 八、Deferred Probe 机制

当 `platform_probe()` 返回 `-EPROBE_DEFER` 时:

```c
// drivers/base/dd.c:really_probe()
// 返回 -EPROBE_DEFER → drivers/base/dd.c:__driver_attach()
// → driver_deferred_probe_add(dev) → 加入 delayed_list
//
// 之后的某个时刻, supplier 驱动 probe 完成:
//   device_links_driver_bound(dev)
//     → driver_deferred_probe_trigger()
//       → 重新遍历 delayed_list, 再次尝试 probe
```

典型场景:

```
1. 驱动 A 的 probe 需要时钟 clk_A
2. clk_A 由 clock-controller 驱动 B 提供
3. 驱动 A 的 module_init 比驱动 B 早
4. 驱动 A probe 时 clk_A 不可用 → -EPROBE_DEFER
5. 驱动 B probe 成功, 注册 clk_A
6. 内核自动重新尝试 A 的 probe
```

---

## 九、Ftrace 追踪 Probe 流程

```bash
# 1. 开启函数追踪
echo function > /sys/kernel/debug/tracing/current_tracer
echo platform_probe > /sys/kernel/debug/tracing/set_ftrace_filter
echo really_probe >> /sys/kernel/debug/tracing/set_ftrace_filter
echo driver_probe_device >> /sys/kernel/debug/tracing/set_ftrace_filter
echo platform_match >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on

# 2. 触发 probe (重新绑定驱动)
echo -n "ff3e0000.i2c" > /sys/bus/platform/drivers/i2c_rk3x/unbind
echo -n "ff3e0000.i2c" > /sys/bus/platform/drivers/i2c_rk3x/bind

# 3. 查看
cat /sys/kernel/debug/tracing/trace
```

---

## 十、关键数据结构一览

```c
// include/linux/platform_device.h
struct platform_device {
    const char      *name;           // 设备名, e.g. "ff3e0000.i2c"
    int             id;              // 设备 ID, -1 = 自动
    bool            id_auto;         // ID 是否自动分配
    struct device   dev;             // ★ 嵌入的通用 device
    u32             num_resources;   // 资源数组大小
    struct resource *resource;       // ★ DTS 解析出来的资源 (reg, irq)
    const struct platform_device_id *id_entry; // 匹配到的 id_table 条目
    char            driver_override[32]; // 强制绑定驱动名
};

// include/linux/platform_device.h
struct platform_driver {
    int (*probe)(struct platform_device *);          // ★ probe 函数
    int (*remove)(struct platform_device *);         // remove
    void (*shutdown)(struct platform_device *);      // 关机
    int (*suspend)(struct platform_device *, pm_message_t);  // 旧式 PM
    int (*resume)(struct platform_device *, pm_message_t);
    struct device_driver driver;                     // ★ 嵌入的通用 driver
    const struct platform_device_id *id_table;       // 非 DTS 匹配表
    bool prevent_deferred_probe;                     // 禁用 deferred probe
};

// include/linux/device.h 中 device_driver 部分:
struct device_driver {
    const char              *name;           // 驱动名
    struct bus_type         *bus;            // ← platform_bus_type
    struct module           *owner;
    const struct of_device_id   *of_match_table;   // ★ DTS compatible 匹配表
    int (*probe)(struct device *);          // bus probe 包装后的调用
    int (*remove)(struct device *);
    void (*shutdown)(struct device *);
    const struct dev_pm_ops *pm;
};
```

---

## 十一、思考题

1. **simple-bus 的作用**：如果某个 DTS 节点的 compatible 不包含 `"simple-bus"`，它的子节点会创建 platform_device 吗？为什么？

2. **`-EPROBE_DEFER` 限制**：`platform_driver.prevent_deferred_probe` 设置为 true 时，probe 返回 `-EPROBE_DEFER` 会怎样？什么场景下需要这个标志？

3. **资源顺序**：`of_address_to_resource()` 解析 reg 属性时，如果 reg = `<0x0 0xff3e0000 0x0 0x1000>`, 最终 `struct resource` 的 start/end/flags 是什么？(提示：`#address-cells = <2>`)

4. **DMA mask**：`of_platform_device_create_pdata()` 中默认设置 `coherent_dma_mask = DMA_BIT_MASK(32)`，但如果设备需要 64-bit DMA 地址空间，驱动应该在哪里修改？

---

## 相关笔记

- [[bsp-device-model-dtb]] — 阶段二：设备模型 + 设备树
- [[bsp-interrupt-concurrency]] — 阶段三：中断
- [[bsp-uboot-dm-deep]] — U-Boot Driver Model 对比
- [[MOC-嵌入式Linux]] — 嵌入式 Linux 学习路线 MOC
