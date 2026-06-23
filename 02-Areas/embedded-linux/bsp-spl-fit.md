---
tags:
  - embedded-linux
  - bsp
  - bootloader
  - spl
  - fit-image
  - secure-boot
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

# SPL FIT 镜像解析与验证 — 完整内核追踪

> **前置笔记**：[[bsp-boot-flow]]  — 阶段一主笔记
>
> 本文是 `spl_load_fit_image()` 及其上下游的深度源码级分析，涵盖 FIT 属性解析、SHA256 hash 校验、RSA2048 签名验证、压缩段 digest 二次验证、以及完整的错误传播链路。

---

## 一、调用链路总览

```
board_init_r()                    ← spl.c:530  SPL 主入口
  └─ boot_from_devices()          ← spl.c:453  遍历启动设备
       └─ spl_load_image()        ← 调用对应设备的 loader
            └─ spl_load_simple_fit()    ← spl_fit.c:941  ★入口
                 ├─ 1. 读取 FIT blob (FDT_MAGIC 校验)
                 ├─ 2. fit_config_verify()  配置层签名
                 ├─ 3. 加载 standalone 固件
                 ├─ 4. 加载 firmware (atf-1)
                 │    └─ spl_load_fit_image() ←★核心函数
                 ├─ 5. 追加 FDT
                 └─ 6. 遍历 loadables (atf-2/3/4, uboot, optee)
                      └─ spl_load_fit_image() ←★每段均调用
```

---

## 二、核心函数 `spl_load_fit_image()` 逐行解析

> 源码位置：`u-boot/common/spl/spl_fit.c:251-409`

### 输入参数

| 参数 | 含义 | 示例 (atf-1) |
|------|------|-------------|
| `info` | eMMC 读取接口 (含 read() 函数指针) | `mmc_bread` |
| `sector` | FIT 镜像起始扇区号 | `0x4000` |
| `fit` | 已加载到内存的完整 FDT/FIT blob | `void *` |
| `base_offset` | 外部数据区的起始偏移 (FDT 结构体之后) | 页对齐偏移 |
| `node` | 当前段在 FIT FDT 中的节点偏移量 | `/images/atf-1` |
| `image_info` | 出参：加载后的段信息 (load_addr, size, entry) | 调用者提供指针 |

### 处理流程 (按代码执行顺序)

#### Step 1: 读取压缩类型 (L268-280)

```c
fit_image_get_comp(fit, node, &image_comp);
// → fdt_getprop(fit, node, "compression", &len)
```

各段对应的值：

| 段名 | ITS 中 compression 值 | image_comp 常量 |
|------|----------------------|----------------|
| atf-1 | `"lzma"` | `IH_COMP_LZMA` |
| atf-2 | 无此属性 | `IH_COMP_NONE` |
| atf-3 | 无此属性 | `IH_COMP_NONE` |
| atf-4 | 无此属性 | `IH_COMP_NONE` |
| uboot | `"lzma"` | `IH_COMP_LZMA` |

> **关键理解**：只有 DDR 中的段 (atf-1, uboot) 才压缩。SRAM 段 (atf-2/3/4) 不压缩，因为 SRAM 空间紧俏，无法容纳解压缓冲。

#### Step 2: 读取加载地址 (L282-283)

```c
fit_image_get_load(fit, node, &load_addr);
// → fdt_getprop(fit, node, "load", &len) → fdt32_to_cpu()
```

| 段 | load_addr | 目标内存类型 |
|----|-----------|------------|
| atf-1 | `0x40000000` | DDR (DRAM 起始) |
| atf-2 | `0x3ffbb000` | System SRAM (0x3ffb0000~0x3ffc0000, 64KB) |
| atf-3 | `0x3ff1e000` | PMU SRAM (0x3ff1e000 区域) |
| atf-4 | `0x3ffbd000` | System SRAM (数据区) |
| uboot | `0x40200000` | DDR |
| fdt | `0x41000000` | DDR (DTB 区域) |

#### Step 3: 计算解压缓冲区 (L285-301)

