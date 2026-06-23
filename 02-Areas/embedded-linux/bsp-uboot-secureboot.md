---
tags:
  - embedded-linux
  - bsp
  - bootloader
  - u-boot
  - secure-boot
  - verified-boot
  - fit-image
  - rsa
  - avb
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

# U-Boot 安全启动 & FIT 签名 — 深度解析

> **前置笔记**：[[bsp-boot-flow]] — Boot Chain 全景 & FIT 基础
>
> **前置笔记**：[[bsp-spl-fit]] — SPL FIT 解析源码追踪 (hash/RSA 验签)
>
> **前置笔记**：[[bsp-uboot-adaptation]] — U-Boot 板级适配 & Defconfig 配置
>
> 本文聚焦安全启动：从 FIT RSA 签名原理到 Rockchip 完整安全启动链，涵盖 SPL 验签、AVB、防回滚、OTP 密钥管理。

---

## 一、安全启动链全景

### 1.1 信任链模型

安全启动的核心是 **Chain of Trust**：每个阶段验证下一阶段的完整性和真实性，根信任锚 (Root of Trust) 位于芯片硬件。

```
BootROM (RoTHash)
  │  SPL 签名验证 ← OTP 中存储的 SPL 公钥 hash
  ▼
SPL / ATF BL31
  │  U-Boot FIT 签名验证 ← SPL 中嵌入的公钥
  ▼
U-Boot proper
  │  Kernel FIT 签名验证 ← U-Boot 中嵌入的公钥
  ▼
Kernel
  │  rootfs dm-verity / fs-verity ← 内核建立完整性
  ▼
Userspace
```

### 1.2 RV1126B 安全启动能力

| 安全特性 | SDK 支持 | sportcam 当前状态 |
|---------|---------|----------------|
| SPL 签名验证 | ✅ `CONFIG_SPL_FIT_SIGNATURE` | ❌ 未启用 |
| U-Boot FIT 签名 | ✅ `CONFIG_FIT_SIGNATURE` | ❌ 未启用 (ITS 已预设) |
| AVB 2.0 | ✅ `CONFIG_AVB_LIBAVB=y` | ✅ 已启用 (library) |
| OTP 密钥烧录 | ✅ `RK_SECURITY_BURN_KEY` | ❌ 未启用 |
| 防回滚 | ✅ rollback-index (ITS) | ❌ ITS 中 `rollback-index=<0x00>` 为默认 |
| dm-verity | ✅ `RK_SECURITY_CHECK_SYSTEM_VERITY` | ❌ 未启用 |
| 系统加密 | ✅ `RK_SECURITY_CHECK_SYSTEM_ENCRYPTION` | ❌ 未启用 |

> **核心矛盾**：`boot.its` 已预置 `signature { algo="sha256,rsa2048"; }` 配置层签名节点，但 defconfig 未启用 `CONFIG_FIT_SIGNATURE`。这意味着签名节点目前是**静态度量** — 验签代码不会执行，签名节点的存在仅用于兼容性。

---

## 二、FIT 签名机制深度拆解

### 2.1 基础概念：RSA + SHA256

FIT 签名使用 **RSA 非对称加密**：

| 角色 | 密钥 | 存放位置 | 作用 |
|------|------|---------|------|
| 开发者/厂商 | **私钥** (dev.key) | 开发机 (机密) | 对 FIT 镜像签名 |
| 设备 | **公钥** (dev.pub) | U-Boot/SPL 二进制中 | 验证 FIT 镜像签名 |
| OTP/eFuse | **公钥 hash** | 芯片 OTP (一次性写入) | 验证公钥本身的完整性 |

**签名过程**：

```
原始数据 (kernel Image + DTB + resource)
  │  SHA256 hash
  ▼
hash (32 bytes)
  │  RSA 2048 私钥加密 (sign)
  ▼
signature (256 bytes)
  │  嵌入到 FIT 的 signature 节点
  ▼
FIT 镜像 (boot.img)
```

**验证过程**：

```
FIT 镜像 (boot.img)
  │  提取 signature 节点
  │  ├─ 签名值 (256 bytes)
  │  └─ 签名算法描述 ("sha256,rsa2048")
  │
  ├─ 步骤 1: 对 kernel + DTB 算 SHA256 hash
  │
  ├─ 步骤 2: RSA公钥解密 signature → 得到 hash_decrypted
  │
  └─ 步骤 3: 对比 hash_calculated == hash_decrypted ?
              ├─ 相等 → ✅ 镜像完整且来自可信源
              └─ 不等 → ❌ 镜像被篡改，拒绝启动
```

### 2.2 `boot.its` 签名节点逐字段解析

