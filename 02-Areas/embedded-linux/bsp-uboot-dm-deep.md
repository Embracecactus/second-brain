---
tags:
  - embedded-linux
  - bsp
  - bootloader
  - u-boot
  - driver-model
  - dm
  - uclass
  - rockchip
  - rv1126b
category: embedded-linux
created: 2026-06-23
updated: 2026-06-23
status: active
soc: Rockchip RV1126B
kernel: Linux 6.1.141
parent: bsp-uboot-adaptation
---

# U-Boot Driver Model 深度解析

> **前置笔记**：[[bsp-uboot-adaptation]] — DM 基础 (u-boot,dm-spl 标记, CRU 驱动示例)
>
> **代码来源**：SDK U-Boot 2017.09 (Rockchip 分支)
>
> **核心文件**：`include/dm/device.h`, `include/dm/uclass.h`, `drivers/core/device.c`, `drivers/core/uclass.c`

---

## 一、DM 三驾马车：uclass / driver / udevice

U-Boot DM 的核心是三个数据结构，它们之间的关系类似 OOP 中的 Class / Class Method / Instance：

```
uclass_driver (类定义)          driver (方法实现)            udevice (实例)
─────────────────              ──────────────              ──────────────
UCLASS_CLK                     rockchip_rv1126b_cru        cru@20000000 实例
  .id = UCLASS_CLK               .id = UCLASS_CLK            .driver = &rv1126b_cru
  .ops → clk_ops                 .ops → rv1126b_clk_ops      .uclass = &uclass_clk
  .per_device_auto_size          .priv_auto_alloc_size        .priv = {clk_priv}
                                 .probe = rv1126b_probe       .flags |= ACTIVATED
```

### 1.1 `struct driver` — 驱动方法实现

```c
// include/dm/device.h:241-259
struct driver {
    char *name;                         // 驱动名称, e.g. "rockchip_rv1126b_cru"
    enum uclass_id id;                  // 所属 uclass ID, e.g. UCLASS_CLK
    const struct udevice_id *of_match;  // DTS compatible 匹配表, e.g. {"rockchip,rv1126b-cru", ...}

    // === 生命周期回调 (4 个核心) ===
    int (*bind)(struct udevice *dev);      // 绑定: 创建设备实例, 注册子设备
    int (*probe)(struct udevice *dev);     // 探活: 初始化硬件, 分配资源
    int (*remove)(struct udevice *dev);    // 移除: 关闭硬件, 释放资源
    int (*unbind)(struct udevice *dev);    // 解绑: 销毁设备实例

    // === 数据转换 ===
    int (*ofdata_to_platdata)(struct udevice *dev);  // DTS → platform data

    // === 子设备回调 ===
    int (*child_post_bind)(struct udevice *dev);
    int (*child_pre_probe)(struct udevice *dev);
    int (*child_post_remove)(struct udevice *dev);

    // === 自动内存分配大小 ===
    int priv_auto_alloc_size;             // 自动分配 dev->priv 的大小
    int platdata_auto_alloc_size;         // 自动分配 dev->platdata 的大小
    int per_child_auto_alloc_size;        // 自动分配 child dev->parent_priv 的大小
    int per_child_platdata_auto_alloc_size;

    const void *ops;                      // 驱动操作函数表 (类型由 uclass 定义)
    uint32_t flags;
};
```

### 1.2 `struct udevice` — 设备实例

```c
// include/dm/device.h:131-153
struct udevice {
    const struct driver *driver;    // 指向该设备使用的 driver
    const char *name;               // 设备名 (通常是 DTS node name)
    void *platdata;                 // platform data (从 DTS 解析而来)
    void *parent_platdata;          // 父设备为其分配的平台数据
    void *uclass_platdata;          // uclass 为其分配的平台数据
    ofnode node;                    // DTS 节点引用
    ulong driver_data;              // of_match 表中的匹配数据
    struct udevice *parent;         // 父设备
    void *priv;                     // 私有数据 (驱动实例的上下文)
    struct uclass *uclass;          // 所属 uclass
    void *uclass_priv;              // uclass 为该设备分配的私有数据
    void *parent_priv;              // 父设备为其分配的私有数据
    struct list_head uclass_node;   // uclass 设备链表节点
    struct list_head child_head;    // 子设备链表头
    struct list_head sibling_node;  // 兄弟设备链表节点
    uint32_t flags;                 // DM_FLAG_* 标志
    int req_seq;                    // 请求序列号
    int seq;                        // 分配序列号 (uclass 内唯一)
};
```

