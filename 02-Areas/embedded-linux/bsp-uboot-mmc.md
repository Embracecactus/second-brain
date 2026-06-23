---
tags:
  - embedded-linux
  - bsp
  - bootloader
  - u-boot
  - mmc
  - emmc
  - sd
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

# U-Boot MMC 子系统深度解析

> **前置笔记**：[[bsp-boot-flow]] — 分区表 & eMMC 布局
>
> **前置笔记**：[[bsp-uboot-adaptation]] — 板级 MMC IOMUX 配置
>
> **核心驱动**：`drivers/mmc/rockchip_dw_mmc.c` (Rockchip DW MMC)

---

## 一、MMC 子系统架构

```
上层: 块设备层 (UCLASS_BLK)
  mmc_get_blk_desc() → blk_dread() / blk_dwrite()
        │
中层: MMC 核心层 (mmc.c / mmc-uclass.c)
  mmc_init() → mmc_set_clock() → mmc_read() / mmc_write()
        │
下层: 控制器驱动层 (rockchip_dw_mmc.c)
  dwmci_send_cmd() → dwmci_read_data() → dwmci_write_data()
        │
硬件: DesignWare MMC 控制器
  寄存器: CMDARG / CMD / RESP0-3 / DATA / CLKENA / CLKDIV
```

---

## 二、RV1126B DW MMC 控制器

### 2.1 硬件特性

| 特性 | 值 |
|------|-----|
| 控制器 | Synopsys DesignWare Mobile Storage Host Controller |
| DTS node | `dwmmc@20300000` (eMMC), `dwmmc@20310000` (SD) |
| eMMC 接口 | 8-bit, HS400 |
| SD 接口 | 4-bit, UHS-I SDR104 |
| 时钟 | eMMC: 150MHz max (HS400), SD: 200MHz max (SDR104) |
| FIFO | 1024 bytes (internal) + IDMAC |
| 驱动文件 | `drivers/mmc/rockchip_dw_mmc.c` |

### 2.2 驱动层级

```c
// 1. UCLASS_MMC 驱动声明 (rockchip_dw_mmc.c)
U_BOOT_DRIVER(rockchip_rk3288_dw_mshc) = {
    .name   = "rockchip_rk3288_dw_mshc",
    .id     = UCLASS_MMC,
    .of_match = rockchip_dwmmc_ids,
    .bind    = rockchip_dwmmc_bind,      // 创建 UCLASS_BLK 子设备
    .probe   = rockchip_dwmmc_probe,     // 初始化控制器硬件
    .priv_auto_alloc_size = sizeof(struct rockchip_dwmmc_priv),
    .platdata_auto_alloc_size = sizeof(struct rockchip_mmc_plat),
};

// 2. 核心与控制器之间的 ops 桥接
static const struct dm_mmc_ops rockchip_dwmmc_ops = {
    .send_cmd   = rockchip_dwmmc_send_cmd,   // 发命令
    .set_ios    = rockchip_dwmmc_set_ios,    // 设时钟/总线宽度
    .get_cd     = rockchip_dwmmc_get_cd,     // 检测卡是否插入
    .get_wp     = rockchip_dwmmc_get_wp,     // 检测写保护
};

// 3. ops → 底层 dwmci API
static int rockchip_dwmmc_send_cmd(struct udevice *dev,
    struct mmc_cmd *cmd, struct mmc_data *data)
{
    struct rockchip_dwmmc_priv *priv = dev_get_priv(dev);
    return dwmci_send_cmd(&priv->host, cmd, data);  // ← dw_mmc.c 通用层
}
```

### 2.3 eMMC vs SD 的 DTS 配置

