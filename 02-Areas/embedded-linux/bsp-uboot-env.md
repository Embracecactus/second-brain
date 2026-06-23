---
tags:
  - embedded-linux
  - bsp
  - bootloader
  - u-boot
  - environment
  - uboot-env
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

# U-Boot 环境变量系统深度解析

> **前置笔记**：[[bsp-boot-flow]] — U-Boot 命令行交互 & bootargs
>
> **前置笔记**：[[bsp-uboot-adaptation]] — U-Boot 板级适配 (Defconfig 配置)
>
> 本文聚焦 U-Boot 环境变量的存储、加载、运行时管理机制，以及在 RV1126B sportcam 上的具体实现。

---

## 一、环境变量系统架构

### 1.1 环境变量的生命周期

```
U-Boot 编译时
  │
  ├─ 默认环境变量: CONFIG_EXTRA_ENV_SETTINGS (在 .h 头文件中定义)
  │
  ├─ 被编译到 U-Boot 二进制中: "default environment"
  │
  └─ 第一次启动 → 检查存储介质中是否有校验有效的 env
       ├─ 有 → 加载到 RAM 中
       └─ 无 → 使用编译时的 default environment
            └─ saveenv → 写入存储介质

运行时:
  ├─ setenv var value            → 修改 RAM 中的 env
  ├─ saveenv                     → 写入存储介质 (持久化)
  ├─ env default -f -a           → 恢复出厂默认
  ├─ env export                  → 导出到内存
  └─ env import                  → 从内存导入
```

### 1.2 RV1126B sportcam 的环境变量源

当前 sportcam 的环境变量定义链：

```
rv1126b_common.h:98
  └─ CONFIG_EXTRA_ENV_SETTINGS
       │
       ├─ ENV_MEM_LAYOUT_SETTINGS     → 内存地址变量
       │   scriptaddr=0x40600000
       │   fdt_addr_r=0x48300000
       │   kernel_addr_r=0x40200000
       │   kernel_addr_c=0x45480000
       │   ramdisk_addr_r=0x4a200000
       │
       ├─ "partitions=" PARTS_RKIMG   → 自动分区检测
       │
       ├─ ROCKCHIP_DEVICE_SETTINGS    → Rockchip 设备变量
       │
       ├─ RKIMG_DET_BOOTDEV           → 启动设备检测
       │
       └─ BOOTENV                     → distro bootcmd (含 boot_fit)
```

---

## 二、环境变量存储机制

### 2.1 U-Boot 支持的存储后端

| 后端 | Kconfig 开关 | EMMC 位置 | 典型访问时间 |
|------|-------------|-----------|------------|
| MMC | `CONFIG_ENV_IS_IN_MMC` | eMMC 指定扇区 | ~5ms |
| FAT | `CONFIG_ENV_IS_IN_FAT` | FAT 分区中的文件 | ~10ms |
| EXT4 | `CONFIG_ENV_IS_IN_EXT4` | ext4 分区中的文件 | ~10ms |
| SPI Flash | `CONFIG_ENV_IS_IN_SPI_FLASH` | SPI NOR/NAND | ~20ms |
| NOWHERE | `CONFIG_ENV_IS_NOWHERE` | 无存储 (仅 RAM) | 0ms |

### 2.2 sportcam 的 eMMC 环境变量布局

```kconfig
# u-boot/configs/rv1126b_sportcam_defconfig (未显式设置, 从架构默认继承)
CONFIG_ENV_IS_IN_MMC=y
# CONFIG_ENV_OFFSET=0x3F8000         # 环境变量在 eMMC 中的偏移 (扇区单位)
# CONFIG_ENV_SIZE=0x2000             # 8KB 环境变量空间
# CONFIG_ENV_SECT_SIZE=0x200         # 512 字节扇区
```

eMMC 布局中的环境变量位置：

```
eMMC 块 0:
┌──────────────────────────────────┐
│ GPT (Protective MBR + GPT Header)│  0x0000 - 0x3FFF (LBA 0-33)
├──────────────────────────────────┤
│ uboot 分区                       │  0x4000 - 0x5FFF (LBA 0x4000)
│   ├─ MiniLoaderAll.bin           │
│   ├─ uboot.img (FIT)             │
│   ├─ trust.img                   │
│   └─ 环境变量块 (env)            │  0x... ← CONFIG_ENV_OFFSET
├──────────────────────────────────┤
│ misc 分区                        │  0x6000 - 0x7FFF
├──────────────────────────────────┤
│ boot 分区                        │  0x8000 - 0x27FFF
├──────────────────────────────────┤
│ ...                              │
└──────────────────────────────────┘
```

