---
tags:
  - embedded-linux
  - bsp
  - bootloader
  - u-boot
  - bringup
  - debug
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

# Board Bring-up 实战诊断指南

> **前置笔记**：[[bsp-boot-flow]] — 正常启动流程 & 日志特征
>
> **前置笔记**：[[bsp-uboot-adaptation]] — 板级适配 Checklist
>
> **本文假设**：你刚拿到一块新打的 RV1126B 板子，eMMC 已烧录固件，上电后串口无输出或行为异常。

---

## 一、故障诊断层次

```
层次 1: 完全不启动 (串口无输出)
  └─ 电源/时钟/复位/DDR 问题
       │
层次 2: 有输出但卡在早期阶段
  ├─ BootROM 有响应 → SPL 崩溃
  ├─ SPL 打印后卡住 → FIT 加载/验签失败
  └─ U-Boot 打印后卡住 → 驱动 probe 失败
       │
层次 3: boot 完成但 kernel 崩溃
  └─ DTB 不匹配/驱动问题/bootargs 错误
       │
层次 4: kernel 启动但 userspace 有问题
  └─ rootfs 损坏/init 失败/服务崩溃
```

---

## 二、层次 1：完全不启动

### 2.1 串口无输出的 5 步排查

```
Step 1: 检查电源
  ├─ 测量各 rail:
  │   VCC_3V3 (3.3V)       → PMIC 主输出
  │   VCC_1V8 (1.8V)       → DDR 和 IO 电压
  │   VCC_DDR (1.35V)      → LPDDR4 专用
  │   VCC_CORE (0.9V)      → CPU 核心电压
  │   VCC_1V0 (1.0V)       → PLL/PHY 电压
  ├─ 看 PMIC RK801 的 IRQ 引脚是否指示故障
  └─ 检查上电时序: PMIC EN → VCC_3V3 → VCC_1V8 → VCC_DDR → VCC_CORE

Step 2: 检查时钟
  ├─ 24MHz 晶振: 示波器测量 XIN24M/XOUT24M
  │   (应看到 24MHz ±100ppm 正弦波)
  ├─ 32.768KHz RTC 晶振 (可选, 影响 RTC)
  └─ DDR 参考时钟 (1332MHz 需要稳定的参考源)

Step 3: 检查复位
  ├─ RESET 引脚: 上电后应为高电平
  ├─ POR (Power-On Reset): 确认持续时间 > 10ms
  └─ WDT (看门狗): 如果 WDT 复位, RESET 引脚有脉冲

Step 4: 检查 BootROM 是否运行
  ├─ 测量 eMMC CLK 引脚:
  │   BootROM 启动时应有 400KHz 时钟脉冲
  ├─ 如果无时钟 → BootROM 未执行 / 卡在 DDR 初始化
  └─ 如果有时钟 → BootROM 尝试读取 eMMC boot 分区

Step 5: 检查 eMMC
  ├─ 测量 eMMC CMD 引脚: 应有双向通信
  ├─ 确认 eMMC 供电: VCC (3.3V) 和 VCCQ (1.8V)
  └─ 如果 eMMC 为全新/空片: 通过 Maskrom 烧录
```

### 2.2 Maskrom 模式验证

```bash
# 确认板子进入 Maskrom 的条件:
# 1. eMMC 无有效 bootloader
# 2. eMMC CLK 和 GND 短接 (强制 Maskrom)
# 3. 按住 MASKROM 按键上电

# PC 端检查:
lsusb | grep -i rockchip
# 预期: "ID 2207:350a Rockchip"
# 如果 lsusb 看不到 → USB 连接或 VBUS 问题

# 如果能识别:
rkdeveloptool db rv1126b_spl_loader_v1.09.105.bin
# 如果成功: 返回 "Download boot OK"
# 如果失败: "Device not found" → 检查 USB 线/驱动

# 下载 loader 后:
rkdeveloptool poll
# 如果返回 chip info → 板子确认进入 Loader 模式
```

---

## 三、层次 2：有输出但卡住

### 3.1 卡在 SPL 之前

| 现象 | 诊断 | 解决 |
|------|------|------|
| 仅有`U-Boot SPL board init`后无输出 | DM 初始化 hang | 检查 `u-boot,dm-spl` DTS 标记 |
| `Trying fit image at 0x4000 sector`后无输出 | eMMC 读取失败 | 检查 eMMC 焊接, IOMUX 配置 |
| `Not fit magic` | FIT 损坏/偏移错误 | 重新烧录 boot.img, 检查分区偏移 |
| `DRAM init failed` | DDR 颗粒不匹配 | 换 DDR bin 或联系 Rockchip |