### 1.3 `struct uclass_driver` — 类定义

```c
// include/dm/uclass.h:87-106
struct uclass_driver {
    const char *name;
    enum uclass_id id;

    // === uclass 级别回调 ===
    int (*post_bind)(struct udevice *dev);      // 设备绑定后
    int (*pre_unbind)(struct udevice *dev);     // 设备解绑前
    int (*pre_probe)(struct udevice *dev);      // 设备探活前
    int (*post_probe)(struct udevice *dev);     // 设备探活后
    int (*pre_remove)(struct udevice *dev);     // 设备移除前
    int (*child_post_bind)(struct udevice *dev);
    int (*child_pre_probe)(struct udevice *dev);
    int (*init)(struct uclass *class);           // uclass 自身初始化
    int (*destroy)(struct uclass *class);        // uclass 销毁

    // === 自动内存分配 ===
    int priv_auto_alloc_size;
    int per_device_auto_alloc_size;
    int per_device_platdata_auto_alloc_size;
    int per_child_auto_alloc_size;
    int per_child_platdata_auto_alloc_size;

    const void *ops;         // uclass 提供的标准操作接口
    uint32_t flags;
};
```

### 1.4 RV1126B 实例：时钟驱动

以 `drivers/clk/rockchip/clk_rv1126b.c` 为例，展示三者如何协作：

```c
// 1. 定义 of_match 表: DTS → driver 匹配
static const struct udevice_id rv1126b_clk_ids[] = {
    { .compatible = "rockchip,rv1126b-cru", .data = ROCKCHIP_RV1126B_CRU },
    { }
};

// 2. 定义 ops (操作函数表)
static struct clk_ops rv1126b_clk_ops = {
    .get_rate       = rv1126b_clk_get_rate,
    .set_rate       = rv1126b_clk_set_rate,
    .enable         = rv1126b_clk_enable,
    .disable        = rv1126b_clk_disable,
};

// 3. 声明 driver (关键宏)
U_BOOT_DRIVER(rockchip_rv1126b_cru) = {
    .name       = "rockchip_rv1126b_cru",
    .id         = UCLASS_CLK,                 // ← 归属于 UCLASS_CLK
    .of_match   = rv1126b_clk_ids,             // ← DTS compatible 匹配
    .priv_auto_alloc_size = sizeof(struct rv1126b_clk_priv),  // ← 自动分配 dev->priv
    .ops        = &rv1126b_clk_ops,            // ← ops = clk_ops
    .bind       = rv1126b_clk_bind,            // ← 创建 reset 子设备
    .probe      = rv1126b_clk_probe,           // ← 初始化 PLL + 时钟树
};
```

---

## 二、设备生命周期：Bind → Probe → Remove → Unbind

```
┌─────────────────────────────────────────────┐
│ 扫描 DTS: dm_init_and_scan()                │
│   └─ 遍历所有 compatible 节点                │
│        │                                     │
│        ▼                                     │
│  Bind (绑定)                                 │
│   ├─ 分配 struct udevice                     │
│   ├─ 设置 driver, name, node, parent         │
│   ├─ 分配 dev->platdata (如果设置了)          │
│   ├─ 分配 dev->priv (如果设置了)              │
│   ├─ 调用 driver.bind() 回调                  │
│   ├─ 调用 uclass_driver.post_bind()           │
│   └─ 将设备加入 uclass.dev_head 链表          │
│        │                                     │
│        ▼                                     │
│  Probe (探活)                                │
│   ├─ 分配 dev->uclass_priv (如果设置了)        │
│   ├─ 调用 uclass_driver.pre_probe()           │
│   ├─ 调用 driver.ofdata_to_platdata()         │
│   │    ← 读取 DTS reg/interrupts/clocks...   │
│   ├─ 调用 driver.probe()                      │
│   │    ← 初始化硬件: 使能时钟, 配置寄存器     │
│   ├─ 设置 DM_FLAG_ACTIVATED                   │
│   ├─ 分配 seq (uclass 内唯一编号)              │
│   └─ 调用 uclass_driver.post_probe()          │
│        │                                     │
│        ▼                                     │
│  Dev Active! ★                                │
│   ├─ 可通过 uclass_get_device() 访问          │
│   └─ 可通过 device probe 获取子设备           │
│        │                                     │
│  Remove (移除) / Unbind (解绑)                │
│   ├─ driver.remove() → 关闭硬件               │
│   ├─ driver.unbind() → 释放 struct udevice    │
│   └─ 从链表中移除                             │
└─────────────────────────────────────────────┘
```