```dts
// device/rockchip/.chips/rv1126b/boot.its:54-65
configurations {
    default = "conf";                           // 默认配置

    conf {
        rollback-index = <0x00>;                // ★ 防回滚版本号
        fdt = "fdt";                            // 引用 images/fdt 节点
        kernel = "kernel";                      // 引用 images/kernel 节点
        multi = "resource";                     // 引用 images/resource 节点

        signature {                             // ★ 签名节点
            algo = "sha256,rsa2048";            //   算法: hash,签名算法
            padding = "pss";                    //   填充: RSA-PSS (更安全)
            key-name-hint = "dev";              //   密钥名提示 (用于查找公钥)
            sign-images = "fdt", "kernel", "multi";  // 哪些 image 被签名
        };
    };
};
```

| 字段 | 值 | 含义 |
|------|-----|------|
| `algo` | `sha256,rsa2048` | 先 SHA256 哈希，再用 RSA 2048 签名 |
| `padding` | `pss` | **RSA-PSS** (Probabilistic Signature Scheme)，比 PKCS#1 v1.5 更安全 |
| `key-name-hint` | `dev` | U-Boot 在 `CONFIG_FIT_SIGNATURE_KEYDIR` 目录中查找 `dev.key`/`dev.pub` |
| `sign-images` | fdt, kernel, multi | 仅签名列出的 image 节点，未列出的不受保护 |
| `rollback-index` | `0x00` | 防降级版本号，递增序列 |

> **关键理解**：U-Boot 的 RSA-PSS 实现和 Linux 内核的 `fs_verity` 共用同一套底层数学库 (`lib/rsa/`)，基于 `libtomcrypt`。

### 2.3 signature 值在 FIT 二进制中的位置

FIT 是 FDT (Flattened Device Tree) 格式，签名数据作为属性存放在 `signature` 节点中：

```
FIT 二进制 (boot.img) 内存布局:

┌──────────────────────────────┐
│  FDT Header                  │  ← 魔数 0xD00DFEED
├──────────────────────────────┤
│  FDT 结构体 (flattened tree) │
│   ├─ /images/fdt             │
│   │   ├─ data = ...          │  ← DTB 数据 (引用外部)
│   │   ├─ hash = sha256(...)  │  ← image 自身的 hash
│   │   └─ ...                 │
│   ├─ /images/kernel          │
│   │   ├─ data = ...          │  ← Image 数据 (引用外部)
│   │   ├─ hash = sha256(...)  │
│   │   └─ ...                 │
│   ├─ /configurations/conf    │
│   │   └─ signature {         │
│   │       algo = "sha256,rsa2048"
│   │       rsa-padding = "pss"
│   │       value = <256 字节> │  ← ★ 签名的二进制数据
│   │       ...                │
│   │   }                      │
│   └─ ...                     │
├──────────────────────────────┤
│  外部数据 (external data)    │  ← 对齐到页边界
│  ┌────────────────────────┐  │
│  │ kernel Image           │  │  ← data 属性指向的偏移
│  ├────────────────────────┤  │
│  │ fdt blob               │  │
│  ├────────────────────────┤  │
│  │ resource (logo etc.)   │  │
│  └────────────────────────┘  │
└──────────────────────────────┘
```

> **注意**：`external data` 模式 (FIT with external data) 下，kernel 和 DTB 数据存储在 FDT 结构体之后的外部区域，`data` 属性通过偏移和长度引用。签名时这些外部数据一起被哈希，因此**外部数据**也在保护范围内。

---

## 三、公钥嵌入流程

公钥必须嵌入到 U-Boot/SPL 二进制中，才能在验签时使用。

### 3.1 密钥对生成

```bash
# 生成 RSA 2048 私钥
openssl genrsa -out dev.key 2048

# 提取公钥
openssl rsa -in dev.key -pubout -out dev.pub

# 查看公钥详情
openssl rsa -pubin -in dev.pub -text -noout
# ┌─────────────────────────────────────────────┐
# │ RSA Public-Key: (2048 bit)                  │
# │ Modulus (2048 bit):                          │
# │     00:e4:d4:... (256 bytes = RSA 模数 N)   │
# │ Exponent: 65537 (0x10001)                    │
# └─────────────────────────────────────────────┘
```

### 3.2 公钥嵌入到 U-Boot DTB

U-Boot 编译时，通过 `tools/keygen` 或编译脚本将公钥嵌入到 U-Boot 的 DTB 中：