```c
if (image_comp != IH_COMP_NONE) {
    // 尝试读取 ITS 中的 comp-addr 属性 (极少使用)
    if (fit_image_get_comp_addr(fit, node, &comp_addr)) {
        // 属性不存在，自己计算:
        // comp_addr = load_addr + 2MB
        if (load_addr + 2*FIT_MAX_SPL_IMAGE_SZ <= gd->ram_top)
            comp_addr = load_addr + FIT_MAX_SPL_IMAGE_SZ;
        else
            comp_addr = load_addr - FIT_MAX_SPL_IMAGE_SZ;  // 小内存设备
    }
} else {
    comp_addr = load_addr;  // 无压缩 → 加载地址即目标地址
}
```

设计意图图：

```
atf-1 (LZMA 压缩):
  comp_addr = 0x40000000 + 2MB = 0x40200000  ← 压缩数据暂存
  load_addr = 0x40000000                      ← 解压后的目标

atf-2 (无压缩):
  comp_addr = load_addr = 0x3ffbb000 ← 直接加载到 SRAM
```

#### Step 4: 定位 FIT 中的数据 (L310-350)

```c
// 先查 data-position (外部数据在 FIT 文件中的绝对偏移)
if (!fit_image_get_data_position(fit, node, &offset)) {
    external_data = true;
}
// 再查 data-offset (相对 FIT 数据区的偏移)
else if (!fit_image_get_data_offset(fit, node, &offset)) {
    offset += base_offset;
    external_data = true;
}

if (external_data) {
    fit_image_get_data_size(fit, node, &len);
    load_ptr = (comp_addr + align_len) & ~align_len;  // DMA 对齐
    info->read(info, sector + offset, nr_sectors, load_ptr);
    src = (void *)load_ptr + overhead;
} else {
    // 内嵌数据模式 (数据在 FDT 结构体内)
    fit_image_get_data(fit, node, &data, &length);
    src = (void *)data;
}
```

FIT 文件物理布局：

```
┌─────────────────┐  offset 0
│ FIT FDT 结构体    │  (/images/atf-1 → data-position=0x0000A000)
│   /images/       │  (/images/atf-2 → data-position=0x0001A000)
│   /configurations│
├─────────────────┤  ← base_offset (FDT 结束 + 页对齐)
│ 填充/空          │
├─────────────────┤  ← data-position[atf-1]
│ bl31_0x40000000 │  ← LZMA 压缩的 86KB
│ .bin.lzma       │
├─────────────────┤  ← data-position[atf-2]
│ bl31_0x3ffbb000 │  ← 未压缩的 8KB
│ .bin            │
├─────────────────┤
│ ...              │
└─────────────────┘
```

#### Step 5: 核心验证 — hash + 签名 (L352-368)

```c
printf("## Checking %s 0x%08lx (%s @0x%08lx) ... ",
       name, load_addr, compression, (long)src);
// 输出: "## Checking atf-1 0x40000000 (lzma @0x40200000) ... "

if (!fit_image_verify_with_data(fit, node, src, length))
    return -EPERM;  // ← 验证失败，直接返回
```

`fit_image_verify_with_data` 内部 (image-fit.c:1408-1468)：