### 2.1 Bind 阶段 (源码: `drivers/core/device.c`)

```c
// device_bind() 的核心逻辑 (精简)
int device_bind(struct udevice *parent, struct driver *drv,
                const char *name, void *platdata, ofnode node,
                struct udevice **devp)
{
    struct udevice *dev;

    // 1. 分配 udevice 结构体
    dev = calloc(1, sizeof(struct udevice));

    // 2. 填充基本字段
    dev->driver = drv;
    dev->name = name;
    dev->node = node;
    dev->parent = parent;
    dev->flags = DM_FLAG_BOUND;          // 标记已绑定

    // 3. 分配 platdata 内存 (如果 driver 声明了大小)
    if (drv->platdata_auto_alloc_size)
        dev->platdata = alloc_priv(drv->platdata_auto_alloc_size);

    // 4. 分配 priv 内存 (如果 driver 声明了大小)
    if (drv->priv_auto_alloc_size)
        dev->priv = alloc_priv(drv->priv_auto_alloc_size);

    // 5. 调用 driver.bind() 回调
    if (drv->bind)
        drv->bind(dev);                  // ← 例如 rv1126b_clk_bind 创建 reset 子设备

    // 6. 调用 uclass_driver.post_bind()
    uc_drv->post_bind(dev);

    // 7. 将设备加入 uclass 链表
    list_add_tail(&dev->uclass_node, &uc->dev_head);

    *devp = dev;
    return 0;
}
```

### 2.2 Probe 阶段

```c
// device_probe() 的核心逻辑 (精简)
int device_probe(struct udevice *dev)
{
    // 1. 跳过已激活的设备
    if (dev->flags & DM_FLAG_ACTIVATED)
        return 0;

    // 2. 确保父设备已探活 (递归向上)
    if (dev->parent)
        device_probe(dev->parent);

    // 3. 分配 uclass_priv
    if (uc_drv->per_device_auto_alloc_size)
        dev->uclass_priv = alloc_priv(...);

    // 4. pre_probe 回调
    if (uc_drv->pre_probe)
        uc_drv->pre_probe(dev);

    // 5. DTS → platdata 转换
    if (drv->ofdata_to_platdata && dev_has_of_node(dev))
        drv->ofdata_to_platdata(dev);     // ← 解析 reg/interrupts

    // 6. 真正的 probe
    if (drv->probe)
        drv->probe(dev);                  // ← 初始化硬件

    // 7. 标记已激活
    dev->flags |= DM_FLAG_ACTIVATED;

    // 8. 分配 seq (自动编号或 alias 映射)
    dev->seq = uclass_resolve_seq(dev);

    // 9. post_probe 回调
    if (uc_drv->post_probe)
        uc_drv->post_probe(dev);

    return 0;
}
```

### 2.3 ofdata_to_platdata — DTS 到平台数据的桥梁

```c
// 示例: 时钟驱动的 ofdata_to_platdata
// 典型的模式: 从 DTS 的 reg 属性解析寄存器基地址
static int rv1126b_clk_ofdata_to_platdata(struct udevice *dev)
{
    struct rv1126b_clk_priv *priv = dev_get_priv(dev);

    // reg = <0x20000000 0x1000>
    priv->base = dev_read_addr(dev);      // → 0x20000000

    // clock-cells = <1>
    priv->cell_count = dev_read_u32_default(dev, "#clock-cells", 0);

    // assigned-clocks / assigned-clock-parents (DTS 中动态配置)
    priv->assigned = dev_read_assigned_clocks(dev);

    return 0;
}
```

---

## 三、uclass 操作接口

### 3.1 uclass 回调钩子

uclass 的 6 个生命周期钩子形成完整的设备管理链：

```
Bind 阶段:
  uclass.post_bind(dev)   → 分配每个设备需要的 uclass 级资源
                            例如: UCLASS_MMC 中设置默认块大小

Probe 阶段:
  uclass.pre_probe(dev)   → probe 前检查, 例如确保电源已就绪
  uclass.post_probe(dev)  → probe 后初始化, 例如使能中断

Remove 阶段:
  uclass.pre_remove(dev)  → 移除前检查, 例如确保无 pending DMA

Unbind 阶段:
  uclass.pre_unbind(dev)  → 解绑前清除 uclass 级资源
  uclass.child_post_bind  → 子设备绑定后, 父设备 uclass 可获知
```

### 3.2 uclass 操作函数表 ops