> **重要**：环境变量存储在 uboot 分区内（FIT 镜像之后），不在单独的分区中。这意味着 `dd if=uboot.img of=/dev/mmcblk0p1` 会**覆盖环境变量**！

### 2.3 冗余环境变量

Rockchip 默认使用**单份环境变量**：

```kconfig
# 启用冗余环境变量 (可选):
CONFIG_ENV_IS_IN_MMC=y
# CONFIG_ENV_OFFSET_REDUND=0x3FC000  # 冗余副本偏移
# 如果是 redundant:
# env_inval() 检查主副本 → CRC 有效? 使用 : 检查冗余副本 → 主副本损坏时自动恢复
```

### 2.4 环境变量存储格式

```
MMC 中存储的环境变量格式:

┌──────────────────────────────────────┐
│ CRC32 (4 bytes)                     │ ← 校验整个 env 数据
├──────────────────────────────────────┤
│ 标志 (1 byte)                       │ ← redundant 有效标志
├──────────────────────────────────────┤
│ env 数据 (CONFIG_ENV_SIZE - 4 bytes) │
│   var1=value1\0                     │
│   var2=value2\0                     │
│   ...                               │
│   \0                                │ ← 终止符
└──────────────────────────────────────┘
```

---

## 三、环境变量分类

### 3.1 预定义变量 (编译时)

| 变量名 | 值 (sportcam) | 来源 | 作用 |
|--------|-------------|------|------|
| `scriptaddr` | 0x40600000 | `ENV_MEM_LAYOUT_SETTINGS` | U-Boot 脚本加载地址 |
| `fdt_addr_r` | 0x48300000 | 同上 | DTB 运行时地址 |
| `kernel_addr_r` | 0x40200000 | 同上 | Kernel 加载地址 |
| `kernel_addr_c` | 0x45480000 | 同上 | Kernel 压缩地址 |
| `ramdisk_addr_r` | 0x4a200000 | 同上 | Ramdisk 地址 |
| `bootargs` | (来自 DTS chosen) | DTS/U-Boot | 内核命令行参数 |
| `bootcmd` | `boot_fit;` 或 `boot_android;` | `RKIMG_BOOTCOMMAND` | **自动启动命令** |
| `preboot` | (空) | `CONFIG_PREBOOT` | **启动前钩子** |
| `partitions` | PARTS_RKIMG | `rockchip-common.h` | 分区检测脚本 |
| `devtype` | mmc | 自动检测 | 启动设备类型 |
| `devnum` | 0 | 自动检测 | 启动设备编号 |

### 3.2 运行时变量 (板端查看)

```bash
# 在 U-Boot shell 中查看:
=> printenv
# 输出: 所有当前环境变量

# 只看特定变量:
=> printenv bootargs
# bootargs=earlycon=uart8250,mmio32,0x20810000 ...

# 查看自动检测的变量:
=> printenv devtype devnum fdtcontroladdr
# devtype=mmc
# devnum=0
# fdtcontroladdr=0x40600000

# 板端可通过 /proc/cmdline 查看内核收到的最终 bootargs
cat /proc/cmdline
```

### 3.3 核心变量详解

**`bootcmd` — 自动启动命令**

```bash
# sportcam 的 bootcmd (RKIMG_BOOTCOMMAND):
# 如果 CONFIG_FIT_SIGNATURE=y:
boot_fit;
```

**`bootargs` — 内核命令行参数**

```bash
# 来源链: DTS chosen.bootargs → U-Boot 读取 → 可被 setenv 覆盖 → 传递给内核
# 优先级:
# 1. CONFIG_BOOTARGS (编译时)
# 2. DTS chosen.bootargs (当前 sportcam 使用)
# 3. U-Boot env bootargs (运行时 setenv, 优先级最高)

# 在 U-Boot shell 中动态修改:
=> setenv bootargs 'console=ttyFIQ0 root=/dev/mmcblk0p7 rw initcall_debug'
=> boot    # 用新参数启动
# 注意: 如果不 saveenv, 重启后丢失
```