```dts
// rv1126b-sportcam.dts

// eMMC (8-bit, HS400, boot 设备)
&emmc {
    status = "okay";
    bus-width = <8>;                    // 8-bit 数据线
    cap-mmc-highspeed;                  // MMC 高速模式
    mmc-hs200-1_8v;                     // HS200 @1.8V
    mmc-hs400-1_8v;                     // HS400 @1.8V
    non-removable;                      // 内置 eMMC, 不可移除
    no-sdio;                            // 非 SDIO 设备
    no-sd;                              // 非 SD 卡
    pinctrl-names = "default";
    pinctrl-0 = <&emmc_bus8 &emmc_cmd &emmc_clk>;
};

// SD 卡 (4-bit, 可移除)
&sdmmc0 {
    status = "disabled";                // sportcam 默认禁用 SD
    bus-width = <4>;
    cap-mmc-highspeed;
    sd-uhs-sdr104;                      // SDR104 超高速
    cd-gpios = <&gpio4 22 GPIO_ACTIVE_LOW>;  // 卡检测引脚
    pinctrl-0 = <&sdmmc0_clk &sdmmc0_cmd &sdmmc0_bus4>;
};
```

---

## 三、MMC 初始化与识别流程

### 3.1 通信协议 — 命令/响应/数据

```
Host                    Card
  │                      │
  ├─ CMD0 (GO_IDLE) ────→│  重置卡到空闲状态
  │←───── 无响应 ────────┤
  │                      │
  ├─ CMD8 (SEND_IF_COND)─→│  检查电压范围
  │←───── R7 ────────────┤  返回支持的电压
  │                      │
  ├─ CMD55 (APP_CMD) ───→│  切换到应用命令模式
  │←───── R1 ────────────┤
  │                      │
  ├─ ACMD41 (SD_SEND_OP_COND) →│  初始化并请求电压
  │←───── R3 (OCR) ─────┤  返回卡容量和电压
  │  (循环直到卡就绪)     │
  │                      │
  ├─ CMD2 (ALL_SEND_CID)─→│  请求卡识别号 (CID)
  │←───── R2 (CID) ──────┤  16 字节唯一 ID
  │                      │
  ├─ CMD3 (SEND_REL_ADDR)→│  请求相对地址 (RCA)
  │←───── R6 (RCA) ─────┤  返回 16 位地址
  │                      │
  ├─ CMD9 (SEND_CSD) ───→│  请求卡特定数据 (CSD)
  │←───── R2 (CSD) ─────┤  返回容量/时序/特性
  │                      │
  ├─ CMD7 (SELECT_CARD) ─→│  选择卡进入传输状态
  │←───── R1 ────────────┤
  │                      │
  ├─ CMD6 (SWITCH_FUNC) ─→│  切换到高速/HS200/HS400
  │←───── R1 ────────────┤
  │                      │
  ├─ CMD16 (SET_BLOCKLEN)─→│  设置块大小 (512)
  │←───── R1 ────────────┤
  │                      │
  ▼                      ▼
  数据传输状态 (CMD18/CMD25 读写)
```

### 3.2 mmc_init() 初始化流程

```c
// drivers/mmc/mmc.c
int mmc_init(struct mmc *mmc)
{
    // 1. 发送 74 个时钟周期给卡 (上电等待)
    mmc_set_clock(mmc, 400000);          // 初始时钟 400KHz

    // 2. 卡识别
    if (IS_SD(mmc))
        mmc_send_if_cond(mmc);           // CMD8: 检查电压
        mmc_app_cmd(sd_send_op_cond);    // ACMD41: 初始化 SD
    else
        mmc_send_op_cond(mmc);           // CMD1: 初始化 MMC

    // 3. 读取 CID / CSD
    mmc_all_send_cid(mmc);               // CMD2: 卡识别号
    mmc_set_rel_addr(mmc);               // CMD3: 分配 RCA
    mmc_send_csd(mmc);                   // CMD9: 卡特定数据

    // 4. 选择卡
    mmc_select_card(mmc);                // CMD7

    // 5. 读取 EXT_CSD (仅 MMC)
    mmc_read_ext_csd(mmc);               // CMD8: EXT_CSD 512 字节
    //   → 读取 HS400 支持、引导分区大小、擦除分组大小等

    // 6. 设置高速模式
    mmc_switch(mmc, EXT_CSD_HS_TIMING, 1);   // HS200
    mmc_switch(mmc, EXT_CSD_HS_TIMING, 3);   // HS400 (如果支持)

    // 7. 设置最终时钟
    mmc_set_clock(mmc, mmc->tran_speed);      // e.g. 150MHz

    // 8. 设置块长度
    mmc_set_blocklen(mmc, 512);
}
```