`uclass_driver.ops` 定义了该类设备的**标准接口**：

```c
// 示例: UCLASS_CLK 的标准 ops
// drivers/clk/clk-uclass.c
struct clk_ops {
    int (*request)(struct clk *clk);
    int (*get_rate)(struct clk *clk, ulong *rate);
    int (*set_rate)(struct clk *clk, ulong rate);
    int (*enable)(struct clk *clk);
    int (*disable)(struct clk *clk);
};

// uclass 提供的公共 API:
struct clk *clk_get(struct udevice *dev, int index);
int clk_enable(struct clk *clk);
int clk_set_rate(struct clk *clk, ulong rate);

// 实现方式: 通过 clk API 调用 → uclass 查找设备 → 调用 driver.ops
int clk_enable(struct clk *clk)
{
    struct clk_ops *ops = clk_dev_ops(clk->dev);
    return ops->enable(clk);
    // → 最终调用 rv1126b_clk_enable()
}
```

### 3.3 通过 uclass 查找设备

```c
// 按 uclass ID 获取设备 (最常用)
struct udevice *dev;
uclass_get_device(UCLASS_MMC, 0, &dev);    // 第 0 个 MMC 设备
uclass_get_device(UCLASS_CLK, 0, &dev);    // 第 0 个时钟设备
uclass_get_device(UCLASS_I2C, 0, &dev);    // 第 0 个 I2C 设备

// 按名称
uclass_get_device_by_name(UCLASS_ETH, "ethernet@215c0000", &dev);

// 按序列号 (DTS 中的 xxx = <&phandle>)
uclass_get_device_by_seq(UCLASS_UART, 0, &dev);  // 串口 0

// 按 DTS node 引用
uclass_get_device_by_ofnode(UCLASS_GPIO, node, &dev);
```

---

## 四、DM 初始化流程

### 4.1 SPL 阶段 (pre-reloc)

```c
// spl_early_init() → dm_init_and_scan(true)
// true = pre_reloc_only 模式

int dm_init_and_scan(bool pre_reloc_only)
{
    // 1. 初始化 DM 核心
    dm_init();

    // 2. 扫描 DTB 中所有节点
    //    仅扫描标记了 u-boot,dm-pre-reloc 或 u-boot,dm-spl 的节点
    if (pre_reloc_only)
        lists_bind_fdt(gd->dm_root, gd->fdt_blob,
                       DM_FLAG_PRE_RELOC);  // ← 过滤标志

    // 3. 对所有扫描到的设备执行 bind
    foreach (node with pre_reloc flag)
        device_bind(root, matched_driver, ...);
}
```

### 4.2 U-Boot proper 阶段 (post-reloc)

```c
// board_init_r() → initr_dm() → dm_init_and_scan(false)
// false = 扫描所有节点 (不限制 pre_reloc 标记)

int initr_dm(void)
{
    // 此时所有设备都会被 bind:
    // - SPL 阶段已 bind 的设备? 需要重新 bind!
    // - 新设备 (无 pre_reloc 标记) 也被 bind
    dm_init_and_scan(false);

    // SPL 阶段已 probe 的设备不会被 re-probe
    // 而是从 SPL 阶段继承状态
}
```

### 4.3 SPL 标记选择策略

| DTS 标记 | 扫描阶段 | 适用场景 |
|---------|---------|---------|
| `u-boot,dm-spl` | SPL board_init_f | **SPL 必须的设备**: 时钟、eMMC、串口、引脚控制 |
| `u-boot,dm-pre-reloc` | SPL board_init_f 早期 | **重定位前必须**: ADC 按键检测 (spl_hotkey_init) |
| `u-boot,dm-pre-proper` | U-Boot proper 重定位前 | **proper 早期**: 需要但 SPL 不需要的外设 |
| 无标记 | U-Boot proper board_init_r | **正常设备**: 大部分驱动在此阶段初始化 |

RV1126B 标记示例 (`arch/arm/dts/rv1126b-u-boot.dtsi`):

```dts
&cru      { u-boot,dm-spl; status = "okay"; };    // SPL 必需的时钟
&grf      { u-boot,dm-spl; };                       // 寄存器映射
&uart0    { u-boot,dm-spl; };                       // SPL 串口输出
&emmc     { u-boot,dm-spl; };                       // SPL 从 eMMC 加载 FIT
&pinctrl  { u-boot,dm-spl; };                       // 引脚配置
&saradc0  { u-boot,dm-pre-reloc; };                 // ADC 按键 (重定位前)
```

