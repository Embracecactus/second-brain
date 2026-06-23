---
tags:
  - embedded-linux
  - bsp
  - device-model
  - sysfs
  - kobject
  - kset
  - uevent
  - kernel-core
  - source-code
category: embedded-linux
created: 2026-06-23
updated: 2026-06-23
status: active
kernel: Linux 6.1.141
parent: bsp-device-model-dtb
---

# sysfs / kobject 机制源码追溯

> **前置笔记**：[[bsp-device-model-dtb]] — 阶段二：设备模型 + 设备树
>
> **核心文件**：`lib/kobject.c`, `include/linux/kobject.h`, `include/linux/kernfs.h`, `fs/sysfs/`, `drivers/base/core.c`

---

## 一、kobject 核心

### 1.1 为什么需要 kobject

```
Linux 设备模型的核心抽象:

  struct device  ── 嵌入 ──→  struct kobject  ── 对应 ──→  /sys/ 目录
      ↑                         ↑
  platform_device            kset (同类 kobject 集合)
  i2c_client                    ↑
  spi_device                 bus_type, class, ...

kobject 提供了:
  - 引用计数 (kref) — 安全生命周期管理
  - 层次结构 (parent) — /sys 目录树
  - 名称管理 (name) — sysfs 目录名
  - 属性接口 (ktype->sysfs_ops) — show/store
  - 热插拔事件 (uevent) — 通知 udev
```

### 1.2 `struct kobject` 完整定义

```c
// include/linux/kobject.h:64
struct kobject {
    const char          *name;        // 目录名 (如 "platform")
    struct list_head    entry;        // 链表节点 (挂在 kset->list)
    struct kobject      *parent;      // 父 kobject (/sys 中的父目录)
    struct kset         *kset;        // 所属的 kset (同类分组)
    const struct kobj_type *ktype;    // 类型 (release, sysfs_ops, default_groups)
    struct kernfs_node  *sd;          // ★ kernfs 内部节点 (sysfs 目录项)
    struct kref         kref;         // 引用计数
    unsigned int state_initialized:1;
    unsigned int state_in_sysfs:1;    // 是否已添加到 sysfs
    unsigned int state_add_uevent_sent:1;
    unsigned int state_remove_uevent_sent:1;
    unsigned int uevent_suppress:1;   // 抑制 uevent
};
```

### 1.3 `struct kobj_type` — 行为定义

```c
// include/linux/kobject.h:120
struct kobj_type {
    void (*release)(struct kobject *kobj);
    // ★ kobject_put() 引用归零时调用, 释放对象

    const struct sysfs_ops *sysfs_ops;
    // ★ show/store 回调: 读写 sysfs 文件的入口

    const struct attribute_group **default_groups;
    // ★ 创建时自动生成的 sysfs 属性文件
};

// 设备模型的 ktype:
struct kobj_type device_ktype = {
    .release       = device_release,
    .sysfs_ops     = &dev_sysfs_ops,
    .default_groups = device_default_groups,
};
```

---

## 二、kobject 生命周期

### 2.1 创建

```c
// ===== 方式 1: 分开初始化 + 添加 =====
struct kobject *kobj;

kobject_init(kobj, &my_ktype);
// → kref_init(&kobj->kref)    ← 引用计数 = 1
// → state_initialized = 1

kobject_add(kobj, parent_kobj, "%s", "my_name");
// → kobject_add_varg(kobj, parent, fmt, args)
//   → kobject_set_name(kobj, fmt, ...) ← 保存 name
//   → kobject_add_internal(kobj)
//     → create_dir(kobj)
//       → sysfs_create_dir_ns(kobj, ns)  ← ★ 创建 /sys/.../my_name/
//       → sysfs_create_groups(kobj, default_groups)

// ===== 方式 2: 一步完成 (推荐) =====
kobject_init_and_add(kobj, &my_ktype, parent, "%s", name);
// → 等价于 kobject_init + kobject_add
// → 出错时需要 kobject_put() 释放
```

### 2.2 引用计数

```c
// 引用计数管理:
kobject_get(kobj)        // 递增, 返回 kobj
kobject_put(kobj)        // 递减, 归零时调用 ktype->release
kobject_get_unless_zero(kobj)  // 不为 0 才递增

// get/put 配对:
// kobject_add 内部会调用 kobject_get(parent) 保持父对象
// → 子对象释放时 kobject_put(parent) 递减父对象计数
```

### 2.3 销毁