```bash
# 方法 1: 编译时 mkimage 嵌入
mkimage -f boot.its -k keys/ -r boot.img
# -k keys/: 指定密钥目录，包含 dev.key 和 dev.pub
# -r: 将公钥嵌入到生成的 FIT 中 (required 模式)

# 方法 2: 编译后手动添加 key 节点到 U-Boot DTB
fdtput -t s u-boot.dtb /signature/key-dev \
    algo sha256,rsa2048 \
    required-conf conf \
    key-name-hint dev
```

公钥嵌入到 U-Boot DTB 后的结构：

```dts
// U-Boot 的 DTB 中 (编译后自动生成)
/signature {
    key-dev {
        algo = "sha256,rsa2048";
        required-conf = <0x...>;       // 指向 configuration 节点
        key-name-hint = "dev";
        rsa,r-squared = <0x...>;       // 蒙哥马利域 (加速验算)
        rsa,n-bits = <0x800>;          // 2048 = 0x800 bits
        rsa,modulus = [e4 d4 ...];     // 模数 N (256 bytes)
        rsa,exponent = <0x10001>;      // 公钥指数
        rsa,n0-inv = <0x...>;          // 加速参数
    };
};
```

> **U-Boot 为什么在 DTB 中存额外参数 (r-squared, n0-inv)？**  
> RSA 2048 验签是计算密集型操作，每次做 2048 位模幂运算非常慢。这些参数是 **蒙哥马利约简**所需的预计算值，可将验签速度提升约 **10 倍**，适合启动场景的时间约束。

### 3.3 SPL / U-Boot 的密钥分离

```
┌────────────────────────────────────────┐
│ SPL 二进制 (u-boot-spl.bin)           │
│  嵌入密钥: SPL-boot-key.pub           │
│  验证: uboot.img (U-Boot proper FIT)  │
└────────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────┐
│ U-Boot 二进制 (u-boot.bin)            │
│  嵌入密钥: boot-key.pub               │
│  验证: boot.img (Kernel FIT)          │
└────────────────────────────────────────┘
```

**注意**：生产环境中 SPL 和 U-Boot 应使用**不同的密钥对**，以降低单密钥泄露的风险。SPL 的公钥 hash 存储在 OTP 中，由 BootROM 验证。

---

## 四、SPL 验签流程

> **参见源码追踪**：[[bsp-spl-fit]] — `spl_load_simple_fit()` 与 `fit_config_verify()` 完整解析

### 4.1 SPL 验签开关

```kconfig
# 启用 SPL FIT 签名验证
CONFIG_SPL_FIT_SIGNATURE=y
CONFIG_SPL_LOAD_FIT=y
CONFIG_SPL_ATF=y

# SPL 密钥目录
CONFIG_SPL_FIT_SIGNATURE_KEYDIR="keys/spl"
```

### 4.2 验签流程

```
spl_load_simple_fit()             ← spl_fit.c:941
  │
  ├─ 1. 读取 FIT header (FDT_MAGIC 校验)
  │
  ├─ 2. fit_config_verify(fit, conf_node)  ← ★ 配置层签名验证
  │      │  common/image-fit.c:
  │      │
  │      ├─ 查找 /configurations/conf/signature 节点
  │      │
  │      ├─ 读取 algo = "sha256,rsa2048"
  │      │
  │      ├─ 从 U-Boot DTB 中查找 /signature/key-dev → 取出公钥
  │      │
  │      ├─ rsa_verify_hash()           ← lib/rsa/rsa-verify.c
  │      │    ├─ SHA256(images_data)    ← 对 sign-images 指定的内容算 hash
  │      │    ├─ RSA 公钥解密 signature → hash_decrypted (蒙哥马利模幂)
  │      │    └─ memcmp(hash_calculated, hash_decrypted)
  │      │
  │      └─ 结果:
  │            ├─ ✅ → 继续加载
  │            └─ ❌ → fit_verify_failed() → 根据 required 模式处理
  │
  ├─ 3. 加载 firmware (atf-1)
  ├─ 4. 追加 FDT
  └─ 5. 遍历 loadables
       └─ spl_load_fit_image()     ← ★ 每个段额外 hash 校验
```

### 4.3 SPL 签名 vs SPL hash 双校验

```
配置层签名                   段级 hash
(fit_config_verify)         (spl_load_fit_image 中的 fit_image_check_hash)
─────────────────           ─────────────────────
验证完整性 + 来源           仅验证完整性
RSA 非对称                   SHA256 对称
CPU 负载高 (~20ms)           CPU 负载低 (~0.1ms)
一次 (configuration)        每段一次 (atf-1~4, uboot, fdt)
```

> **为什么需要双重校验？** 配置层签名保来源可信，段级 hash 保数据完整性（如 LZMA 解压后）。在 `spl_load_fit_image()` 中，压缩段解压后还要再做一次 digest 验证。