**DDR 问题详细诊断**:

```bash
# RV1126B 的 DDR 初始化在闭源 TPL 中完成
# 如果在 TPL 阶段崩溃 → 根本打印不出任何日志

# 常见 DDR 问题:
# 1. 颗粒型号不匹配 (DDR bin 是针对特定颗粒的)
# 2. 容量设置不对 (512MB vs 1GB/2GB)
# 3. 频率不兼容 (1332MHz vs 1600MHz)
# 4. PCB 走线不等长 → 时序偏移

# 排查:
# 使用 ddrbin_tool.py 检查当前 DDR bin
python3 rkbin/tools/ddrbin_tool.py -p ddrbin_param.txt \
    rkbin/bin/rv11/rv1126b_ddr_1332MHz_v1.09.bin

# ddrbin_param.txt 中的关键参数:
# ddr_freq=1332            # DDR 频率
# ddr_type=3               # 0=DDR3, 1=DDR4, 2=LPDDR3, 3=LPDDR4
# ddr_capacity=512          # 总容量 MB
# ddr_chip="hynix"         # 颗粒品牌
```

### 3.2 SPL FIT 加载卡住

```bash
# SPL FIT 加载卡住的典型场景:
# Trying fit image at 0x4000 sector
# ## Checking atf-1 0x40000000 (lzma @0x40200000) ...
# [卡住]

# 原因: LZMA 解压超时 / SPL stack overflow / 加载地址冲突

# 检查 SPL 加载地址是否冲突:
# SPL 运行地址: 0x4FE00000
# U-Boot 加载地址: 0x40200000
# Kernel 加载地址: 0x40200000 (与 U-Boot 相同!)

# 冲突排查:
# SPL 加载 U-Boot 到 0x40200000
# U-Boot 重定位后 kernel_addr_r = 0x40200000
# → 如果 SPL 堆栈在 0x4FE00000 附近向下增长
# → 可能踩到加载的 U-Boot 数据

# 修复: 调整 kernel_addr_r 到不同地址
# rv1126b_common.h 中:
# "kernel_addr_r=0x40200000\0"  →  "kernel_addr_r=0x42000000\0"
```

### 3.3 ATF BL31 卡住

```
最后一条 SPL 日志: ## Checking fdt 0x41000000 ... sha256+ OK
之后无输出 (约 20ms 无日志)
  │
  ├─ 正常: ATF BL31 在 EL3 初始化 (GIC/MMU/PSCI)
  │         之后跳转到 U-Boot, 输出 "U-Boot 2017.09-..."
  │
  └─ 异常: 超过 100ms 仍无输出
       ├─ ATF entry point 错误
       │   boot.its 中 atf-1 的 load 地址 ≠ 实际 ELF entry
       ├─ OP-TEE (BL32) 崩溃
       │   尝试禁用 BL32: trust_merger 中去掉 BL32_OPTION
       └─ MMU 配置错误 (ATF 中页表映射)
```

### 3.4 U-Boot proper 卡住

```
U-Boot 2017.09-g... (Jun 22 2026 - 17:00:00)
  │
  ├─ 打印版本号后卡:
  │    → env 加载失败 (MMC 读取 env 扇区出错)
  │    解: "env erase" 或检查 MMC 驱动
  │
  ├─ 打印 "Model: ..." 后卡:
  │    → 某个驱动 probe 失败 (I2C/PMIC)
  │    解: 在 DTS 中暂时 disable 问题设备
  │
  ├─ 打印 "DRAM: 512 MiB" 后卡:
  │    → MMC 卡识别超时 (mmc_init 阻塞)
  │    解: 检查 eMMC 供电或 DTS 中的 bus-width
  │
  └─ 打印 "Net: eth0: ..." 后卡:
       → 网卡驱动 probe 失败/等待 PHY 自动协商
       解: 在 DTS 中 disable 以太网节点
```

**快速定位问题驱动的技巧**:

```bash
# 方法 1: 二分禁用 DTS 节点
# 在 DTS 中逐个 disable 设备:
&i2c1 { status = "disabled"; };
&usbdrd { status = "disabled"; };
&gmac { status = "disabled"; };

# 方法 2: 在 defconfig 中禁用驱动
# CONFIG_PMIC_RK8XX is not set
# CONFIG_DM_ETH is not set

# 方法 3: 使用 CONFIG_BOOTDELAY=0 跳过 autoboot
# 逐条运行命令:
=> mmc dev 0
=> mmc info
=> i2c bus
=> ...
```

---