**`preboot` — 启动前钩子**

```bash
# sportcam 已定义 CONFIG_PREBOOT 但值为空
# 可用于在 autoboot 前执行检查:

# 示例: 检查按键进入 USB 下载模式
=> setenv preboot 'if gpio input 100; then run usb_boot; fi'
=> saveenv

# 此时每次启动时先检查 GPIO100, 如果为高则进入 USB 下载模式
```

---

## 四、环境变量操作命令

### 4.1 完整命令集

```bash
# 基本操作
=> printenv [var]             # 打印环境变量
=> setenv var value           # 设置变量
=> setenv var                 # 删除变量 (不指定值)
=> saveenv                    # 保存到存储介质
=> saveenv 0x4000 0x2000      # (某些版本) 指定偏移和大小保存

# 默认环境
=> env default -f -a          # 强制恢复出厂默认 (所有变量)
=> env default var            # 恢复单个变量为默认值

# 导入导出 (用于脚本/升级)
=> env export -t [addr]       # 导出 env 到内存 (文本格式)
=> env export -b [addr]       # 导出 env 到内存 (二进制格式)
=> env import -t [addr] [size] # 从内存导入 env (文本格式)

# 擦除
=> env erase                  # 擦除存储介质中的 env
# 下次启动时使用默认 env, 然后自动重新保存

# 校验
=> env info                   # 显示环境变量存储状态
# 输出: "Environment: env is stored in MMC, CRC OK"
```

### 4.2 环境变量脚本编程

U-Boot 环境变量支持简单的脚本功能：

```bash
# 条件执行 (基于 ${var} 的值)
=> setenv check_boot '
    if test ${reboot_mode} = recovery; then
        echo "Booting recovery...";
        run recovery_cmd;
    elif test ${reboot_mode} = fastboot; then
        echo "Entering fastboot...";
        run fastboot_cmd;
    else
        echo "Normal boot...";
        run normal_boot;
    fi'
=> saveenv

# 循环
=> setenv loop_test '
    i=0;
    while test $i -lt 10; do
        echo "Count: $i";
        i=$((i+1));
    done'

# 数学运算
=> setenv calc '
    setexpr val1 0x100 + 0x200;
    echo "Result: ${val1}";'   # → Result: 0x300

# 字符串操作
=> setenv name "RV1126B"
=> setenv msg 'Hello ${name}!'
=> echo ${msg}                # → Hello RV1126B!
```

### 4.3 常见操作场景

**场景 1：修改 bootargs 调试**

```bash
# 进入 U-Boot shell (启动时按任意键)
=> setenv bootargs '${bootargs} initcall_debug loglevel=8'
=> saveenv
=> boot

# 验证:
cat /proc/cmdline | grep initcall_debug
# 预期: 包含 initcall_debug loglevel=8
```

**场景 2：设置静态 IP 用于网络启动**

```bash
=> setenv ipaddr 192.168.1.100
=> setenv serverip 192.168.1.1
=> setenv gatewayip 192.168.1.1
=> setenv netmask 255.255.255.0
=> saveenv

# 网络启动:
=> tftp 0x40200000 boot.img
=> bootm 0x40200000
```

**场景 3：A/B 分区切换**

```bash
# 如果启用了 CONFIG_SPL_AB:
=> setenv boot_part 1     # 尝试从 boot_a 启动
=> saveenv
=> boot

# 如果 boot_a 启动失败, SPL 自动切换到 boot_b
```

**场景 4：USB 下载模式强制进入**

```bash
# 方式 1: U-Boot shell 中直接进入
=> rockusb 0 0x20000000 0x20000000

# 方式 2: 设置环境变量, 下次启动自动进入
=> setenv bootcmd 'rockusb 0 0x20000000 0x20000000'
=> saveenv
=> reset
```

---

## 五、`bootcmd` 执行链深度解析

### 5.1 sportcam 的 bootcmd 链

```bash
# 实际执行链 (简化):
bootcmd → "boot_fit; boot_android ${devtype} ${devnum};"
  │
  ├─ boot_fit:
  │   └─ 从 boot 分区加载 boot.img (FIT 格式)
  │       → 验签 → 解压 → bootm
  │
  └─ (如果 boot_fit 失败) boot_android:
      └─ 从 boot 分区加载 android boot image
```