### 3.3 速度模式协商

| 模式 | 时钟 (MHz) | 数据线 | 速率 (MB/s) | MMC/SD |
|------|-----------|--------|------------|--------|
| Default | 0.4 | 1 | 0.05 | 初始识别 |
| High Speed (SDR12) | 25 | 4 | 12.5 | SD |
| SDR25 | 50 | 4 | 25 | SD |
| SDR50 | 100 | 4 | 50 | SD |
| SDR104 | 208 | 4 | 104 | SD |
| MMC HS | 52 | 8 | 52 | MMC |
| HS200 | 200 | 4/8 | 200 | MMC |
| **HS400** | **200** | **8 DDR** | **400** | **MMC (eMMC 5.0+)** |

RV1126B eMMC 最终运行在 **HS400 @150MHz** → **~300 MB/s** (实际受 eMMC 颗粒限制)

---

## 四、U-Boot MMC 命令

```bash
# === 基础信息 ===
=> mmc info                  # 查看 eMMC 信息
# 输出:
# Device: dwmmc@20300000
# Manufacturer ID: 11
# OEM: 100
# Name: AJNB4R
# Bus Speed: 150000000       # 150MHz
# Mode: HS400                # HS400 模式
# Rd Block Len: 512
# MMC version 5.1
# High Capacity: Yes
# Capacity: 7.3 GB           # 7.3 GiB
# Bus Width: 8-bit
# Erase Group Size: 512 KB
# Boot Partition Size: 4 MB

=> mmc dev 0                 # 切换到 eMMC 设备 0
=> mmc dev 1                 # 切换到 SD 卡 (如有)

# === 分区操作 ===
=> mmc part                  # 打印分区表
# 输出: GPT 分区列表

=> mmc dev 0                 # 确保在 eMMC
=> mmc list                  # 列出所有 MMC 设备
# dwmmc@20300000 (eMMC)
# dwmmc@20310000 (SD)

# === 读写操作 ===
=> mmc read 0x40600000 0x8000 0x20000   # 从偏移 0x8000 读 0x20000 扇区到内存
# 0x8000 = boot 分区起始扇区
# 0x20000 * 512 = 64MB (整个 boot 分区)
# 用于验证 boot.img 是否正确烧录

=> mmc write 0x40600000 0x8000 0x20000  # 从内存写到 eMMC

# === 速度测试 ===
=> mmc dev 0
=> mmc read 0x40600000 0x0 0x8000      # 读 16MB 数据
# 计时: "time: X.XXXms" → 计算读取速度

# === Boot 分区操作 ===
=> mmc partconf 0 1 1 1                 # 设置: 设备0, boot确认=1, boot分区1=RW, boot分区2=RO
=> mmc bootbus 0 2 0 0                  # 设置 boot 总线: 宽度x2, 复位=0, 回退=0
=> mmc read 0x40600000 0x0 0x2000       # 从 boot 分区 1 读取
```

---

## 五、RPMB 安全分区

### 5.1 RPMB 基础

RPMB (Replay Protected Memory Block) 是 eMMC 内部的**安全分区**：

```
eMMC 内部 (非文件系统可见):
┌──────────────────┐
│ Boot Partition 1  │ 4MB (用于 SPL/U-Boot)
├──────────────────┤
│ Boot Partition 2  │ 4MB (A/B 备份)
├──────────────────┤
│ RPMB              │ 512KB-4MB (安全存储, 防重放攻击)
├──────────────────┤
│ User Data Area    │ 剩余 (GPT 分区可见)
└──────────────────┘
```

### 5.2 RPMB 操作