```
┌─────────────────────────────────────────────────────────────┐
│  fit_image_verify_with_data(fit, node, data, size)          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  第一层: required 签名验证 (强制通过，失败则终止)               │
│  ┌─────────────────────────────────────────────┐            │
│  │ fit_image_verify_required_sigs()             │            │
│  │   → 读取 U-Boot control FDT 中 /signature   │            │
│  │   → 遍历所有 required="image" 的 RSA 公钥   │            │
│  │   → 逐个验证签名 (RSA 解密 → 比对 hash)      │            │
│  │   → 任一失败 → return 0 (ERROR)             │            │
│  └─────────────────────────────────────────────┘            │
│                            ↓                                 │
│  第二层: 遍历所有子节点                                       │
│  ┌─────────────────────────────────────────────┐            │
│  │ fdt_for_each_subnode(noffset, fit, node) {   │            │
│  │                                              │            │
│  │   if 子节点名 == "hash" or "hash@N":         │            │
│  │     fit_image_check_hash()                   │            │
│  │       → 读取 algo="sha256"                   │            │
│  │       → 读取 value=<32字节 hash>             │            │
│  │       → calculate_hash(data, size, sha256)   │            │
│  │       → memcmp(computed, expected, 32)       │            │
│  │       → 成功: puts("+ ")                     │            │
│  │       → 失败: goto error (强制终止)          │            │
│  │                                              │            │
│  │   if 子节点名 == "signature" or "signature@":│            │
│  │     fit_image_check_sig()                    │            │
│  │       → RSA 公钥解密签名                      │            │
│  │       → 比对 hash                             │            │
│  │       → 成功: puts("+ ")                     │            │
│  │       → 失败: puts("- ") (不阻断!)           │            │
│  │   }                                          │            │
│  └─────────────────────────────────────────────┘            │
│                                                              │
│  返回 1 (成功) 或 0 (失败)                                   │
└─────────────────────────────────────────────────────────────┘
```

**两层验证的区别**：

| 校验类型 | 节点名称 | 作用 | 失败后果 |
|---------|---------|------|---------|
| **hash** | `hash` / `hash@N` | SHA256 数据完整性校验 | **强制终止** |
| **signature (required)** | `signature@` + `required="image"` | RSA 公钥签名，防篡改 | **强制终止** |
| **signature (non-required)** | `signature@` | 额外的可选签名 | 仅 `-` 警告 |

> **关键设计**：hash 始终强制验证，签名可分级。这是"完整性优先、信任链追加"的安全哲学。

#### Step 6: 后处理 — 解压 + digest 二次验证 (L380-384)

```c
board_fit_image_post_process(fit, node, &load_addr, &src, &length, info);
// 进入 fit_misc.c:155 的 Rockchip 定制实现
```

`board_fit_image_post_process` 内部：

```c
int board_fit_image_post_process(fit, node, load_addr, src_addr, src_len, spec) {
    // 1. LZMA 解压
    fit_decomp_image(fit, node, load_addr, src_addr, src_len, spec);
    //   src_addr = 压缩数据 → load_addr = 解压目标
    //   调用 lzmaBuffToBuffDecompress()

    // 2. 校验解压后的 hash (digest 节点)
    fit_image_check_uncomp_hash(fit, node, after_decomp_data, size);
    //   读取 digest 子节点:
    //     digest { value = SHA256(原始.bin); algo = "sha256"; }
    //   计算解压后数据的 SHA256 ← → 比对
    //   → 防篡改：即使压缩数据通过 hash 校验，篡改者也可能
    //     伪造压缩数据，使其解压出恶意代码
}
```

**三次验证的架构图**：

```
                    ┌──────────────────┐
                    │  bl31.bin 原始文件 │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │  LZMA 压缩        │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
     ┌────────────┐  ┌────────────┐  ┌────────────┐
     │SHA256(原始) │  │SHA256(压缩)│  │ RSA2048签名│
     │→ digest节点│  │→ hash节点  │  │→ signature │
     └────────────┘  └────────────┘  └────────────┘
           │               │               │
           │               │               │
    ═══════╪═══════════════╪═══════════════╪══════════
    验证顺序 │               │               │
           │               │               │
    ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
    │③ digest校验 │←│① hash校验   │ │② RSA验签   │
    │(解压后校验)  │ │(压缩数据校验)│ │(公钥解密)   │
    └─────────────┘ └─────────────┘ └─────────────┘
```

#### Step 7: 数据搬运到目标地址 (L387-400)

```c
// 对于 atf-1 (LZMA 压缩):
//   解压已在 Step 6 完成，src 指向解压后数据
memcpy((void *)load_addr, src, length);
// atf-1: memcpy(0x40000000, decompressed, 139KB)
// atf-2: memcpy(0x3ffbb000, raw_data, 8KB)
```

#### Step 8: 记录加载信息 (L402-406)

```c
image_info->load_addr = load_addr;
image_info->size = length;
image_info->entry_point = fdt_getprop_u32(fit, node, "entry");
```

---