```c
void kobject_put(struct kobject *kobj)
{
    if (!kobj) return;

    // ★ 递减引用计数
    if (kref_put(&kobj->kref, kobject_release)) {
        // 引用归零时调用 kobject_release()
    }
}

static void kobject_release(struct kref *kref)
{
    struct kobject *kobj = container_of(kref, struct kobject, kref);

    // 延迟释放 (debug 模式)
    if (CONFIG_DEBUG_KOBJECT_RELEASE)
        schedule_delayed_work(&kobj->release, HZ);

    // → kobject_cleanup()
    //   → kobj_type->release(kobj)      ← ★ 你的释放函数
    //   → kfree(kobj->name)
    //   → kfree(kobj)
}
```

---

## 三、kobject → sysfs 目录映射

### 3.1 `create_dir()` — sysfs 目录创建

```c
// lib/kobject.c:57
static int create_dir(struct kobject *kobj)
{
    // 1. ★ 创建 kernfs 目录节点
    error = sysfs_create_dir_ns(kobj, kobject_namespace(kobj));
    //   → kobj->sd 指向新创建的 kernfs_node

    // 2. ★ 创建默认属性文件
    if (ktype = get_ktype(kobj)) {
        error = sysfs_create_groups(kobj, ktype->default_groups);
        //  → 在 kobj->sd 目录下创建默认文件

        // 每个 ktype 的 default_groups 决定了
        // 这个 kobject 有哪些 sysfs 文件
        // 如 device_ktype: uevent, modalias, ... (device_default_groups)
    }

    kobj->state_in_sysfs = 1;
}
```

### 3.2 设备模型的 sysfs 结构

```
/sys/                          ← sysfs 根
├── devices/                   ← kobject: system_kobj
│   └── platform/              ← kobject: platform_bus
│       ├── ff3e0000.i2c/      ← struct device 的 kobject
│       │   ├── driver → ../../bus/platform/drivers/i2c_rk3x
│       │   ├── modalias
│       │   ├── uevent
│       │   ├── driver_override
│       │   └── ...
│       └── ff3d0000.spi/
├── bus/                       ← kobject: system_kset
│   └── platform/
│       ├── devices/           ← 软链接到 devices/platform/xxx
│       └── drivers/
│           └── i2c_rk3x/
│               ├── ff3e0000.i2c → ../../../devices/platform/ff3e0000.i2c
│               ├── bind
│               ├── uevent
│               ├── unbind
│               └── ...
├── class/
├── kernel/
└── ...

每个目录 = 一个 kobject
每个文件 = 一个 attribute 通过 sysfs_ops 的 show/store 读写
```

### 3.3 `struct device` 的 kobject 初始化

```c
// drivers/base/core.c:3156
void device_initialize(struct device *dev)
{
    // ★ 设备的内嵌 kobject 使用 device_ktype
    kobject_init(&dev->kobj, &device_ktype);

    // 初始化设备的各种链表
    INIT_LIST_HEAD(&dev->dma_poll_list);
    ...
}

// device_add() 中:
int device_add(struct device *dev)
{
    // 1. 设置设备名 (如 "ff3e0000.i2c")
    dev_set_name(dev, "%s", ...);

    // 2. ★ 添加 kobject (创建 /sys/devices/... 目录)
    error = kobject_add(&dev->kobj, dev->kobj.parent, "%s", ...);

    // 3. 创建设备特定的属性
    device_create_file(dev, &dev_attr_uevent);
    device_create_file(dev, &dev_attr_online);
    ...

    // 4. ★ 发送添加 uevent
    kobject_uevent(&dev->kobj, KOBJ_ADD);
}
```

---

## 四、sysfs 属性文件

### 4.1 `DEVICE_ATTR` 宏展开

```c
// include/linux/device.h:127
#define DEVICE_ATTR(_name, _mode, _show, _store) \
    struct device_attribute dev_attr_##_name = \
        __ATTR(_name, _mode, _show, _store)

// 使用:
static ssize_t my_show(struct device *dev,
                       struct device_attribute *attr, char *buf)
{
    return sysfs_emit(buf, "%d\n", my_value);
}

static ssize_t my_store(struct device *dev,
                        struct device_attribute *attr,
                        const char *buf, size_t count)
{
    sscanf(buf, "%d", &my_value);
    return count;
}

static DEVICE_ATTR(my_value, 0644, my_show, my_store);
// 展开为:
//   static struct device_attribute dev_attr_my_value = {
//       .attr = { .name = "my_value", .mode = 0644 },
//       .show = my_show,
//       .store = my_store,
//   };

// 注册到 sysfs:
device_create_file(dev, &dev_attr_my_value);
// → /sys/devices/platform/xxx/my_value
```