---

## 五、U-Boot 验签流程

### 5.1 U-Boot 验签开关

```kconfig
CONFIG_FIT_SIGNATURE=y           # U-Boot proper FIT 签名
CONFIG_FIT_SIGNATURE_KEYDIR="keys/boot"
# 注意: CONFIG_FIT_SIGNATURE 不是 CONFIG_SPL_FIT_SIGNATURE
```

### 5.2 bootm 命令中的验签

```
U-Boot proper 启动 kernel:
  │
  bootm <kernel_addr> - <fdt_addr>
  │
  ├─ bootm_find_os()           ← 读取 FIT 中的 kernel 节点
  │
  ├─ fit_image_load()          ← 加载 FIT image
  │    └─ fit_image_check_sig()  ← ★ 检查签名
  │         │
  │         ├─ fit_config_verify()  ← 配置层签名验证
  │         │
  │         └─ 失败时:
  │             ├─ required 模式 → hang() / panic()
  │             └─ optional 模式 → 打印警告，继续启动
  │
  ├─ bootm_load_os()           ← 加载到内存
  └─ boot_jump_linux()         ← 跳转内核
```

### 5.3 required vs optional 模式

```dts
// FIT 中的 signature 节点
signature {
    algo = "sha256,rsa2048";
    key-name-hint = "dev";
    sign-images = "fdt", "kernel", "multi";
    required = <0x01>;          // ★ 关键字段
};
```

| required 值 | 模式 | 验签失败行为 |
|-------------|------|-------------|
| 未设置 | optional | 打印 warning，**继续启动** |
| `<0x01>` | required | **停止启动** (for SPL: hang(); for U-Boot: panic()) |
| `<0x02>` | required-sig | 仅签名失败阻止，hash 匹配但无签名仍可通过 |

对于 DJI 等安全敏感产品，所有阶段都应配置为 **required** 模式。

---

## 六、AVB (Android Verified Boot)

### 6.1 AVB vs FIT 签名对比

Rockchip SDK 同时支持两种安全启动方式：

| 特性 | FIT Signature | AVB 2.0 |
|------|-------------|---------|
| 标准 | U-Boot 原生 | Google Android |
| 适用范围 | Bootloader, Kernel | Android 系统 + VBMeta |
| 根信任 | OTP 公钥 hash | OTP 公钥 hash (avbkey) |
| 防回滚 | `rollback-index` | `rollback_index` (分区级) |
| dm-verity | 无 | 原生支持 |
| 多分区 | 手动配置 `sign-images` | VBMeta 自动管理 |
| Rockchip 芯片 | RK3568/RK3588/RV1126B | RK3308/RK3399/RK3326/PX30 |

当前 sportcam 配置：

```kconfig
# u-boot/configs/rv1126b_sportcam_defconfig:201-204
CONFIG_AVB_LIBAVB=y               # AVB 库
CONFIG_AVB_LIBAVB_AB=y            # A/B 分区支持
CONFIG_AVB_LIBAVB_ATX=y           # Android Things 扩展
CONFIG_AVB_LIBAVB_USER=y          # 用户空间工具
```

> **注意**：`CONFIG_AVB_LIBAVB=y` 表示编译了 AVB 库，但未启用 `CONFIG_AVB_BOOT` 或设置 VBMeta 分区，因此不执行 AVB 验证。该配置主要是为了兼容 fastboot 烧录协议。

### 6.2 AVB 信任链

```
BootROM
  → 验证 SPL 签名 (OTP 中公钥 hash)
    → SPL 验证 U-Boot signature
      → U-Boot 验证 boot.img (VBMeta)
        → VBMeta 中的 dm-verity hash tree 验证 system 分区
          → Kernel 用 dm-verity 校验每次块读取
```

### 6.3 启用 AVB 需要的配置

```kconfig
# 在 defconfig 中启用完整 AVB
CONFIG_AVB_BOOT=y                  # AVB 启动验证
CONFIG_AVB_VBMETA_PUBLIC_KEY_VALIDATE=y  # 验证 VBMeta 公钥

# 分区要求
# system 分区: squashfs + dm-verity hash tree
# vbmeta 分区: VBMeta 描述符 (分区 hash + rollback index)
# boot 分区: FIT 或 Android bootimage
```

---

## 七、防回滚机制

### 7.1 防回滚原理

防回滚 (Rollback Protection) 防止攻击者将固件降级到有已知漏洞的旧版本。

```
版本: v1.0 → v1.1 → v1.2 → v1.3 (当前)
                        ↑
                    攻击者想降级到 v1.2 (有漏洞)
                       → ❌ 防回滚阻止
```

