---
tags:
  - embedded-linux
  - bsp
  - device-tree
  - dtb
  - unflatten
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

# DTB Unflatten 深入 — 从二进制到设备树

> **前置笔记**：[[bsp-device-model-dtb]] — 阶段二：设备模型 + 设备树
>
> **核心文件**：`drivers/of/fdt.c`, `drivers/of/base.c`, `drivers/of/dynamic.c`, `include/linux/of.h`

---

## 一、什么是 Unflatten

```
启动时内存布局:

  DTB (Flattened)                    Device Tree (Unflattened)
  ┌─────────────────┐               ┌──────────────────────────┐
  │  struct fdt_header│               │  struct device_node "/"  │
  │  magic = 0xd00dfeed│               │  ├── child: "chosen"    │
  │  totalsize = 119KB│               │  ├── child: "cpus"      │
  │  ...              │               │  │   └── child: "cpu@0" │
  ├─────────────────┤  unflatten    │  ├── child: "i2c@ff3e..."│
  │  FDT struct      │ ────────────→  │  │   └── property:      │
  │  (node/property)  │               │  │       reg, clocks,    │
  │  紧凑二进制格式   │               │  │       interrupts, ... │
  └─────────────────┘               │  └── ...                  │
                                    └──────────────────────────┘
  特点: 线性、可移动                  特点: 链表、可动态修改
  优点: 占内存少 (~120KB)             优点: 运行时方便遍历和修改
```

**DTB 格式**是扁平的（Flattened），所有节点以 token 分隔线性排列。
**Unflatten** 将其展开为内核便于操作的 `struct device_node` 树结构（链表）。

---

## 二、DTB 二进制格式

### 2.1 DTB 在内存中的布局

```
DTB 内存布局 (total ~120KB for RV1126B):

┌──────────────────────────────────────────────┐
│ struct fdt_header (40 bytes)                  │
│  - magic          = 0xD00DFEED               │
│  - totalsize      = 0x0001D000 (118784 bytes) │
│  - off_dt_struct  = 0x00000038               │
│  - off_dt_strings = 0x0001F740               │
│  - version        = 17                        │
├──────────────────────────────────────────────┤
│ 填充 / 保留                                    │
├──────────────────────────────────────────────┤
│ FDT Structure Block                          │
│  (节点和属性, token 分隔)                      │
│  FDT_BEGIN_NODE (0x00000001)                  │
│    node name + unit name                      │
│    FDT_PROPERTY (0x00000003)                  │
│      len + nameoff + value                    │
│    ...                                        │
│    FDT_BEGIN_NODE                              │
│      ...                                      │
│    FDT_END_NODE (0x00000002)                  │
│  FDT_END_NODE                                  │
├──────────────────────────────────────────────┤
│ FDT Strings Block                             │
│  "compatible\0status\0reg\0interrupts\0..."    │
│  ← 所有属性名的字符串池, 避免重复               │
└──────────────────────────────────────────────┘
```

### 2.2 DTB Header 结构

```c
struct fdt_header {
    uint32_t magic;           // = 0xD00DFEED (FDT magic)
    uint32_t totalsize;       // DTB 总大小 (bytes)
    uint32_t off_dt_struct;   // FDT Structure Block 偏移
    uint32_t off_dt_strings;  // FDT Strings Block 偏移
    uint32_t off_mem_rsvmap;  // 保留内存映射偏移
    uint32_t version;         // 格式版本 (17 for v17)
    uint32_t last_comp_version; // 兼容的最低版本
    uint32_t boot_cpuid_phys; // 启动 CPU 的物理 ID
    uint32_t size_dt_strings; // 字符串块大小
    uint32_t size_dt_struct;  // 结构块大小
};
```

### 2.3 FDT Token

| Token | 值 | 含义 |
|-------|-----|------|
| `FDT_BEGIN_NODE` | 0x00000001 | 开始一个节点 |
| `FDT_END_NODE` | 0x00000002 | 结束一个节点 |
| `FDT_PROP` | 0x00000003 | 一个属性 |
| `FDT_NOP` | 0x00000004 | 空操作 (填充/禁用) |
| `FDT_END` | 0x00000009 | DTB 结束 |

---

## 三、Unflatten 两阶段算法

### 3.1 调用入口

```c
// start_kernel → setup_arch → unflatten_device_tree()

// arch/arm64/kernel/setup.c 中:
unflatten_device_tree();

// drivers/of/fdt.c:1327
void __init unflatten_device_tree(void)
{
    // initial_boot_params 指向 DTB (由 bootloader 传递)
    __unflatten_device_tree(initial_boot_params, NULL, &of_root,
                            early_init_dt_alloc_memory_arch, false);

    // 解析 /chosen 和 /aliases
    of_alias_scan(early_init_dt_alloc_memory_arch);
}
```