### 4.2 `sysfs_ops` — show/store 分发

```c
// drivers/base/core.c
static ssize_t dev_sysfs_show(struct kobject *kobj,
                               struct attribute *attr, char *buf)
{
    struct device_attribute *dev_attr = to_dev_attr(attr);
    struct device *dev = kobj_to_dev(kobj);

    if (dev_attr->show)
        return dev_attr->show(dev, dev_attr, buf);
    return -EIO;
}

const struct sysfs_ops dev_sysfs_ops = {
    .show  = dev_sysfs_show,
    .store = dev_sysfs_store,
};
```

### 4.3 读写路径

```
读操作:
  cat /sys/devices/platform/xxx/my_value
    ↓
  sysfs_kf_read()           → fs/sysfs/file.c
    ↓
  kernfs_file_read_iter()   → fs/kernfs/file.c
    ↓
  kernfs_ops->read()        → kernfs 操作
    = sysfs_ops->show(kobj, attr, buf)  ★ 关键跳转
      ↓
  dev_sysfs_show(kobj, attr, buf)
    → dev_attr->show(dev, dev_attr, buf)
      → my_show(dev, attr, buf)         ← ★ 你的 show 函数!
```

---

## 五、kset — kobject 分组

### 5.1 `struct kset`

```c
// include/linux/kobject.h:172
struct kset {
    struct list_head list;          // ★ 所有成员 kobject 的链表
    spinlock_t list_lock;
    struct kobject kobj;            // ★ 内嵌 kobject (自身也创建目录)
    const struct kset_uevent_ops *uevent_ops;  // uevent 过滤/扩展
};

// 例子:
// bus_type 内嵌 kset:
//   bus_type.p->subsys = kset
//   → /sys/bus/platform/ 目录
//
// class 内嵌 kset:
//   class.p->class_dirs = kset
//   → /sys/class/video4linux/ 目录
```

### 5.2 kset 的 uevent 控制

```c
struct kset_uevent_ops {
    int (*filter)(struct kobject *kobj);
    // 返回 0 = 过滤掉这个 uevent, 不发

    const char *(*name)(struct kobject *kobj);
    // 修改 uevent 的 SUBSYSTEM= 值

    int (*uevent)(struct kobject *kobj, struct kobj_uevent_env *env);
    // ★ 添加自定义环境变量
};

// platform_bus_type 的 kset 在 bus_type 内部管理
// bus_uevent_filter / bus_uevent 等
```

---

## 六、uevent — 内核 → 用户空间通知

### 6.1 原理

```
kobject_uevent(dev->kobj, KOBJ_ADD)
  ↓
kobject_uevent_env(kobj, action, NULL)
  ↓
// 1. 构造环境变量数组
// 2. 通过 kset->uevent_ops 过滤/扩展
// 3. 通过 netlink (NETLINK_KOBJECT_UEVENT) 发送
// 4. 用户空间 udev / mdev 接收
```

### 6.2 `kobject_uevent_env()` 核心

```c
// lib/kobject_uevent.c
int kobject_uevent_env(struct kobject *kobj, enum kobject_action action,
                       char *envp_ext[])
{
    struct kobj_uevent_env *env;
    const struct kset_uevent_ops *uevent_ops;

    // 1. 检查是否抑制 uevent
    if (kobj->uevent_suppress) return 0;

    // 2. 找到 kset 的 uevent_ops
    kset = kobj->kset;
    uevent_ops = kset->uevent_ops;

    // 3. 过滤 (filter 返回 0 则不发送)
    if (uevent_ops && uevent_ops->filter)
        if (!uevent_ops->filter(kobj)) return 0;

    // 4. 构造环境变量
    env = kzalloc(sizeof(struct kobj_uevent_env), GFP_KERNEL);

    // ★ 标准变量:
    add_uevent_var(env, "ACTION=%s", action_string);   // "add"/"remove"
    add_uevent_var(env, "DEVPATH=%s", devpath);         // /devices/platform/xxx
    add_uevent_var(env, "SUBSYSTEM=%s", subsystem);     // "platform"
    add_uevent_var(env, "SEQNUM=%llu", ++uevent_seqnum);

    // ★ kset 自定义变量:
    if (uevent_ops && uevent_ops->uevent)
        uevent_ops->uevent(kobj, env);
    //   → bus_uevent() → add_uevent_var("MODALIAS=platform:xxx")

    // ★ 调用者扩展变量:
    if (envp_ext)
        for (i = 0; envp_ext[i]; i++)
            add_uevent_var(env, envp_ext[i]);

    // 5. ★ 通过 NETLINK 发送
    kobject_uevent_net_broadcast(kobj, env, action_string);

    // 6. 老式 uevent_helper (echo /sbin/hotplug)
    //    → 默认禁用
}
```

