---
tags:
  - embedded-linux
  - bsp
  - device-model
  - driver-probe
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

# Driver Probe 全路径源码追溯

> **前置笔记**：[[bsp-device-model-platform-bus-deep]] — Platform Bus 源码追溯
>
> **核心文件**：`drivers/base/dd.c`, `drivers/base/driver.c`, `drivers/base/bus.c`, `drivers/base/core.c`

---

## 一、触发入口：Driver 注册触发 Probe

### 1.1 `driver_attach()` — 总线设备遍历

```c
// drivers/base/dd.c:1216
int driver_attach(struct device_driver *drv)
{
    // 遍历 drv->bus (platform_bus_type) 上所有已注册的 device
    // 对每个 device 调用 __driver_attach()
    return bus_for_each_dev(drv->bus, NULL, drv, __driver_attach);
}
```

### 1.2 `__driver_attach()` — 匹配 + 调度 Probe

```c
// drivers/base/dd.c:1141
static int __driver_attach(struct device *dev, void *data)
{
    struct device_driver *drv = data;

    // 1. ★ 调用 bus->match() 做匹配检查
    ret = driver_match_device(drv, dev);
    //     → bus->match() = platform_match()
    //     ret=0: 不匹配, 跳过
    //     ret=-EPROBE_DEFER: 推迟 (供应商依赖)
    //     ret>0: 匹配成功

    if (ret == 0)   return 0;    // 不匹配, 继续下一个
    if (ret == -EPROBE_DEFER) {
        driver_deferred_probe_add(dev);   // 加入推迟列表
        return 0;
    }

    // 2. 检查异步 probe
    if (driver_allows_async_probing(drv)) {
        // → 异步调度, 不阻塞其他设备 probe
        async_schedule_dev(__driver_attach_async_helper, dev);
        return 0;
    }

    // 3. ★ 同步 probe: 调用 driver_probe_device()
    __device_driver_lock(dev, dev->parent);
    driver_probe_device(drv, dev);
    __device_driver_unlock(dev, dev->parent);
}
```

---

## 二、Probe 执行路径

### 2.1 `driver_probe_device()` — Probe 总入口

```c
// drivers/base/dd.c:809
static int driver_probe_device(struct device_driver *drv, struct device *dev)
{
    int trigger_count = atomic_read(&deferred_trigger_count);

    atomic_inc(&probe_count);           // 追踪进行中的 probe 数量
    ret = __driver_probe_device(drv, dev);

    // ★ 如果返回 -EPROBE_DEFER → 加入推迟队列
    if (ret == -EPROBE_DEFER || ret == EPROBE_DEFER) {
        driver_deferred_probe_add(dev);
        // 如果在 probe 过程中触发了新的依赖就绪,
        // 立即重新触发推迟队列
        if (trigger_count != atomic_read(&deferred_trigger_count) &&
            !defer_all_probes)
            driver_deferred_probe_trigger();
    }

    atomic_dec(&probe_count);
    wake_up_all(&probe_waitqueue);
}
```

### 2.2 `__driver_probe_device()` — 前置准备

```c
// drivers/base/dd.c:764
static int __driver_probe_device(struct device_driver *drv, struct device *dev)
{
    // 1. 严格检查: 设备不能是 dead, 必须已注册, 不能已经有 driver
    if (dev->p->dead || !device_is_registered(dev))
        return -ENODEV;
    if (dev->driver)
        return -EBUSY;

    dev->can_match = true;

    // 2. ★ Runtime PM 唤醒供应商链
    //    例如: I2C adapter 是 I2C client 的 parent
    //    在 client probe 之前必须确保 adapter 已 resume
    pm_runtime_get_suppliers(dev);
    if (dev->parent)
        pm_runtime_get_sync(dev->parent);

    // 3. PM 屏障: 确保所有待处理的 PM 操作完成
    pm_runtime_barrier(dev);

    // 4. ★ 调用 really_probe (或 debug 版本)
    if (initcall_debug)
        ret = really_probe_debug(dev, drv);  // 带耗时统计
    else
        ret = really_probe(dev, drv);

    // 5. 清理 PM
    pm_request_idle(dev);                  // 可重新进入 idle
    if (dev->parent)
        pm_runtime_put(dev->parent);
    pm_runtime_put_suppliers(dev);
}
```

### 2.3 `really_probe()` — Probe 核心执行体