### 3.2 `__unflatten_device_tree()` — 核心框架

```c
// drivers/of/fdt.c:365
void *__unflatten_device_tree(const void *blob, struct device_node *dad,
                              struct device_node **mynodes,
                              void *(*dt_alloc)(u64 size, u64 align),
                              bool detached)
{
    // ===== Phase 1: 扫描 (Dry Run) =====
    // 只计算需要的内存大小, 不实际分配
    size = unflatten_dt_nodes(blob, NULL, dad, NULL);
    //                        ↑  base=NULL → dryrun=true

    size = ALIGN(size, 4);
    mem = dt_alloc(size + 4, __alignof__(struct device_node));
    //   → early_init_dt_alloc_memory_arch()
    //   → memblock_alloc() 从连续物理内存分配

    // ===== Phase 2: 实际展开 =====
    ret = unflatten_dt_nodes(blob, mem, dad, mynodes);
    //                        ↑  base=mem → 分配内存并填充

    if (detached && mynodes && *mynodes)
        of_node_set_flag(*mynodes, OF_DETACHED);
}
```

### 3.3 `unflatten_dt_nodes()` — 遍历 FDT

```c
// drivers/of/fdt.c:284
static int unflatten_dt_nodes(const void *blob, void *mem,
                              struct device_node *dad,
                              struct device_node **nodepp)
{
    // nps[]: 深度优先遍历的栈, 记录每一层当前的 device_node
    struct device_node *nps[FDT_MAX_DEPTH];  // 最大深度 64
    void *base = mem;
    bool dryrun = !base;     // mem==NULL → 第一遍 (dry run)

    nps[depth] = dad;        // 初始: 父节点

    // 关键: 用 fdt_next_node() 遍历 FDT 的所有节点
    for (offset = 0;
         offset >= 0 && depth >= initial_depth;
         offset = fdt_next_node(blob, offset, &depth)) {

        // ★ 对每个节点调用 populate_node()
        ret = populate_node(blob, offset, &mem,
                            nps[depth],    // 父节点
                            &nps[depth+1], // 输出: 子节点
                            dryrun);
    }

    // ★ 反转子节点链表 (保持 DTS 中的节点顺序)
    if (!dryrun)
        reverse_nodes(root);

    return mem - base;  // 返回消耗的总内存
}
```

### 3.4 `populate_node()` — 创建单个节点

```c
// drivers/of/fdt.c:205
static int populate_node(const void *blob, int offset, void **mem,
                         struct device_node *dad,
                         struct device_node **pnp, bool dryrun)
{
    // 1. 获取节点名 (如 "i2c@ff3e0000")
    pathp = fdt_get_name(blob, offset, &len);
    len++;   // +1 for '\0'

    // 2. 分配 device_node + full_name 空间
    //    两者在同一块内存中连续存放
    np = unflatten_dt_alloc(mem, sizeof(struct device_node) + len,
                            __alignof__(struct device_node));

    if (!dryrun) {
        // 3. 初始化 device_node
        of_node_init(np);
        //   → kobject_init(&np->kobj, &of_node_ktype)
        //   → fwnode_init(&np->fwnode, &of_fwnode_ops)

        // 4. 设置 full_name (指向自身末尾)
        np->full_name = (char *)np + sizeof(*np);
        memcpy(np->full_name, pathp, len);

        // 5. ★ 构建树结构
        np->parent  = dad;          // 指向父节点
        np->sibling = dad->child;   // 成为父节点的第一个子节点
        dad->child  = np;           // (后续 reverse_nodes 反转)
    }

    // 6. 解析节点的所有属性
    populate_properties(blob, offset, mem, np, pathp, dryrun);

    if (!dryrun) {
        // 7. 设置 name 字段 (从"name"属性或从节点名解析)
        np->name = of_get_property(np, "name", NULL);
        if (!np->name)
            np->name = "<NULL>";
    }

    *pnp = np;
}
```

### 3.5 `populate_properties()` — 解析属性