## 四、层次 3：Kernel 启动失败

### 4.1 内核完全无输出

```
Starting kernel ...    ← 最后一条 U-Boot 日志
[无任何内核日志]
```

| 可能原因 | 排查 |
|---------|------|
| DTB 损坏 | `mkimage -l boot.img` 检查 FIT |
| DTB 不兼容 | 确认 `CONFIG_DEFAULT_DEVICE_TREE` 正确 |
| ATF 未正确返回 EL2 | 检查 ATF entry point |
| Kernel Image 损坏 | 重新编译/检查 md5sum |
| bootargs 错误 | `printenv bootargs` 确认参数 |

```bash
# 验证 DTB 兼容性:
# 在板端 U-Boot shell:
=> fdt addr 0x48300000          # DTB 加载地址
=> fdt list /                   # 查看根节点
# 确认 compatible = "alientek,rv1126b-sportcam", "rockchip,rv1126b"
```

### 4.2 Kernel panic / Oops

```
[    0.123456] Kernel panic - not syncing: ...
```

```bash
# 常见原因:
# 1. "VFS: Unable to mount root fs"
#    → root=PARTUUID 错误或 rootfstype 不对
#    解: 检查 `blkid /dev/mmcblk0p7` PARTUUID

# 2. "No working init found"
#    → /sbin/init 不存在或无法执行
#    解: bootargs 加 `init=/bin/sh` 进入 shell

# 3. "Internal error: synchronous external abort"
#    → 驱动访问了未映射的内存地址
#    解: 检查 DTS reg 属性中的地址范围

# 4. "rcu_sched self-detected stall"
#    → 驱动在 probe 中 spinlock 死锁
#    解: 检查驱动代码或添加 cond_resched()
```

---

## 五、层次 4：Rootfs 启动失败

```bash
# init 进程崩溃 → kernel 重启
# 跳过 init 直接进 shell 诊断:
# bootargs 添加: init=/bin/sh

# 然后手动挂载:
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -o remount,rw /

# 检查关键文件:
ls -la /sbin/init
ldd /sbin/init               # 检查动态库依赖
cat /etc/inittab              # BusyBox init 配置
systemctl status              # systemd 状态
```

---

## 六、Bring-up 排查工具箱

### 6.1 最小 U-Boot 配置

```kconfig
# 最简 U-Boot 配置 (排除所有可能出问题的驱动)
# 启用后只保留:
CONFIG_DM=y
CONFIG_DM_MMC=y
CONFIG_DM_SERIAL=y
# CONFIG_DM_I2C is not set
# CONFIG_DM_ETH is not set
# CONFIG_DM_USB is not set
# CONFIG_PMIC_RK8XX is not set
CONFIG_BOOTDELAY=0
```

### 6.2 有用的一组 U-Boot 调试命令

```bash
# 1. 检查板子基本状态
bdinfo          # 板级信息 (DDR 大小/地址, 时钟)
clk dump        # 时钟树打印

# 2. 检查 eMMC
mmc dev 0       # 切换到 eMMC
mmc info        # eMMC 信息
mmc part        # 分区表

# 3. 从 eMMC 读取并验 FIT
load mmc 0:3 0x42000000 boot.img   # 从 boot 分区加载
iminfo 0x42000000                  # FIT 镜像头信息

# 4. 检查 I2C 总线
i2c bus         # 列出所有 I2C 总线
i2c dev 0       # 切换到 bus 0
i2c probe       # 扫描总线上的设备

# 5. 内存测试
mtest 0x40000000 0x50000000        # DDR 压力测试

# 6. 检查 GPIO
gpio status -a  # 查看所有 GPIO 状态

# 7. 寄存器转储
md 0x20000000 0x10  # 读取 CRU 寄存器 (时钟)
md 0x20200000 0x10  # 读取 GRF 寄存器 (配置)
md 0x20300000 0x10  # 读取 DW MMC 寄存器
```

---

## 七、4 个实战场景

### 场景 1: 新板串口无输出

```bash
# 排查步骤:
1. 万用表测 VCC_3V3, VCC_1V8, VCC_DDR, VCC_CORE
2. 示波器测 24MHz 晶振
3. 测 eMMC CLK (应有 400KHz BootROM 时钟)
4. 如果以上都 OK → 进 Maskrom
5. rkdeveloptool db loader.bin → 如果成功 → DDR OK
6. rkdeveloptool wl 0x40 uboot.img → 重烧
7. 如果还是无输出 → 换串口芯片/检查 TX/RX 交叉
```

### 场景 2: DDR 初始化失败