---

## 五、多级设备树与 Bus 模型

### 5.1 Parent-Child 层级

```
root (UCLASS_ROOT)
  │
  ├─ simple-bus (UCLASS_SIMPLE_BUS)     ← 对应 DTS 根节点 "/"
  │    │
  │    ├─ cru@2000000
  │    │    └─ reset-controller          ← rv1126b_clk_bind 创建的子设备
  │    │
  │    ├─ i2c@2030000 (UCLASS_I2C)
  │    │    ├─ pmic@1b (UCLASS_PMIC)    ← 从 DTS 中 child 节点探测
  │    │    └─ rtc@51  (UCLASS_RTC)
  │    │
  │    └─ dwmmc@2030000 (UCLASS_MMC)    ← eMMC 控制器
  │         └─ blk (UCLASS_BLK)         ← 块设备自动创建
  │
  └─ pinctrl (UCLASS_PINCTRL)
```

### 5.2 Bus 模型: I2C 实例

```c
// I2C 总线的 DM 模型:
// 1. I2C 控制器 controller (如 rk8xx_i2c) 继承自 UCLASS_I2C
// 2. I2C 上的从设备 (如 PMIC RK801) 继承自 UCLASS_PMIC

U_BOOT_DRIVER(rockchip_rv1126b_i2c) = {
    .name   = "rockchip_rv1126b_i2c",
    .id     = UCLASS_I2C,
    .probe  = rv1126b_i2c_probe,        // 初始化 I2C 控制器
};

// 从设备: 子节点自动匹配 UCLASS_PMIC 驱动
U_BOOT_DRIVER(rk801_pmic) = {
    .name   = "rk801_pmic",
    .id     = UCLASS_PMIC,
    .probe  = rk801_probe,              // 初始化 PMIC
};

// I2C 总线的 child_pre_probe 会在探测子设备前
// 设置正确的 I2C 从地址 (从 DTS reg 属性读取)
```

---

## 六、DM 调试方法

### 6.1 U-Boot shell 调试命令

```bash
# 列出所有已绑定的设备 (按 uclass 分组)
=> dm tree
# 输出:
#  Class     Probed   Name
#  ----------------------------------------
#  root      [ + ]    root_driver
#  simple_bus [ + ]   |-- soc
#  clk       [ + ]    |   |-- cru@20000000
#  reset     [ + ]    |   |   |-- reset-controller
#  mmc       [ + ]    |   |-- dwmmc@20300000
#  i2c       [ + ]    |   |-- i2c@20310000
#  pmic      [ + ]    |   |-- |-- pmic@1b
#  ...
#  [ + ] = probed, [ ] = bound only

# 查看特定 uclass 的所有设备
=> uclass mmc
# 输出: 所有 UCLASS_MMC 设备及其状态

# 查看设备信息
=> devinfo cru@20000000
# 输出: 设备驱动名, 父设备, uclass, 寄存器地址, 状态

# 查看 dm 内存使用
=> dm mem
# 输出: DM 数据结构占用的内存统计
```

### 6.2 RV1126B 实际 DM 树

```bash
# 在 sportcam 板端 U-Boot shell 中执行:
=> dm tree
# 预期输出结构:
Class           Probed  Name
----------------------------------------
root            [ + ]   root_driver
simple_bus      [ + ]   |-- soc
clk             [ + ]   |   |-- xin24m
clk             [ + ]   |   |-- cru@20000000
reset           [ + ]   |   |   |-- reset-controller
pinctrl         [ + ]   |   |-- pinctrl
serial          [ + ]   |   |-- serial@20810000
mmc             [ + ]   |   |-- dwmmc@20300000
blk             [ + ]   |   |   |-- dwmmc@20300000.blk
i2c             [ + ]   |   |-- i2c@20310000
pmic            [ + ]   |   |   |-- rk801@1b
regulator       [ + ]   |   |   |   |-- DCDC_REG1
regulator       [ + ]   |   |   |   |-- DCDC_REG2
...
```

### 6.3 驱动 probe 失败调试

```bash
# 场景: 某个驱动 probe 返回错误, 系统挂起

# 方法 1: 逐设备手动 probe
=> probe cru@20000000
# 输出: 如果成功无输出, 失败返回错误码

# 方法 2: 查看 DM 状态
=> dm tree | grep -v "\[ + \]"
# 只显示未 probe 的设备

# 方法 3: 检查 DTS 节点是否匹配
# 查看 compatible 字符串
=> fdt list /soc/cru@20000000
# compatible = "rockchip,rv1126b-cru"
# 检查驱动是否声明了相同 compatible

# 方法 4: 启动时增加 DM 调试
# 在 defconfig 中:
# CONFIG_DM_WARN=y            # DM 警告输出
# CONFIG_DM_STATS=y           # DM 统计信息
```