```c
// drivers/of/fdt.c:108
static void populate_properties(const void *blob, int offset, void **mem,
                                struct device_node *np,
                                const char *nodename, bool dryrun)
{
    // 遍历 FDT 中当前节点的所有属性
    for (cur = fdt_first_property_offset(blob, offset);
         cur >= 0;
         cur = fdt_next_property_offset(blob, cur)) {

        // 1. 读取属性原始数据
        val = fdt_getprop_by_offset(blob, cur, &pname, &sz);

        // 2. 分配 struct property
        pp = unflatten_dt_alloc(mem, sizeof(struct property), ...);

        if (!dryrun) {
            // 3. 特殊处理 phandle
            if (!strcmp(pname, "phandle") || !strcmp(pname, "linux,phandle"))
                np->phandle = be32_to_cpup(val);

            // 4. ★ 填充 property 字段
            pp->name   = (char *)pname;   // 指向 FDT 字符串池
            pp->length = sz;               // 值长度
            pp->value  = (__be32 *)val;    // 指向原始数据
            // 注意: value 直接指向 DTB 中的原始数据, 不复制!

            // 5. 加入链表
            *pprev = pp;
            pprev  = &pp->next;
        }
    }

    // 如果没有 "name" 属性, 从节点名自动生成
    // "i2c@ff3e0000" → name = "i2c"
}
```

### 3.6 `reverse_nodes()` — 修正子节点顺序

```c
// drivers/of/fdt.c:251
static void reverse_nodes(struct device_node *parent)
{
    struct device_node *child, *next;

    // 深度优先: 先递归反转子节点的子节点
    child = parent->child;
    while (child) {
        reverse_nodes(child);
        child = child->sibling;
    }

    // 反转当前层的兄弟链表
    // (populate_node 中每次插入到头部, 所以顺序是反的)
    child = parent->child;
    parent->child = NULL;
    while (child) {
        next = child->sibling;
        child->sibling = parent->child;
        parent->child = child;
        child = next;
    }
}
```

---

## 四、Unflatten 后的内存结构

### 4.1 完整视图

```
DTB 中的线性数据:
  /cpus/cpu@0 { compatible = "arm,cortex-a53"; ... };
  /cpus/cpu@1 { compatible = "arm,cortex-a53"; ... };
  /i2c@ff3e0000 { compatible = "rockchip,i2c"; ... };

Unflatten 后:

of_root (/)                struct device_node
  ├── full_name = "/"
  ├── child ───────────────────────────────┐
  │                                         ▼
  │                                  struct device_node "cpus"
  │         ┌─────────────────────── child = ┤
  │         │                                ├── full_name = "/cpus"
  │         ▼                                ├── parent → /
  │   struct device_node "cpu@0"              ├── sibling ──────────────┐
  │    ├── full_name = "/cpus/cpu@0"          │                          ▼
  │    ├── properties → struct property      │                 struct device_node "cpu@1"
  │    │    ├── name = "compatible"           │                   ├── full_name = "/cpus/cpu@1"
  │    │    ├── length = 13                   │                   ├── sibling → ... (more cpus)
  │    │    └── value = "arm,cortex-a53\0"    │                   └── properties
  │    └── (其他属性)                          │
  │                                           │
  │                                           ▼
  │                                  struct device_node "i2c@ff3e0000"
  │                                   ├── full_name = "/i2c@ff3e0000"
  │                                   ├── properties:
  │                                   │    ├── compatible = "rockchip,rk3308-i2c"
  │                                   │    ├── reg = <0xff3e0000 0x1000>
  │                                   │    ├── interrupts = <GIC_SPI 45 ...>
  │                                   │    └── clocks = <&cru CLK_I2C5>, ...
  │                                   └── child → (如果有子节点)
  └── ... (其他顶层节点)
```

### 4.2 `struct device_node` 核心定义

```c
// include/linux/of.h:52
struct device_node {
    const char *name;              // 节点名 (如 "i2c")
    phandle phandle;               // phandle (DTS 引用标识)
    const char *full_name;         // 完整路径 (如 "/soc/i2c@ff3e0000")
    struct fwnode_handle fwnode;   // 通用固件节点接口

    struct property *properties;   // 属性链表
    struct property *deadprops;    // 已删除的属性

    struct device_node *parent;    // 父节点
    struct device_node *child;     // 第一个子节点
    struct device_node *sibling;   // 下一个兄弟节点

    unsigned long _flags;          // 状态标志
    void *data;                    // 驱动自定义数据
};

// 遍历宏 (基于链表):
// for_each_child_of_node(parent, child)
//     → child = parent->child; child; child = child->sibling
// for_each_node_by_name(dn, name)
// for_each_compatible_node(dn, NULL, compatible)
```

### 4.3 `struct property` 定义

```c
struct property {
    char    *name;       // 属性名 (指向 FDT 字符串池或新分配内存)
    int     length;      // 值长度 (bytes)
    void    *value;      // 属性值 (直接指向 DTB 原始数据, 除非被修改)
    struct property *next;  // 下一个属性
};
```

---

## 五、属性访问 API 源码

### 5.1 `of_get_property()` — 获取属性值