```bash
# 特征: 上电后 500ms 无任何输出 (BootROM + DDR Init)
# 原因: DDR 颗粒型号不匹配

# 解决:
1. 确认 eMMC 焊盘上的 DDR 颗粒型号
2. 查找 SDK rkbin/bin/rv11/ 中是否有匹配的 DDR bin
3. 如果没有, 用 ddrbin_tool.py 调整参数
4. 或联系 Rockchip FAE 获取匹配 bin
5. 重新 merge loader:
   boot_merger RKBOOT/RV1126BMINIALL.ini
```

### 场景 3: U-Boot 加载 FIT 卡死

```bash
# 特征: "Trying fit image at 0x4000 sector" 后卡死
# 原因: SPL 地址冲突

# 排查:
# 检查 kernel_addr_r 是否与 SPL 加载地址重叠
# SPL 栈顶: 0x4FE00000 → 向下增长
# U-Boot: 0x40200000
# kernel_addr_r: 0x40200000 ← 冲突!

# 解决:
# 修改 rv1126b_common.h 中 kernel_addr_r:
# "kernel_addr_r=0x42000000\0"
```

### 场景 4: kernel 启动后找不到 rootfs

```bash
# 特征: "VFS: Unable to mount root fs on unknown-block(179,7)"
# 原因: PARTUUID 或 rootfstype 不匹配

# 排查:
cat /proc/cmdline    # 检查内核收到的 bootargs
lsblk                # 检查分区
blkid                # 检查 PARTUUID

# 解决:
# 1. 在 U-Boot 中修正 bootargs:
setenv bootargs 'root=/dev/mmcblk0p7 rootfstype=ext4'
boot

# 2. 或修改 DTS chosen.bootargs
```

---

## 八、信号级调试

### 8.1 关键信号测量点

| 信号 | 位置 | 测量方式 | 正常值 |
|------|------|---------|--------|
| 24MHz | XIN24M/XOUT24M | 示波器 10x 探头 | ±100ppm |
| eMMC CLK | eMMC pin 5 | 示波器 1x | 400KHz (BootROM) → 150MHz (HS400) |
| eMMC CMD | eMMC pin 3 | 逻辑分析仪 | 双向通信, 有协议时序 |
| DDR CLK | DDR 颗粒 CK P/N | 示波器 差分探头 | 1332MHz |
| RESET | SoC RESET pin | 万用表 | 上电延迟后为高 |
| USB DP/DM | USB 座子 | 示波器 | 差分 480Mbps 信号 |

### 8.2 DDR 时序调试

```bash
# DDR 时序问题是最难排查的硬件问题之一

# 工具: ddrbin_tool.py (rkbin/tools/)
python3 ddrbin_tool.py -p ddrbin_param.txt rv1126b_ddr_1332MHz_v1.09.bin

# ddrbin_param.txt 关键时序参数:
# ddr_tREXT=0x...     # 刷新时序
# ddr_tWTR=0x...      # 写入到读取延迟
# ddr_tRTW=0x...      # 读取到写入延迟
# ddr_tFAW=0x...      # 四激活窗口
# ddr_tRFC=0x...      # 刷新周期

# DDR 调试策略:
# 1. 降低频率测试 (如 1066MHz → 800MHz)
# 2. 调整时序参数 (放松关键路径)
# 3. 如果所有频率都失败 → 更换 DDR bin
```

---

## 九、思考题

1. **定位最难**: 你拿到一块新板子，所有电压正常、晶振起振、eMMC 有 400KHz 时钟，但串口没有任何输出。你会按什么顺序排查？

2. **DDR 兼容**: 假设你的板子使用 LPDDR4 2GB 颗粒，但 SDK 默认 DDR bin 是 512MB LPDDR4。除了换 bin，还有没有其他方式让至少 SPL/U-Boot 先跑起来？

3. **最小启动**: 如果你怀疑某个驱动卡住了 U-Boot 启动，但不确定是哪一个，如何用最少的步骤定位？

4. **eMMC 损坏**: 如果 `mmc read` 报 I/O error，但 `rkdeveloptool wl` 写入正常，可能是 eMMC 的哪个部分出了问题？

---

## 相关笔记

- [[bsp-boot-flow]] — 正常启动流程 & 实验
- [[bsp-uboot-adaptation]] — 板级适配 Checklist
- [[bsp-uboot-rktools]] — 烧录 & 恢复工具
- [[bsp-uboot-dm-deep]] — DM 驱动模型 (驱动 probe 失败)
- [[MOC-嵌入式Linux]] — 嵌入式 Linux 学习路线 MOC