---

## 七、实践：写一个最简单的 DM 驱动

```c
// === step_demo.c: 一个完整的 U-Boot DM 驱动 ===

#include <common.h>
#include <dm.h>
#include <dm/device.h>

// 私有数据结构
struct step_priv {
    unsigned long base_addr;
    int enabled;
};

// DTS compatible 匹配表
static const struct udevice_id step_demo_ids[] = {
    { .compatible = "demo,step-driver" },
    { }
};

// probe 函数: DTS 解析 + 硬件初始化
static int step_demo_probe(struct udevice *dev)
{
    struct step_priv *priv = dev_get_priv(dev);

    // 从 DTS 解析 reg 属性
    priv->base_addr = dev_read_addr(dev);
    printf("Step-Demo: probed at 0x%lx\n", priv->base_addr);

    // 读取自定义属性
    priv->enabled = dev_read_bool(dev, "demo,enabled");

    return 0;
}

// 驱动操作函数 (可选, 取决于 uclass)
static const struct step_ops {
    void (*ping)(struct udevice *dev);
} step_ops = {
    .ping = NULL,
};

// 声明驱动: 关键宏!
U_BOOT_DRIVER(step_demo) = {
    .name      = "step_demo",
    .id        = UCLASS_MISC,             // 归入 Miscellaneous 类
    .of_match  = step_demo_ids,
    .probe     = step_demo_probe,
    .priv_auto_alloc_size = sizeof(struct step_priv),  // 自动分配
    .ops       = &step_ops,
};
```

DTS 节点:

```dts
// arch/arm/dts/rv1126b-alientek.dts
/ {
    step-demo {
        compatible = "demo,step-driver";
        reg = <0x0 0x20810000 0x0 0x1000>;
        demo,enabled;
    };
};
```

编译后查看:

```bash
=> dm tree | grep step
# misc   [ + ]   |-- step-demo

=> probe step-demo
# 输出: Step-Demo: probed at 0x20810000
```

---

## 八、U_BOOT_DRIVER 宏的魔法

`U_BOOT_DRIVER` 宏利用 GCC 的链接器段 (linker section) 将所有驱动声明**自动注册**：

```c
// 展开前:
U_BOOT_DRIVER(rockchip_rv1126b_cru) = { ... };

// 展开后 (通过 ll_entry_declare):
struct driver _u_boot_list_driver_rockchip_rv1126b_cru
    __attribute__((section(".u_boot_list_2_driver_2_rockchip_rv1126b_cru")))
    = { ... };

// 启动时, DM 扫描 .u_boot_list_2_driver_* 段:
// → 得到所有驱动的数组
// → 遍历 DTS 节点, 逐个匹配 .of_match
// → 找到匹配的 → device_bind()
```

这个机制意味着一件事：**不需要手动 register_driver()**，只需要声明 `U_BOOT_DRIVER(...)` 宏，编译时自动注册。

---

## 九、思考题

1. **Bind vs Probe**: 为什么要设计 Bind 和 Probe 两个独立的阶段？合并为一个阶段会有什么问题？

2. **自动内存分配**: `priv_auto_alloc_size` 和 `per_child_auto_alloc_size` 由谁释放？内存泄露的可能场景是什么？

3. **of_match 匹配顺序**: 如果一个 DTS 节点有多个 compatible 字符串（如 `"mycompany,mydev", "rockchip,generic-dev"`），U-Boot 如何选择匹配的 driver？

4. **uclass 的 ops vs driver 的 ops**: 两者都是操作函数表，它们各自的用途是什么？一个 driver 是否可以同时提供 uclass 标准 ops 和自己扩展的 ops？

5. **DM vs Linux 设备模型**: U-Boot 的 driver/udevice/uclass 分别对应 Linux 设备模型的哪些概念？两者为什么有这样的差异？

---

## 相关笔记

- [[bsp-uboot-adaptation]] — DM 基础 (SPL 标记, CRU 驱动)
- [[bsp-device-model-dtb]] — 阶段二: Linux 设备模型 + 设备树
- [[bsp-boot-flow]] — U-Boot 命令行交互
- [[MOC-嵌入式Linux]] — 嵌入式 Linux 学习路线 MOC