### 7.2 FIT rollback-index

```dts
// boot.its -> configurations -> conf
conf {
    rollback-index = <0x03>;    // 版本号 (每次发布递增)
    signature {
        ...
    };
};
```

SPL 验签时检查 `rollback-index`：

```c
// spl_fit.c:770-785
static int spl_fit_rollback_index(struct spl_fit_info *ctx, int node)
{
    int ret;
    u32 index;
    
    ret = fdtdec_get_int(ctx->fit, node, "rollback-index", -1);
    if (ret < 0)
        return 0;  // 无 rollback-index 配置，跳过
    
    index = read_rollback_index_from_storage();  // 从 RPMB/OTP 读取

    if ((u32)ret < index) {
        printf("fit reject rollback: %d < %d(min)\n", ret, index);
        return -EPERM;  // ❌ 固件过旧
    }
    
    // 更新存储中的版本号
    write_rollback_index_to_storage(ret);
    printf("rollback index: %d >= %d(min), OK\n", ret, index);
    return 0;
}
```

### 7.3 存储 rollback-index 的安全性

| 存储方式 | 防篡改能力 | 说明 |
|---------|-----------|------|
| RPMB (eMMC) | ✅ 硬件保护 | eMMC 内部 Replay Protected Memory Block |
| OTP (eFuse) | ✅ 一次性写 | 写入不能修改，适用于关键版本 |
| 普通文件 | ❌ 可篡改 | root 可随意修改，不安全 |
| REE 文件系统 | ❌ 不安全 | Linux 用户空间可写 |

> DJI 等产品的防回滚通常使用 **RPMB** + **OTP** 组合：
> - RPMB 存储当前版本号 (可递增)
> - OTP 存储最低版本号 (不可逆，用于紧急强制升级)

---

## 八、OTP/eFuse 密钥管理

### 8.1 OTP 物理原理

OTP (One-Time Programmable) 是芯片内部的一次性可写存储单元：

```c
// RV1126B OTP 控制器寄存器基址: 0x20804000
// drivers/nvmem/rockchip-otp.c

// 写入一次后，熔丝物理熔断，不能恢复到原始状态
// 用于存储:
//   1. 公钥 hash (用于验证 SPL 签名)
//   2. rollback-index 最小值
//   3. 芯片唯一 ID (UID)
//   4. 安全启动使能标志
```

### 8.2 密钥 HASH → OTP 流程

```
OpenSSL RSA 私钥
  │
  ├─ 提取公钥 (dev.pub)
  │
  ├─ SHA256(公钥) → Key Hash (32 bytes)
  │
  ├─ RK 脚本将 hash 写入 loader:
  │   rkbin/tools/rk_sign_tool 或 rkdeveloptool
  │   → rv1126b_spl_loader_v1.xx.xxx.bin (带 hash 的 loader)
  │
  ├─ 烧录到板子:
  │   rkdeveloptool db loader.bin
  │   rkdeveloptool ul loader.bin
  │   rkdeveloptool wl 0x40 uboot.img
  │   rkdeveloptool rd          ← ★ 触发 OTP 烧录
  │
  └─ 第 N 次启动时:
      BootROM 从 OTP 读取 key hash
      → 对比 loader 中声明的 key hash
      → 一致 → 允许启动
      → 不一致 → 拒绝启动 (变砖保护)
```

> **警告**：OTP 烧录是**不可逆**操作。一旦烧录公钥 hash，只有使用对应私钥签名的固件才能启动。如果私钥丢失，设备永久变砖。

### 8.3 Rockchip 安全烧录保护机制

```
开发者状态:
  ├─ 未锁芯片 (Unlocked)
  │   ├─ 可烧录任意固件
  │   ├─ 可通过 USB 下载
  │   └─ 串口登录无限制
  │
  └─ 已锁芯片 (Locked / Secure)
      ├─ 仅签名固件可启动
      ├─ USB 下载需身份验证
      ├─ 串口登录需密码
      └─ JTAG/SWD 端口禁用
```

Rockchip 通过 `CONFIG_ROCKCHIP_SECURE_MODE` 控制锁定状态：

```bash
# U-Boot 命令行查看安全状态
=> rockchip sec_status
# 输出:
#   Status: Unlocked (0x0000)       ← 未锁定
#   OTP:    Key hash NOT burned     ← OTP 未烧写
#   Mode:   Normal boot

# U-Boot 命令行锁定 (不可逆!)
=> rockchip sec_lock
# 输出:
#   Status: Locked (0x0001)
#   WARNING: This operation is irreversible!
```

---

## 九、Rockchip 安全启动 SDK 配置流程