### 5.2 U-Boot 正常启动流程

```
上电 → SPL → U-Boot proper
  │
  ├─ board_init_r() 初始化完成
  │
  ├─ run_main_loop()
  │    │
  │    ├─ cli_init()           ← 初始化命令行接口
  │    │
  │    ├─ run_preboot()        ← ★ 执行 preboot 变量中的命令
  │    │                         (sportcam: preboot 为空, 跳过)
  │    │
  │    ├─ autoboot_command()   ← ★ 自动启动
  │    │    │
  │    │    ├─ CONFIG_BOOTDELAY=0 → 无延迟, 直接启动
  │    │    │
  │    │    └─ run_command(bootcmd)  ← ★ 执行 bootcmd
  │    │         │
  │    │         └─ boot_fit; boot_android;
  │    │              │
  │    │              └─ bootm → Linux Kernel
  │    │
  │    └─ cli_loop()           ← autoboot 失败/打断后进入 shell
  │
  └─ [此处用户交互: 串口输入命令]
```

---

## 六、运行时环境变量覆盖

### 6.1 从内核启动参数修改 U-Boot 环境变量

Linux 内核可以通过 `bootargs` 修改 U-Boot 的环境变量行为：

```bash
# 在 Linux 中修改 U-Boot 环境变量 (需要 fw_env 工具)

# 1. board 安装 fw_printenv/fw_setenv:
# 编译 U-Boot 的 fw_env 工具:
cd u-boot
make envtools

# 2. 配置 /etc/fw_env.config:
# /etc/fw_env.config
# Device     Offset       Size     Sector Size
/dev/mmcblk0 0x3F8000    0x2000   0x200

# 3. 在 Linux 中读写 U-Boot 环境变量:
fw_printenv bootargs
fw_setenv bootargs '${bootargs} quiet'
fw_setenv bootdelay 5
```

### 6.2 bootcount 保护机制

```bash
# 启动计数: 限制启动失败重试次数
CONFIG_BOOTCOUNT=y
CONFIG_BOOTCOUNT_LIMIT=3         # 最多重试 3 次

# 流程:
# 1. 每次启动 bootcount++
# 2. 用户空间启动成功后执行: fw_setenv bootcount 0
# 3. 如果 bootcount > 3 → 自动恢复出厂设置

# 板端用户空间脚本:
# /etc/init.d/S99reset_bootcount
#!/bin/sh
fw_setenv bootcount 0
# → 每次系统成功启动后重置计数
```

---

## 七、环境变量调试

### 7.1 常见问题排查

| 问题 | 原因 | 解决 |
|------|------|------|
| `setenv` 后重启变化 | 没有 `saveenv` | 执行 `saveenv` |
| `saveenv` 报错 `Writing MMC failed` | eMMC 写保护/坏块 | `mmc dev 0; mmc wp` 检查 |
| 修改 bootargs 后内核没收到 | U-Boot DTS 中的 chosen 覆盖 | 删 DTS 中的 bootargs 或用 setenv |
| `env default -f -a` 后 bootcmd 恢复 | 恢复默认 env | 重新 setenv + saveenv |
| 板子一直进入 shell | CONFIG_BOOTDELAY > 0 | 改为 0 或按需 |
| `printenv` 显示乱码 | env 存储 CRC 校验失败 | `env erase; saveenv` |

### 7.2 环境变量损坏恢复

```bash
# 场景: env CRC 校验失败, U-Boot 打印 "*** Warning - bad CRC, using default environment"

# 方法 1: 自动恢复 (U-Boot 自动使用默认 env, 执行 saveenv 即可)
=> saveenv    # 将默认 env 写回存储

# 方法 2: 手动擦除
=> env erase
=> boot        # 使用默认 env 启动
# 进入 Linux 后:
fw_setenv bootargs '...'
fw_setenv bootcmd 'boot_fit;'

# 方法 3: 完全重新初始化
=> env default -f -a
=> saveenv
```

---

## 八、RV1126B OTP 环境变量

RV1126B 的 OTP 也存储了部分"准环境变量"：