```c
// drivers/mmc/rpmb.c

// RPMB 写入 (需要认证密钥):
// - 写入数据 + MAC (Message Authentication Code)
// - eMMC 硬件内部验证 MAC → 写入计数器递增
// - 防止重放攻击: 如果攻击者保存旧数据重新发送, MAC 匹配但计数器不匹配 → 拒绝

// RPMB 读取:
// - 请求读取 + nonce (随机数)
// - eMMC 返回数据 + MAC + 计数器
// - 验证 MAC 和计数器确认为最新数据

// U-Boot 中:
mmc_rpmb_set_key(mmc, key, 32);         // 设置 256-bit 认证密钥 (一次性)
mmc_rpmb_write(mmc, addr, data, key);   // 写入 RPMB
mmc_rpmb_read(mmc, addr, buf, key);     // 读取 RPMB
```

### 5.3 RPMB 在产品中的用途

| 用途 | 存储内容 | 为何用 RPMB |
|------|---------|------------|
| 防回滚版本号 | rollback-index | 防止篡改降级 |
| 设备密钥 | 私钥/证书 | 不可读, 只能通过 OP-TEE 访问 |
| DRM 密钥 | 内容解密密钥 | 硬件保护的密钥存储 |
| 启动計數 | bootcount | 防止计数篡改 |

---

## 六、MMC 驱动调试

### 6.1 调试命令

```bash
# 检查 eMMC 识别是否成功
=> mmc info
# 如果失败: "Card did not respond to voltage select!"

# 调试卡识别过程 (需要 CONFIG_MMC_DEBUG=y)
# 在 defconfig 中添加:
# CONFIG_MMC_DEBUG=y
# 重新编译 → 输出卡识别每一步的 CMD/RESP 日志

# 时钟问题检查
=> mmc dev 0
=> mmc info | grep "Bus Speed"
# 如果时钟远低于预期, 检查 DTS 中 clock-frequency 或 PLL 配置

# IO 信号完整性
# 通过 timing con 寄存器调节相位:
=> md 0x20300000 0x130 1   # 读取 SDMMC_TIMING_CON0
# 寄存器值反映当前延迟校准状态
```

### 6.2 常见问题

| 现象 | 原因 | 排查 |
|------|------|------|
| `Card did not respond to voltage` | eMMC 未上电/焊接问题 | 检查 VCCQ 1.8V, VCC 3.3V |
| `mmc_init: -110, time out` | 时钟问题/卡初始化超时 | `mmc info` 确认卡存在 |
| `** Unrecognized ext_csd **` | eMMC 版本太老 | 检查 EXT_CSD 版本字段 |
| 读速度只有 20MB/s | 只有 4-bit 模式 | 检查 `bus-width=<8>` DTS |
| `dw_mmc: FIFO underflow` | 时钟相位不对 | 调整 timing con 寄存器 |

---

## 七、思考题

1. **BootROM 如何读取 eMMC？** BootROM 不支持 MMC 协议栈，它直接从 eMMC 的**boot 分区**读取原始数据。U-Boot 完整初始化后如何使用 MMC 协议访问**用户数据分区**？

2. **HS400 vs HS200**：HS400 的双倍数据速率 (DDR) 模式对 PCB 走线有什么要求？为什么有些 RV1126B 板子只能用 HS200？

3. **RPMB 密钥管理**：RPMB 的认证密钥在产线是如何设置的？如果密钥丢失，eMMC 能否恢复？

4. **MMC 与文件系统**：U-Boot 读取 ext4 分区中的内核镜像，需要经过 mmc 驱动 → 块设备层 → 分区层 → 文件系统层。这个调用链的具体路径是什么？

---

## 相关笔记

- [[bsp-boot-flow]] — 分区表, eMMC 布局
- [[bsp-uboot-adaptation]] — IOMUX, eMMC 驱动配置
- [[bsp-uboot-boottime]] — eMMC 读取速度, 压缩算法权衡
- [[bsp-uboot-dm-deep]] — DM 模型 (UCLASS_MMC, UCLASS_BLK)
- [[MOC-嵌入式Linux]] — 嵌入式 Linux 学习路线 MOC