### 9.1 完整安全启动配置步骤

```bash
# ====== 1. SDK menuconfig 启用安全特性 ======
make menuconfig

# Security feature → [*] Security feature (secureboot...)
#   Secureboot Method → (X) fit     (或 avb)
#   Security check method → (X) base (检查 loader/uboot/boot)
# [*] burn security key              (OTP 烧录)
# [*] security remote sign           (远程签名)

# 保存配置后，以下选项自动生效:
# RK_SECURITY=y
# RK_SECUREBOOT_FIT=y
# RK_SECURITY_CHECK_BASE=y
# RK_UBOOT_SPL=y (自动选中)

# ====== 2. 生成密钥对 ======
mkdir -p keys/spl keys/boot

# SPL 级别密钥
openssl genrsa -out keys/spl/dev.key 2048
openssl rsa -in keys/spl/dev.key -pubout -out keys/spl/dev.pub

# U-Boot 级别密钥
openssl genrsa -out keys/boot/dev.key 2048
openssl rsa -in keys/boot/dev.key -pubout -out keys/boot/dev.pub

# ====== 3. 配置密钥路径 ======
# u-boot/configs/rv1126b_sportcam_defconfig 添加:
# CONFIG_SPL_FIT_SIGNATURE=y
# CONFIG_SPL_FIT_SIGNATURE_KEYDIR="keys/spl"
# CONFIG_FIT_SIGNATURE=y
# CONFIG_FIT_SIGNATURE_KEYDIR="keys/boot"

# ====== 4. 设置双密钥 (不同级别用不同密钥) ======
# ITS 文件中:
# signature { key-name-hint = "boot"; }     # U-Boot 使用 boot 密钥
# SPL 编译时使用: mkimage -k keys/spl ...

# ====== 5. 编译 ======
./build.sh loader       # 生成带签名的 loader
./build.sh firmware     # 生成带签名的 boot.img

# ====== 6. 烧录到板子 ======
# 第 1 步: 普通烧录
sudo ./rkflash.sh all

# 第 2 步: 烧录 OTP (不可逆!)
# 进入 Maskrom 模式:
sudo ./rkflash.sh db        # 下载 loader
sudo ./rkflash.sh otp       # 烧录 OTP 密钥 hash
# 或手动:
sudo rkdeveloptool rd       # 重启到 BootROM
# WARNING: 此步之后只能启动签名固件！

# ====== 7. 验证 ======
# 重启板子，检查日志:
grep "fit verify\|rollback\|signature" /dev/kmsg
# 预期: fit verify configure passed
```

### 9.2 sportcam 当前配置分析

当前 `rv1126b_sportcam_defconfig`：

| 配置 | 值 | 作用 | 安全状态 |
|------|-----|------|---------|
| `CONFIG_AVB_LIBAVB` | y | AVB 库编译 | ✅ 基础设施 |
| `CONFIG_AVB_LIBAVB_AB` | y | A/B 分区 | ✅ |
| `CONFIG_AVB_LIBAVB_ATX` | y | Android Things | ✅ |
| `CONFIG_AVB_LIBAVB_USER` | y | 用户空间工具 | ✅ |
| `CONFIG_FIT_SIGNATURE` | **未设置** | FIT 签名验证 | ❌ 缺失 |
| `CONFIG_SPL_FIT_SIGNATURE` | **未设置** | SPL FIT 签名验证 | ❌ 缺失 |
| `RK_SECURITY` | **未设置** | 顶层安全开关 | ❌ 缺失 |

**ITS 文件已预设签名节点但 U-Boot 未启用验签 → 签名节点被 U-Boot 静默忽略。**

---

## 十、安全启动完整日志特征

### 10.1 验签通过

```
## Checking kernel ... sha256+ OK                     ← hash 校验
## Checking fdt ... sha256+ OK
## Checking resource ... sha256+ OK
fit verify configure passed                           ← ★ RSA 签名验证通过
## Loading kernel from FIT Image at 0x... ...
```

`sha256+` 中的 `+` 表示 RSA 签名校验通过（不仅仅 hash 匹配）。

### 10.2 验签失败

```
## Checking kernel ... sha256+ OK
fit verify configure failed, ret=-128                 ← ★ RSA 签名验证失败
```
或：
```
## Checking kernel ... sha256+ OK
fit verify configure required failed, ret=-128
```

### 10.3 防回滚拒绝

```
fit reject rollback: 0x01 < 0x03(min)                ← 固件版本过旧
```

### 10.4 密钥不匹配

```
No key found for configuration 'conf'                 ← U-Boot DTB 中无匹配公钥
fit verify configure required failed, ret=-2
```

---