```c
// drivers/base/dd.c:584
static int really_probe(struct device *dev, struct device_driver *drv)
{
    // ==== 阶段 0: 前提检查 ====
    if (defer_all_probes) return -EPROBE_DEFER;

    // ==== 阶段 1: 供应商依赖检查 ====
    link_ret = device_links_check_suppliers(dev);
    // → fw_devlink 机制: 检查所有 supplier 是否已 probe
    //   如 &i2c5 引用的 clock-controller 是否已就绪
    if (link_ret == -EPROBE_DEFER)
        return link_ret;

    // ==== 阶段 2: 设备-驱动绑定 ====
    dev->driver = drv;

    // ==== 阶段 3: pinctrl 绑定 ====
    ret = pinctrl_bind_pins(dev);
    // → DTS pinctrl-0 = <&i2c5_pins>;
    //   将 GPIO 复用为 I2C 功能

    // ==== 阶段 4: DMA 配置 ====
    if (dev->bus->dma_configure)
        ret = dev->bus->dma_configure(dev);
    // → of_dma_configure(): 解析 dma-ranges, dma-coherent
    //   设置 dev->coherent_dma_mask

    // ==== 阶段 5: sysfs 链接创建 ====
    ret = driver_sysfs_add(dev);
    // → /sys/bus/platform/drivers/xxx/device → ../devices/xxx
    // → /sys/devices/platform/xxx/driver       → ../../drivers/xxx

    // ==== 阶段 6: PM domain 激活 ====
    if (dev->pm_domain && dev->pm_domain->activate)
        ret = dev->pm_domain->activate(dev);
    // → 打开设备所属电源域 (如 RK806_PD_NPU)

    // ==== 阶段 7: device_add_groups ====
    ret = device_add_groups(dev, drv->dev_groups);

    // ==== 阶段 8: ★★ 真正的驱动 probe 调用 ====
    ret = call_driver_probe(dev, drv);
    //           ↓
    //   dev->bus->probe(dev) = platform_probe(dev)
    //           ↓
    //   of_clk_set_defaults()       ← 时钟初始化
    //   dev_pm_domain_attach()      ← PM domain attach
    //   drv->probe(dev)             ← ★ 你的 probe 函数!

    // ==== 阶段 9: 成功后处理 ====
    if (!ret) {
        pinctrl_init_done(dev);      // pinctrl 状态完成
        if (dev->pm_domain->sync)
            dev->pm_domain->sync(dev);

        driver_bound(dev);           // ★ 绑定完成通知
    }

    // ==== 失败处理 ====
    //   driver_sysfs_remove / dma_cleanup / device_unbind_cleanup
    //   → dev->driver = NULL (probe 失败回滚)

    return ret;
}
```

### 2.4 `call_driver_probe()` — 最终分发

```c
// drivers/base/dd.c:553
static int call_driver_probe(struct device *dev, struct device_driver *drv)
{
    if (dev->bus->probe)
        ret = dev->bus->probe(dev);     // → platform_probe()
    else if (drv->probe)
        ret = drv->probe(dev);          // 没有 bus → 直接调用

    // 错误分类:
    switch (ret) {
    case 0:        break;               // 成功
    case -EPROBE_DEFER:                 // 推迟 (供应商未就绪)
    case -ENODEV:
    case -ENXIO:   break;               // 不匹配, 静默处理
    default:       pr_warn(...);        // 其他错误, 打印警告
    }
}
```

### 2.5 `driver_bound()` — 绑定完成

```c
// drivers/base/dd.c:393
static void driver_bound(struct device *dev)
{
    // 1. 加入驱动设备链表 (klist_driver.klist_devices)
    klist_add_tail(&dev->p->knode_driver, &dev->driver->p->klist_devices);

    // 2. ★ 触发供应商依赖链
    device_links_driver_bound(dev);
    //   → 检查哪些 consumer 在等我 probe
    //   → 如果 consumer 之前因为等我而 -EPROBE_DEFER
    //   → → driver_deferred_probe_trigger()

    // 3. 检查 PM callback 有效性
    device_pm_check_callbacks(dev);

    // 4. 从推迟队列移除 + 触发重新 probe
    driver_deferred_probe_del(dev);
    driver_deferred_probe_trigger();

    // 5. 总线通知 + uevent
    blocking_notifier_call_chain(&dev->bus->p->bus_notifier,
                                  BUS_NOTIFY_BOUND_DRIVER, dev);
    kobject_uevent(&dev->kobj, KOBJ_BIND);    // udev 通知
}
```