### 6.3 用户空间接收

```bash
# udevadm 监听:
udevadm monitor --property
# 输出示例:
# KERNEL[123.456] add      /devices/platform/ff3e0000.i2c (platform)
# ACTION=add
# DEVPATH=/devices/platform/ff3e0000.i2c
# SUBSYSTEM=platform
# MODALIAS=platform:rk3308-i2c
# SEQNUM=1234

# udev 规则: /etc/udev/rules.d/
# 匹配 DEVPATH / SUBSYSTEM / MODALIAS, 执行动作
```

---

## 七、Platform Bus sysfs 集成

### 7.1 `driver_sysfs_add()` — Probe 中的 sysfs 创建

```c
// drivers/base/dd.c:434
static int driver_sysfs_add(struct device *dev)
{
    // ★ 创建两个软链接:

    // 1. /sys/bus/platform/drivers/xxx/device_name → ../../devices/.../device_name
    sysfs_create_link(&dev->driver->p->kobj, &dev->kobj,
                      kobject_name(&dev->kobj));

    // 2. /sys/devices/platform/xxx/driver → ../../bus/platform/drivers/xxx
    sysfs_create_link(&dev->kobj, &dev->driver->p->kobj, "driver");
}
```

### 7.2 `driver_register` → sysfs

```c
// bus_add_driver() 中:
kobject_init_and_add(&priv->kobj, &driver_ktype, ...);
// → /sys/bus/platform/drivers/i2c_rk3x/

sysfs_create_file(&priv->kobj, &driver_attr_uevent);
sysfs_create_file(&priv->kobj, &driver_attr_bind);
sysfs_create_file(&priv->kobj, &driver_attr_unbind);
```

---

## 八、完整流程：设备注册到 sysfs

```
device_add(dev)
  ↓
device_initialize(dev)         ← kobject_init(&dev->kobj, &device_ktype)
  ↓
dev_set_name(dev, "ff3e0000.i2c")
  ↓
kobject_add(&dev->kobj)        ← /sys/devices/platform/ff3e0000.i2c/
  ├── kobject_add_varg()
  │   → kobject_set_name()
  │   → kobject_add_internal()
  │     → create_dir()
  │       → sysfs_create_dir_ns()     ← kernfs mkdir
  │       → sysfs_create_groups()     ← uevent, modalias, ...
  ↓
device_add_attrs(dev)          ← 设备类型和类的属性
  ↓
kobject_uevent(KOBJ_ADD)       ← 通知 udev
  → 构造 ACTION=add, DEVPATH=...
  → NETLINK broadcast
  → udevd 接收 → 加载模块
  ↓
driver_attach(drv)             ← 匹配驱动
  ↓ probe 成功
driver_sysfs_add(dev)
  → /sys/bus/platform/drivers/xxx/ → 软链接
  → /sys/devices/.../driver → 软链接
```

---

## 九、思考题

1. **引用计数循环**：kobject 通过 parent 指针持有父对象的引用。如果父对象持有一个指向子对象的指针（但不通过 kref），会形成循环引用吗？内核如何防止 kobject 内存泄漏？

2. **uevent_suppress 用途**：系统启动早期（initramfs 阶段），uevent 发送可能导致问题。内核在哪个阶段抑制 uevent？什么时候重新开启？

3. **`state_in_sysfs` 检查**：`kobject_del()` 和 `kobject_put()` 中为什么要检查 `state_in_sysfs`？如果忘记调用 `kobject_del()` 直接 `kobject_put()` 会怎样？

4. **sysfs vs debugfs**：sysfs 适合展示"一个设备一个文件"的结构化属性（如 `cat /sys/class/thermal/thermal_zone0/temp`），而 debugfs 适合调试信息。如果一个驱动需要导出一个大数据量的调试日志，应该用哪个？为什么？

---

## 相关笔记

- [[bsp-device-model-dtb]] — 阶段二：设备模型 + 设备树
- [[bsp-device-model-platform-bus-deep]] — Platform Bus 源码追溯
- [[bsp-device-model-probe-deep]] — Driver Probe 全路径
- [[bsp-device-model-dtb-unflatten-deep]] — DTB Unflatten 深入
- [[MOC-嵌入式Linux]] — 嵌入式 Linux 学习路线 MOC