## 三、验证失败 — 完整错误传播层级 (L0-L6)

### 错误传播树

```
L6 fit_image_check_hash()       → err_msg="Bad hash value" → return -1
  │
L5 fit_image_verify_with_data() → printf("error!\n...")    → return 0
  │
L4 spl_load_fit_image()         → return -EPERM
  │
L3 spl_internal_load_simple_fit → return ret (终止遍历)
  │
L2 spl_load_simple_fit()        → 尝试多份 FIT / A/B 降级
  │
L1 spl_ab_decrease_reset()      → tries_remaining-- → do_reset()
  │
L0 hang()                       → 所有设备失败 → 死循环
```

### 各层错误码

| 层级 | 函数 | 失败返回 | 成功返回 |
|------|------|---------|---------|
| L6 | `fit_image_check_hash` | `-1` | `0` |
| L5 | `fit_image_verify_with_data` | `0` (FALSE) | `1` (TRUE) |
| L4 | `spl_load_fit_image` | `-EPERM` | `0` |
| L3 | `spl_internal_load_simple_fit` | `-1` | `0` |
| L2 | `spl_load_simple_fit` | 传递下层错误 / A/B 处理 | `0` |
| L1 | `spl_ab_decrease_reset` | `-ENODEV` (无可用slot) | `do_reset()` (不复返) |
| L0 | `hang()` | 不复返 | — |

### 5 种 hash 校验失败原因

| err_msg | 触发条件 |
|---------|---------|
| `Can't get hash algo property` | ITS 中 hash 节点缺少 `algo` 属性 |
| `Can't get hash value property` | ITS 中 hash 节点缺少 `value` 属性 |
| `Unsupported hash algorithm` | algo 不是 sha256/sha1/md5 |
| `Bad hash value len` | 计算结果长度 ≠ 期望长度 |
| `Bad hash value` | **hash 不匹配** — 数据被篡改/损坏 |

### 3 层容错机制

```
层1: CONFIG_SPL_FIT_IMAGE_MULTIPLE  — 一份 eMMC 上的多份 FIT 镜像
层2: SPL_AB (AvbABData)              — misc 分区 A/B slot 状态机
层3: boot_from_devices()             — 遍历所有启动设备 (eMMC → USB → SPI → 网络)
```

### 典型串口日志 (hash 失败 + A/B 降级)

```
Trying fit image at 0x4000 sector

## Checking atf-1 0x40000000 (lzma @0x40200000) ... sha256 error!
Bad hash value for 'hash' hash node in 'atf-1' image node

A/B: slot boot fail, do reset

[系统复位...]

Trying fit image at 0x8000 sector   ← slot B 的 FIT 镜像
## Checking atf-1 0x40000000 ... sha256+ OK
## Checking uboot 0x40200000 ... sha256+ OK
## Checking atf-2 0x3ffbb000 ... sha256+ OK
## Checking atf-3 0x3ff1e000 ... sha256+ OK
## Checking atf-4 0x3ffbd000 ... sha256+ OK
## Checking fdt 0x41000000 ... sha256+ OK
```

### 所有路径耗尽时的日志

```
Trying fit image at 0x4000 sector
Trying to boot from MMC1
## Checking atf-1 ... error!
Trying to boot from MMC2
## Checking atf-1 ... error!
Trying to boot from USB
...
SPL: failed to boot from all boot devices
[hang — 系统死锁，需硬件复位]
```

---

## 四、`spl_internal_load_simple_fit()` 加载顺序

> 源码：`spl_fit.c:714-939`

### [阶段 A] standalone 固件 (其他核心)

```c
for (;; index++) {
    node = spl_fit_get_image_node(fit, images, FIT_STANDALONE_PROP, index);
    if (node < 0) break;
    
    spl_load_fit_image(...);           // 加载 RISC-V MCU 固件
    spl_fit_standalone_release(...);   // 释放其他核心 (从 reset 唤醒)
}
```

> RV1126B AMP 架构中，RISC-V MCU 可能在此阶段被释放。当前 sportcam 配置中此属性通常为空。

### [阶段 B] firmware (ATF BL31 entry point)