---

## 三、完整的 Probe 流程图

```
Driver 模块加载
  ↓
module_init(xxx_init)
  → platform_driver_register(&xxx_driver)
    ↓
  __platform_driver_register()
    → drv->driver.bus = &platform_bus_type
    → driver_register(&drv->driver)
      ↓
    bus_add_driver()
      ← /sys/bus/platform/drivers/xxx 创建
      ↓
    driver_attach(drv)              /* ★ 关键: 与已有设备匹配 */
      ↓
    bus_for_each_dev(drv->bus, ...)
      → 遍历所有 platform_device
        ↓
      __driver_attach(dev, drv)
        → driver_match_device()     /* bus->match = platform_match */
        → 匹配成功?
            ↓ YES
          driver_probe_device(drv, dev)
            → __driver_probe_device()
              → pm_runtime_get_suppliers()
              → really_probe(dev, drv)
                ├─ device_links_check_suppliers()  /* fw_devlink */
                ├─ dev->driver = drv
                ├─ pinctrl_bind_pins()              /* 引脚复用 */
                ├─ dev->bus->dma_configure()        /* DMA 配置 */
                ├─ driver_sysfs_add()               /* sysfs 链接 */
                ├─ dev->pm_domain->activate()       /* 电源域 */
                ├─ device_add_groups()
                ├─ call_driver_probe()
                │   → dev->bus->probe()
                │     = platform_probe()
                │       ├─ of_clk_set_defaults()    /* 时钟配置 */
                │       ├─ dev_pm_domain_attach()   /* PM 绑定 */
                │       └─ drv->probe(dev)          /* ★ 你的驱动 */
                └─ driver_bound()
                  ├─ device_links_driver_bound()    /* 唤醒 consumer */
                  ├─ driver_deferred_probe_trigger()
                  └─ kobject_uevent(KOBJ_BIND)
```

---

## 四、fw_devlink 依赖链

### 4.1 原理

```c
// DTS 中的依赖关系:
// &i2c5 引用了 &cru 提供的时钟:
//   clocks = <&cru CLK_I2C5>, <&cru PCLK_I2C5>;
//
// 内核在 device_add() 时自动解析:
//   device_links_check_suppliers(dev)
//     → 遍历 dev->links.suppliers 列表
//     → 如果任何 supplier 还没 probe
//     → 返回 -EPROBE_DEFER
```

### 4.2 触发重新尝试

```c
// 当 supplier 完成 probe:
driver_bound(dev)
  → device_links_driver_bound(dev)
    → driver_deferred_probe_trigger()
      → 将 pending_list → active_list
      → schedule_work(deferred_probe_work)
        ↓
      deferred_probe_work_func()
        → 遍历 active_list, 重新 probe 所有被推迟的设备
```

---

## 五、Deferred Probe 机制细节

### 5.1 数据结构

```c
// drivers/base/dd.c

// 两条链表:
static LIST_HEAD(deferred_probe_pending_list);  // 等待触发的设备
static LIST_HEAD(deferred_probe_active_list);    // 正在重新 probe 的设备

// 每个设备通过 dev->p->deferred_probe 链表节点连接
DECLARE_WORK(deferred_probe_work, deferred_probe_work_func);
```

### 5.2 完整流程

```c
driver_probe_device() 返回 -EPROBE_DEFER
  ↓
driver_deferred_probe_add(dev)
  → list_add_tail(&dev->p->deferred_probe, &deferred_probe_pending_list)
  ↓
// ...供应商驱动 probe 完成...
driver_bound(dev)
  → device_links_driver_bound(dev)
    → driver_deferred_probe_trigger()
      ↓
deferred_probe_work_func()
  → pending_list → active_list (移动)
  → for each dev in active_list:
      ret = really_probe(dev, dev->driver)
      // ★ 重新走完整 probe 路径
  ↓
// 如果还不能 probe? → 再次进入 pending_list
```

### 5.3 查看推迟原因

```bash
# 查看所有被推迟 probe 的设备及其原因:
ls /sys/kernel/debug/devices_deferred
# 输出示例:
# ff3e0000.i2c    supplier 2021f000.clock-controller not ready
# ff3d0000.spi    supplier ff3e0000.i2c not ready... wait

# 手动触发重试:
echo 1 > /sys/bus/platform/drivers/xxx/bind
```