## 十一、实验

### 11.1 实验 1：生成密钥并手动签名 FIT

```bash
# ====== PC 端 ======

# 1. 生成密钥
mkdir -p ~/secureboot-keys
cd ~/secureboot-keys

openssl genrsa -out dev.key 2048
openssl rsa -in dev.key -pubout -out dev.pub

# 2. 查看公钥
openssl rsa -pubin -in dev.pub -text -noout
# 记录 Modulus (256 bytes) 和 Exponent

# 3. 用 mkimage 签名 FIT
cd $SDK_ROOT

# -f boot.its: 使用 boot.its 描述文件
# -k ~/secureboot-keys: 密钥目录
# -r: required 模式 (验签失败则禁止启动)
# -E: 启用 external data
mkimage -f device/rockchip/.chips/rv1126b/boot.its \
        -k ~/secureboot-keys \
        -r \
        -E \
        output/firmware/boot_signed.img

# 4. 查看签名后的 FIT
mkimage -l output/firmware/boot_signed.img

# 输出中应包含:
#   Hash algo: sha256,rsa2048
#   Hash value: <签名值>
#   Sign images: fdt, kernel, multi
```

### 11.2 实验 2：验签验证

```bash
# ====== PC 端 ======

# 1. 尝试验签 (用 mkimage 的 -l 或 check 命令)
mkimage -l output/firmware/boot_signed.img

# 2. 篡改内核数据后验证
cp output/firmware/boot_signed.img boot_tampered.img
# 在 kernel 数据偏移处修改一个字节
printf '\x00' | dd of=boot_tampered.img bs=1 seek=$((0x1000)) conv=notrunc

# 3. 验证篡改镜像
mkimage -l boot_tampered.img
# 预期: 签名验证失败
```

### 11.3 实验 3：启用 FIT_SIGNATURE 编译 U-Boot

```bash
# ====== PC 端 ======

# 1. 创建密钥
mkdir -p $SDK_ROOT/keys/boot
cd $SDK_ROOT/keys/boot
openssl genrsa -out dev.key 2048

# 2. 修改 U-Boot defconfig
cd $SDK_ROOT/u-boot
cat >> configs/rv1126b_sportcam_defconfig << 'EOF'
CONFIG_FIT_SIGNATURE=y
CONFIG_FIT_SIGNATURE_KEYDIR="keys/boot"
EOF

# 3. 编译 U-Boot
cd $SDK_ROOT
./build.sh loader

# 4. 验证 U-Boot DTB 中包含公钥
fdtget u-boot/u-boot.dtb /signature
# 预期: key-dev { algo; required-conf; rsa,modulus; ... }
```

### 11.4 实验 4：防回滚测试

```bash
# ====== PC 端 ======

# 1. 修改 boot.its 的 rollback-index
# boot.its: rollback-index = <0x01>;  →  <0x02>;

# 2. 重新签名
mkimage -f boot.its -k ~/secureboot-keys -r boot_v2.img

# 3. 在板上启动
# 如果当前板上记录的 rollback-index 已经是 0x02，则 v2 正常启动
# 之后再启动 v1 (rollback-index=0x01) → 被拒绝
```

---

## 十二、DJI 场景下的安全启动

### 12.1 大疆无人机安全架构参考

根据大疆 BSP 分析文档 ([[dji-bsp-analysis]]) 的推断：

```
┌─────────────────────────────────────────────┐
│ DJI 安全启动链 (推断)                        │
├─────────────────────────────────────────────┤
│ BootROM (不可修改)                           │
│   └─ → 验证 SPL 签名 (OTP hash)             │
│        └─ → 验证 U-Boot 签名                │
│             └─ → 验证 Kernel 签名            │
│                  ├─ → 验证 VMM/AMP 固件签名   │
│                  └─ → 验证 rootfs dm-verity  │
│                       └─ → 验证用户 APP 签名  │
└─────────────────────────────────────────────┘
```

### 12.2 RV1126B → DJI 映射

| DJI 安全要求 | RV1126B 实现 | 差距与建议 |
|-------------|-------------|-----------|
| BootROM 验证 SPL | Rockchip OTP + 签名 loader | OTP 烧录流程相同 |
| 固件防降级 | rollback-index + RPMB | sportcam 当前为 0x00，需实现 RPMB |
| 产线密钥管理 | RK_SECURITY_REMOTE_SIGN | 远程签名可集成到 CI/CD |
| 签名镜像 | FIT signature + AVB | 需同时启用 FIT_SIGNATURE + AVB_BOOT |
| 加密存储 | OP-TEE + RPMB | sportcam 已启用 OP-TEE |
| JTAG 禁用 | OTP lock + eFuse 配置 | 需在 loader 中添加安全 config |