```c
// u-boot/include/configs/rv1126b_common.h:43-66

// 防回滚版本号 (2 words = 64 bits)
#define OTP_UBOOT_ROLLBACK_OFFSET    0x310

// 安全启动使能 (1 byte)
#define OTP_SECURE_BOOT_ENABLE_ADDR  0x20

// RSA 公钥 hash (32 bytes)
#define OTP_RSA_HASH_ADDR            0x180

// USB/串口/SD 禁用控制
#define OTP_DISABLE_USB_VAL          0x3
#define OTP_DISABLE_SD_VAL           0xc
#define OTP_DISABLE_UART_VAL         0x30
```

这些 OTP 区域的读写方式：

```bash
# U-Boot shell:
=> otp read 0x20          # 读取安全启动使能标志
=> otp read 0x180 32      # 读取 RSA 公钥 hash

# Linux (需要 OP-TEE 支持):
# /sys/bus/nvmem/devices/rockchip-otp0/nvmem
hexdump -C /sys/bus/nvmem/devices/rockchip-otp0/nvmem | head -20
```

---

## 九、sportcam 环境变量实战

### 9.1 查看当前环境变量

```bash
# 方式 1: 串口进入 U-Boot shell
# 启动时按任意键中断 autoboot
=> printenv

# 方式 2: Linux 中查看 bootargs (内核收到的最终值)
cat /proc/cmdline

# 方式 3: 查看 DTS 中的默认 bootargs
dtc -I dtb -O dts /sys/firmware/fdt | grep chosen -A 10
```

### 9.2 自定义启动行为示例

```bash
# 场景: 实现"长按电源键进入恢复模式"

# U-Boot shell:
=> setenv preboot 'if gpio input 117; then run recovery_boot; fi'
=> setenv recovery_boot 'setenv bootargs ${bootargs} recovery_mode; boot_fit;'
=> saveenv

# 这样上电时如果 GPIO117 (假设为电源键) 为高, 进入恢复模式
# GPIO117 对应的 DTS 节点: &gpio3 { /rockchip,gpio-bank=3; gpio117 = <&gpio3 21 ...>; }
```

### 9.3 启动日志分析

```bash
# 环境变量加载时的 U-Boot 日志:
# 正常情况 (无输出, 静默加载)
# 环境变量损坏: "*** Warning - bad CRC, using default environment"
# 冗余 env 自动恢复: "*** Warning - no valid environment area found"
#                     "*** Warning - env CRC is invalid, swap fail"

# 检查 env 在 MMC 中的存储:
=> mmc read 0x40600000 0x3F80 0x40   # 读取 env 扇区到内存
=> md 0x40600000                      # 查看原始数据 (第一个 word 是 CRC32)
```

---

## 十、思考题

1. **环境变量 vs 配置文件**：U-Boot 为什么不直接用配置文件（如 extlinux.conf）而要用环境变量这种基于 key-value 的存储方式？两种方案的优缺点是什么？

2. **默认 env 的更新**：如果 SDK 更新了 `rv1126b_common.h` 中的 `CONFIG_EXTRA_ENV_SETTINGS`，已部署的板子上已有保存的环境变量不会自动更新。如何设计升级策略来确保环境变量安全更新？

3. **`preboot` 的用途**：`preboot` 在 autoboot 之前执行，常用于检测按键或下载模式。如果在 `preboot` 中执行长时间操作（如网络检测），会延迟启动。你会如何设计 `preboot` 使其既满足功能需求又不会明显增加启动时间？

4. **env CRC 校验失败**：如果 eMMC 的某位翻转导致 env 的 CRC 校验失败，U-Boot 会使用默认 env。这种情况下系统还能正常启动吗？有什么潜在风险？

5. **OTP 作为"准环境变量"**：OTP 中存储的 RSA hash、安全启动标志等与普通环境变量有什么本质区别？为什么这些信息不直接放在 MMC 的环境变量中？

---

## 相关笔记

- [[bsp-boot-flow]] — Boot Chain 全景 & U-Boot 命令行
- [[bsp-uboot-adaptation]] — U-Boot 板级适配 (Defconfig)
- [[bsp-uboot-secureboot]] — 安全启动 (OTP 密钥存储)
- [[bsp-uboot-boottime]] — 启动速度 (preboot 优化)
- [[bsp-uboot-rktools]] — Rockchip 工具链 (env 烧录/备份)
- [[MOC-嵌入式Linux]] — 嵌入式 Linux 学习路线 MOC