---

## 六、Async Probe 机制

```c
// 驱动声明异步 probe:
static struct platform_driver my_driver = {
    .driver = {
        .probe_type = PROBE_PREFER_ASYNCHRONOUS,
        // 或: 启动参数 driver_async_probe=my_driver
    },
};

// __driver_attach 中的异步路径:
if (driver_allows_async_probing(drv)) {
    async_schedule_dev(__driver_attach_async_helper, dev);
    // → 不阻塞其他设备 probe, 提升启动速度
}
```

---

## 七、Probe 中的资源获取（DTS → 驱动）

这是开发中最常写的代码：

```c
static int my_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;

    // 1. ★ 获取 MMIO 地址 (来自 DTS reg)
    struct resource *res;
    void __iomem *base;
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    base = devm_ioremap_resource(dev, res);
    // → 对应 DTS: reg = <0xff3e0000 0x1000>;

    // 2. ★ 获取中断号 (来自 DTS interrupts)
    int irq = platform_get_irq(pdev, 0);
    ret = devm_request_irq(dev, irq, handler,
                           IRQF_TRIGGER_RISING, "mydev", dev);
    // → 对应 DTS: interrupts = <GIC_SPI 45 IRQ_TYPE_LEVEL_HIGH>;

    // 3. ★ 获取时钟 (来自 DTS clocks)
    struct clk *clk = devm_clk_get(dev, "bus");
    clk_prepare_enable(clk);
    // → 对应 DTS: clocks = <&cru CLK_I2C5>, <&cru PCLK_I2C5>;
    //   clock-names = "bus", "pclk";

    // 4. 获取 GPIO (来自 DTS xxx-gpios)
    struct gpio_desc *gpio = devm_gpiod_get(dev, "enable", GPIOD_OUT_LOW);
    // → 对应 DTS: enable-gpios = <&gpio0 5 GPIO_ACTIVE_HIGH>;

    // 5. 获取 DMA 通道 (来自 DTS dmas)
    struct dma_chan *chan = dma_request_slave_channel(dev, "tx");
    // → 对应 DTS: dmas = <&dmac 0 1>, <&dmac 1 1>;
    //   dma-names = "tx", "rx";

    return 0;
}
```

### 资源获取 API 调用链

```c
platform_get_resource(pdev, IORESOURCE_MEM, 0)
  → pdev->resource[0] (在 of_device_alloc 中已填充)

devm_ioremap_resource(dev, res)
  → devm_ioremap(dev, res->start, resource_size(res))
  → 返回 __iomem void*

platform_get_irq(pdev, 0)
  → platform_get_resource(pdev, IORESOURCE_IRQ, 0)
  → irq_create_of_mapping()           // hwirq → virq
  → return virq

devm_clk_get(dev, "bus")
  → clk_get(dev, "bus")
    → of_clk_get_by_name(dev->of_node, "bus")
      → of_parse_phandle_with_args(dev->of_node, "clocks",
                                    "#clock-cells", 0, &clkspec)
      → __of_clk_get_from_provider(&clkspec)
```

---

## 八、思考题

1. **Probe 重入**：`really_probe()` 会检查 `dev->driver` 和 `devres_head`, 但如果在 probe 函数中又注册了一个新的 platform_driver, 会触发嵌套 probe 吗？

2. **`-EPROBE_DEFER` 无限循环**：如果 A 依赖 B, B 依赖 A, 两者都返回 `-EPROBE_DEFER`, 内核会无限重试吗？什么时候停止？

3. **Probe 超时**：`really_probe()` 没有超时机制。如果 probe 函数中 `ioremap` 挂死（比如访问了不存在的地址），会怎样？

4. **驱动热拔插**：platform 设备没有物理热拔插能力，但 `/sys/bus/platform/drivers/xxx/unbind` 可以触发 `platform_remove()`。如果 remove 时某个硬件操作失败，应该如何处理？

---

## 相关笔记

- [[bsp-device-model-dtb]] — 阶段二：设备模型 + 设备树
- [[bsp-device-model-platform-bus-deep]] — Platform Bus 源码追溯
- [[bsp-interrupt-concurrency]] — 阶段三：中断处理
- [[bsp-peripheral-drivers]] — 阶段四：外设驱动
- [[MOC-嵌入式Linux]] — 嵌入式 Linux 学习路线 MOC