### 12.3 建议的 sportcam 安全启动升级路径

```
阶段 1 (当前):
  └─ 无签名验证，任何固件都可启动

阶段 2 (启用 FIT 签名):
  └─ 生成密钥 → 启用 CONFIG_FIT_SIGNATURE → 签名 boot.img
  └─ U-Boot 验证 boot.img 签名
  └─ OTP 不烧录 (开发模式，签名可选)

阶段 3 (SPL 签名 + 完整验证):
  └─ 双重密钥: SPL 密钥 + U-Boot 密钥
  └─ 启用 CONFIG_SPL_FIT_SIGNATURE
  └─ SPL 验证 U-Boot → U-Boot 验证 Kernel

阶段 4 (OTP 烧录 + 防回滚):
  └─ 烧录 OTP 公钥 hash (不可逆)
  └─ 配置 rollback-index + RPMB 存储
  └─ 启用 required 模式 (任何未签名固件不启动)

阶段 5 (产品级):
  └─ AVB + FIT 双验证
  └─ dm-verity rootfs 保护
  └─ 远程签名 + 产线自动化
  └─ 安全 JTAG/串口 策略
```

---

## 十三、代码仓库关键文件索引

| 文件路径 | 作用 | 核心函数/数据结构 |
|---------|------|-----------------|
| `common/image-fit.c` | FIT 镜像验证实现 | `fit_config_verify()`, `fit_image_check_sig()` |
| `common/image-sig.c` | 签名框架 | `image_sig_algos[]`, `fit_image_setup_sig()` |
| `lib/rsa/rsa-verify.c` | RSA 验签实现 | `rsa_verify_hash()`, `rsa_mod_exp()` |
| `lib/rsa/rsa-sign.c` | RSA 签名实现 (仅 mkimage 使用) | `rsa_sign_hash()` |
| `common/spl/spl_fit.c` | SPL FIT 加载 | `spl_load_simple_fit()`, `spl_fit_rollback_index()` |
| `common/avb/` | AVB 2.0 实现 | `avb_slot_verify()`, `avb_vbmeta_image_verify()` |
| `tools/mkimage.c` | mkimage 主工具 | uboot 编译产物，用于签名和打包 |
| `tools/image-fit-host.c` | mkimage 的 host FIT 处理 | `fit_set_sha256()`, `fit_image_add_sig()` |
| `drivers/nvmem/rockchip-otp.c` | RV1126B OTP 控制器驱动 | `rockchip_otp_read()`, `rockchip_otp_write()` |
| `arch/arm/mach-rockchip/rv1126b.c` | SoC 级安全初始化 | `arch_cpu_init()` 中的防火墙配置 |
| `device/rockchip/common/configs/Config.in.security` | Rockchip 安全 Kconfig | 安全特性菜单树 |

---

## 十四、思考题

1. **信任链起点**：BootROM 如何获取 SPL 公钥 hash？OTP 烧录时如何保证安全？

2. **双重签名**：SPL 和 U-Boot 使用不同的密钥对有什么好处？如果两个密钥使用同一个 RSA 私钥签名，安全上有什么隐患？

3. **RSA-PSS vs PKCS#1 v1.5**：为什么 U-Boot 默认使用 RSA-PSS 填充方案？两种方案在安全性上的主要区别是什么？

4. **external data 模式**：FIT 的 external data 模式下，签名是如何包含外部数据的？如果外部数据在 FIT 结构体之后，hash 时为什么还能覆盖到？

5. **防回滚局限性**：如果设备 OTP 已烧录且 rpmb 中记录了 rollback-index=5，攻击者能否通过物理手段替换 eMMC 来绕过防回滚？

6. **AVB vs FIT**：在 RV1126B 上同时启用 AVB 和 FIT 签名是否冗余？两种方案的保护范围分别是什么？

7. **密钥生命周期**：产品量产 100K 台，每台都烧录同一个公钥 hash。如果私钥在生产过程中泄露，如何在不报废设备的情况下恢复安全启动？

---

## 相关笔记

- [[bsp-boot-flow]] — 阶段一主笔记 (Boot Chain, FIT, 分区表)
- [[bsp-spl-fit]] — SPL FIT 镜像解析与验证源码追踪
- [[bsp-uboot-adaptation]] — U-Boot 板级适配与启动流程
- [[bsp-device-model-dtb]] — 阶段二: 设备模型 + 设备树
- [[dji-bsp-analysis]] — 大疆嵌入式技术栈深度分析
- [[MOC-嵌入式Linux]] — 嵌入式 Linux 学习路线 MOC