```c
node = spl_fit_get_image_node(fit, images, FIT_FIRMWARE_PROP, 0);
// → 查找 /configurations/conf 中的 "firmware" 属性
// → 返回 "/images/atf-1"

spl_load_fit_image(info, sector, fit, base_offset, node, spl_image);
// → 加载 atf-1 到 spl_image (主镜像结构)
// → spl_image->os = "arm-trusted-firmware" (IH_OS_ARM_TRUSTED_FIRMWARE)
```

### [阶段 C] 追加 FDT

```c
if (spl_image->os == IH_OS_U_BOOT) {
    spl_fit_append_fdt(spl_image, info, sector, fit, images, base_offset);
    // → 加载 /images/fdt (U-Boot DTB)
    // → 加载 /images/kern-fdt (Kernel DTB, 可选)
}
```

> **注意**：当前 sportcam 配置中 firmware 是 ATF 而非 U-Boot，此条件不成立。FDT 加载发生在 loadables 遍历第 1 项 (uboot) 中。

### [阶段 D] 遍历 loadables

```c
for (; ; index++) {
    node = spl_fit_get_image_node(fit, images, "loadables", index);
    if (node < 0) break;

    spl_load_fit_image(...);   // 每段独立验证+加载

    // 特殊处理 os 类型
    if (os_type == IH_OS_U_BOOT) {
        // uboot 是 loadables 中的特殊一员
        spl_image->entry_point_bl33 = load_addr;
        // ATF 完成后将跳转到此地址启动 U-Boot
        spl_fit_append_fdt(...);  // 追加 U-Boot DTB
    }
    else if (os_type == IH_OS_OP_TEE) {
        spl_image->entry_point_bl32 = load_addr;
    }

    // 记录到 FDT (供 U-Boot proper 使用)
    spl_fit_record_loadable(fit, images, index, spl_image->fdt_addr, &image_info);
}
```

加载顺序对照 ITS：

```
configurations/conf {
    firmware  = "atf-1";           ← 阶段 B: 唯一的 entry point
    loadables = "uboot",           ← 阶段 D: 第1项 → U-Boot proper
                "atf-2",           ←        第2项 → SRAM 代码
                "atf-3",           ←        第3项 → PMU SRAM
                "atf-4";           ←        第4项 → SRAM 数据
    fdt       = "fdt";            ← 阶段 C (在 uboot 加载中追加)
};
```

### 完整的运行时加载时序

```
    SPL 串口输出                              内部操作
────────────────────────────────────────────────────────────────
Trying fit image at 0x4000 sector    ← SPL 读取 FIT Header
                                      ← 配置层 RSA 验签
                                      
## Checking atf-1 0x40000000 ... OK  ← firmware 加载
  sha256+                            ← hash 通过
  +                                  ← 签名通过
  OK                                 ← LZMA 解压完成
                                      
## Checking uboot 0x40200000 ... OK  ← loadable #1
  sha256+                            ← hash 通过  
  OK                                 ← LZMA 解压完成
  (同时追加 FDT 到 DTB 区域)
                                      
## Checking atf-2 0x3ffbb000 ... OK  ← loadable #2 (SRAM)
  sha256+                            ← 无压缩，直接 hash
                                      
## Checking atf-3 0x3ff1e000 ... OK  ← loadable #3 (PMU SRAM)
  sha256+
                                      
## Checking atf-4 0x3ffbd000 ... OK  ← loadable #4 (SRAM data)
  sha256+
                                      
                                      ← 跳转到 atf-1 entry (0x40000000)
[ATF BL31 启动]                       ← EL3 初始化
[ATF → U-Boot]                       ← EL3 返回 EL2
[U-Boot proper 启动]                  ← 继续第二阶段启动
```

---

## 五、相关笔记

- [[bsp-boot-flow]] — 阶段一主笔记 (Boot Chain, 分区表, FIT 镜像, AMP)
- [[bsp-device-model-dtb]] — 阶段二：设备模型 + 设备树
- [[MOC-嵌入式Linux]] — 嵌入式 Linux 学习路线 MOC