```c
// drivers/of/base.c:280
const void *of_get_property(const struct device_node *np,
                            const char *name, int *lenp)
{
    struct property *pp = of_find_property(np, name, lenp);
    return pp ? pp->value : NULL;
}

// 遍历 properties 链表, 匹配 name
// O(n) 时间复杂度, n = 节点属性数
struct property *of_find_property(const struct device_node *np,
                                   const char *name, int *lenp)
{
    struct property *pp;

    // 遍历 properties 链表
    for (pp = np->properties; pp; pp = pp->next) {
        if (strcmp(pp->name, name) == 0) {
            // 找到了!
            if (lenp)
                *lenp = pp->length;
            break;
        }
    }
    return pp;
}
```

### 5.2 `of_property_read_u32()` — 便捷读取

```c
// include/linux/of.h
int of_property_read_u32(const struct device_node *np,
                         const char *propname, u32 *out_value)
{
    const __be32 *val;
    int ret;

    val = of_get_property(np, propname, NULL);
    if (!val)
        return -EINVAL;

    // ★ 大小端转换 (DTB 永远是大端)
    *out_value = be32_to_cpup(val);
    return 0;
}
// 同理: of_property_read_u32_array, of_property_read_string,
//        of_property_count_elems_of_size, ...
```

### 5.3 调用层叠

```c
DTS 源文件: i2c5: i2c@ff3e0000 { reg = <0xff3e0000 0x1000>; }
       ↓ dtc 编译
DTB 二进制: FDT_PROP + "reg" + \xff\x3e\x00\x00\x00\x00\x10\x00
       ↓ unflatten
device_node.property: name="reg", length=8, value=0xff3e0000 (be32)
       ↓ 驱动中
platform_get_resource(pdev, IORESOURCE_MEM, 0)
  → of_address_to_resource(np, 0, &res)
    → of_get_property(np, "reg", NULL)    ← 读二进制值
    → of_translate_address(np, reg)        ← 地址转换
    → res->start = 0xff3e0000
    → res->end   = 0xff3e0fff
```

---

## 六、OF 标志位

```c
// include/linux/of.h
#define OF_DYNAMIC      1  // 动态分配(非来自 DTB)
#define OF_DETACHED     2  // 从主树分离(overlay)
#define OF_POPULATED    3  // 已创建 platform_device
#define OF_POPULATED_BUS 4 // 子节点已 populate
#define OF_OVERLAY      5  // overlay 节点

// 检查:
of_node_check_flag(np, OF_POPULATED)

// 设置/清除:
of_node_set_flag(np, OF_POPULATED)
of_node_clear_flag(np, OF_POPULATED)
```

---

## 七、查看 DTB 的实际方法

```bash
# 1. 从 /sys 导出当前 DTB (需要 root)
cat /sys/firmware/fdt > /tmp/rv1126b.dtb
#    → /sys/firmware/fdt → drivers/of/fdt.c:of_fdt_raw_init()
#       → bin_attribute 直接读 initial_boot_params

# 2. 用 dtc 反编译
dtc -I dtb -O dts /tmp/rv1126b.dts > /tmp/rv1126b.dts

# 3. 查看 device_node 树结构
ls /sys/firmware/devicetree/base/
#    每个目录对应一个 device_node, 属性以文件形式展示
cat /sys/firmware/devicetree/base/model
cat /sys/firmware/devicetree/base/soc/i2c@ff3e0000/compatible

# 4. 查看 DTB 原始二进制
hexdump -C /sys/firmware/fdt | head -10
# 00000000  d0 0d fe ed 00 01 d0 00  ...
```

---

## 八、思考题

1. **两阶段算法**：为什么 unflatten 需要两遍扫描（dry run + real run）？为什么不能一边扫描一边分配？

2. **`reverse_nodes()` 必要性**：`populate_node()` 每次都把新子节点插入到链表头部，导致子节点顺序与 DTS 相反。为什么需要保持 DTS 中的原始顺序？哪些驱动依赖于这个顺序？

3. **`OF_POPULATED` 标志**：`of_platform_device_create_pdata()` 在创建 platform_device 后会设置 `OF_POPULATED`，阻止重复创建。如果 DTS 中有两个驱动匹配同一个节点，会怎样？

4. **性能开销**：`of_get_property()` 通过遍历链表查找属性，时间复杂度 O(n)。如果一个节点有 50 个属性，调用一次 `of_get_property` 最多需要比较 50 次字符串。对于 RT 场景，有什么优化手段？

---

## 相关笔记

- [[bsp-device-model-dtb]] — 阶段二：设备模型 + 设备树
- [[bsp-device-model-platform-bus-deep]] — Platform Bus 源码追溯
- [[bsp-device-model-probe-deep]] — Driver Probe 全路径
- [[MOC-嵌入式Linux]] — 嵌入式 Linux 学习路线 MOC
