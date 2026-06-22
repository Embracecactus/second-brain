---
tags:
  - embedded-linux
  - bsp
  - power-management
  - clock
  - regulator
  - runtime-pm
  - suspend
  - cpufreq
  - rockchip
category: embedded-linux
created: 2026-06-22
updated: 2026-06-22
status: active
soc: Rockchip RV1126B
kernel: Linux 6.1.141
pmu: 0x20838000
pmic: RK801 @ I2C0 0x27
---

# 阶段五：电源管理

> **JD对标**：具备Linux设备模型、中断处理、电源管理等相关知识
>
> 电源管理是 BSP 开发中"能让设备跑起来"和"让设备跑得稳、跑得省电"之间的分水岭。本章覆盖 Clock / Regulator / Runtime PM / PM Domain / Suspend 五大子系统。

---

## 一、电源管理全景

```
┌────────────────────────────────────────────────────┐
│                 Linux 电源管理框架                   │
├────────────────────────────────────────────────────┤
│                                                    │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │ Clock (CCF) │  │ Regulator    │  │ PM Domain │ │
│  │ clk-rv1126b │  │ rk801-reg    │  │ pm_domains│ │
│  │ CRU @0x2e8  │  │ PMIC I2C     │  │ PMU @0x208│ │
│  └──────┬──────┘  └──────┬───────┘  └─────┬─────┘ │
│         │                │                │       │
│         └────────┬───────┴────────────────┘       │
│                  │                                │
│          ┌───────▼───────┐                        │
│          │  Runtime PM   │  pm_runtime_get/put    │
│          │  (per-device) │  自动管理时钟和电源      │
│          └───────┬───────┘                        │
│                  │                                │
│          ┌───────▼───────┐                        │
│          │  CPUFreq      │  cpufreq / OPP         │
│          │  CPU 调频调压  │  408MHz~1.6GHz         │
│          └───────┬───────┘                        │
│                  │                                │
│          ┌───────▼───────┐                        │
│          │  Suspend/Resume│  mem / standby        │
│          │  系统级休眠     │  rv1126b-pm.config    │
│          └───────────────┘                        │
└────────────────────────────────────────────────────┘
```

### 1.1 RV1126B 电源管理硬件

| 硬件 | 基地址/接口 | 驱动文件 | 作用 |
|------|-----------|---------|------|
| CRU (Clock Reset Unit) | 0x20000000 | `clk-rv1126b.c` | 全局时钟树 + 复位控制 |
| PMU (Power Management Unit) | 0x20838000 | `pm_domains.c` | 电源域开关 + 休眠控制 |
| RK801 PMIC | I2C0 @0x27 | `rk801-regulator.c` | 电压调节 (CPU/NPU/DDR/IO) |
| PVT PLL (Core) | 0x20480000 | `clk-pvtpll.c` | CPU 自适应电压频率 |
| PVT PLL (NPU) | 0x22080000 | `clk-pvtpll.c` | NPU 自适应电压频率 |
| TSADC | 0x20bb0000 | `rockchip-tsadc.c` | 温度传感器 (过热保护) |

### 1.2 内核配置状态

当前 sportcam defconfig 的 PM 配置：

| 配置项 | 状态 | 说明 |
|--------|------|------|
| `CONFIG_PM` | y | 电源管理核心 |
| `CONFIG_PM_DEBUG` | y | PM 调试支持 |
| `CONFIG_CPU_IDLE` | y | CPU 空闲节能 |
| `CONFIG_CPU_FREQ` | y | CPU 动态调频 |
| `CONFIG_ROCKCHIP_PM_DOMAINS` | y | PM 域支持 |
| `CONFIG_ROCKCHIP_SYSTEM_MONITOR` | y | 系统监控 |
| `CONFIG_SUSPEND` | **未启用** | 需添加 `rv1126b-pm.config` |

> **注意**：当前 sportcam 配置未启用 Suspend（`CONFIG_SUSPEND` 未设置）。要测试休眠功能，需要在 `RK_KERNEL_CFG_FRAGMENTS` 中添加 `rv1126b-pm.config`。

---

## 二、Clock 框架 (CCF)

### 2.1 Common Clock Framework 架构

```
用户层
  /sys/kernel/debug/clk/clk_summary  — 查看时钟树
  /sys/kernel/debug/clk/clk_dump     — JSON 格式时钟树

内核层 (CCF: Common Clock Framework)
  ├── clk-provider (时钟提供者)
  │   ├── fixed-clock    — 固定频率时钟 (如晶振 24MHz)
  │   ├── pll            — 锁相环 (如 GPLL 1.2GHz)
  │   ├── mux            — 多路选择器 (选时钟源)
  │   ├── divider        — 分频器 (降频)
  │   └── gate           — 时钟门控 (开关)
  │
  └── clk-consumer (时钟使用者)
      └── 驱动通过 clk_get() + clk_prepare_enable() 使用时钟
```

### 2.2 RV1126B 时钟树

```dts
/* rv1126b.dtsi */
cru: clock-reset-controller@20000000 {
    compatible = "rockchip,rv1126b-cru";
    reg = <0x20000000 0x1000>;
    rockchip,grf = <&grf>;
    #clock-cells = <1>;
    #reset-cells = <1>;
};
```

```
24MHz 晶振 (xin24m)
    │
    ├──→ GPLL (1.2GHz)  ──→ 各总线时钟 (ACLK/PCLK/HCLK)
    ├──→ CPLL (1.0GHz)  ──→ 多媒体时钟 (ISP/VEPU/VDPU)
    ├──→ APLL (1.6GHz)  ──→ CPU 时钟 (ARMCLK)
    └──→ DPLL           ──→ DDR 时钟

CPU 时钟路径:
  APLL → mux_armclk → div_aclk_core → ARMCLK (最高 1.6GHz)

I2C5 时钟路径:
  GPLL → mux_i2c5 → div_i2c5 → CLK_I2C5 (通常 100MHz)
                    └→ PCLK_I2C5 (APB 总线时钟)

NPU 时钟路径:
  CPLL → mux_npu → div_npu → ACLK_RKNN + HCLK_RKNN
```

### 2.3 Clock API (驱动中使用)

```c
#include <linux/clk.h>

/* 获取时钟 (从 DTS 的 clocks/clock-names 属性) */
struct clk *clk = devm_clk_get(dev, "pclk");
if (IS_ERR(clk))
    return PTR_ERR(clk);

/* 使能时钟 (prepare + enable) */
clk_prepare_enable(clk);    /* 原子上下文不能用 prepare */

/* 禁用时钟 */
clk_disable_unprepare(clk);

/* 设置时钟频率 */
clk_set_rate(clk, 100000000);  /* 设为 100MHz */

/* 获取当前频率 */
unsigned long rate = clk_get_rate(clk);

/* Runtime PM 自动管理 (推荐) */
/* 如果驱动正确声明了 pm_runtime, 时钟会自动开关 */
```

### 2.4 DTS 中时钟声明

```dts
i2c5: i2c@21140000 {
    compatible = "rockchip,rv1126b-i2c";
    clocks = <&cru CLK_I2C5>, <&cru PCLK_I2C5>;
    clock-names = "i2c", "pclk";
    /* "i2c" = 功能时钟 (传输用), "pclk" = APB 寄存器访问时钟 */
};
```

```c
/* 驱动中获取两个时钟 */
struct clk *clk_i2c = devm_clk_get(dev, "i2c");
struct clk *clk_pclk = devm_clk_get(dev, "pclk");
clk_prepare_enable(clk_i2c);
clk_prepare_enable(clk_pclk);
```

### 2.5 板端验证

```bash
# 查看完整时钟树
cat /sys/kernel/debug/clk/clk_summary
# 输出示例:
#                                 enable  prepare  protect                                duty  hardware
#    clock                          count    count    count        rate   accuracy phase  cycle  enable
# ----------------------------------------------------------------------------------------------------
#  xin24m                               8        8        0    24000000          0     0  50000         Y
#  clk_apll                            1        1        0  1608000000          0     0  50000         Y
#  armclk                               1        1        0  1608000000          0     0  50000         Y
#  clk_gpll                            9        9        0  1200000000          0     0  50000         Y
#  clk_i2c5                             1        1        0    100000000          0     0  50000         Y
#  pclk_i2c5                            1        1        0   100000000          0     0  50000         Y
#  ...

# 查看特定时钟
cat /sys/kernel/debug/clk/clk_summary | grep -E "i2c5|uart0|spi0"

# 查看时钟频率
cat /sys/kernel/debug/clk/armclk/clk_rate
# 预期: 1608000000

# 查看时钟引用计数
cat /sys/kernel/debug/clk/clk_i2c5/clk_enable_count
# 预期: 1 (被 I2C5 驱动启用)
```

---

## 三、Regulator 框架

### 3.1 Regulator 架构

```
用户层
  /sys/class/regulator/regulator.X/
  echo "1200000" > /sys/class/regulator/regulator.3/microvolts

内核层
  ├── regulator-core (核心层)
  │   ├── regulator_enable() / regulator_disable()
  │   ├── regulator_set_voltage()
  │   └── regulator_get_voltage()
  │
  └── regulator 驱动 (PMIC 厂商提供)
      └── rk801-regulator.c
          ├── DCDC1 → vdd_npu (NPU 供电)
          ├── DCDC2 → vcc3v3_sys (系统 3.3V)
          ├── DCDC3 → vcc_ddr (DDR 供电)
          ├── DCDC4 → vdd_logic (逻辑供电)
          ├── LDO1 → vdd_ldo1
          └── LDO2 → vcc_1v8
```

### 3.2 RV1126B PMIC 配置

```dts
/* FET1126B-S.dtsi */
&i2c0 {
    rk801: pmic@27 {
        compatible = "rockchip,rk801";
        reg = <0x27>;
        #interrupt-cells = <2>;
        interrupt-controller;
        rockchip,system-power-controller;  /* 作为系统电源控制器 */

        regulators {
            vdd_npu: DCDC1 {
                regulator-name = "vdd_npu";
                regulator-min-microvolt = <800000>;
                regulator-max-microvolt = <1150000>;
                regulator-ramp-delay = <6000>;
                regulator-always-on;
            };
            vcc3v3_sys: DCDC2 { ... };
            vcc_ddr: DCDC3 { ... };
            vdd_logic: DCDC4 { ... };
            vcc_1v8: LDO2 { ... };
        };
    };
};
```

### 3.3 Regulator API (驱动中使用)

```c
#include <linux/regulator/consumer.h>

/* 获取 regulator (从 DTS 的 xxx-supply 属性) */
struct regulator *supply = devm_regulator_get(dev, "vdd");
if (IS_ERR(supply))
    return PTR_ERR(supply);

/* 使能/禁用 */
regulator_enable(supply);    /* 上电 */
regulator_disable(supply);   /* 断电 */

/* 设置电压 */
regulator_set_voltage(supply, 900000, 1100000);
/* 参数: min_uV=0.9V, max_uV=1.1V */

/* 获取当前电压 */
int voltage = regulator_get_voltage(supply);
```

### 3.4 DTS 中 regulator 引用

```dts
/* 外设节点引用 regulator */
rknpu: rknpu@22080000 {
    compatible = "rockchip,rknpu";
    vdd-supply = <&vdd_npu>;     /* NPU 供电来自 DCDC1 */
    /* 驱动中: devm_regulator_get(dev, "vdd") */
};
```

### 3.5 板端验证

```bash
# 查看所有 regulator
ls /sys/class/regulator/
# 预期: regulator.0 ~ regulator.N

# 查看各 regulator 名称和状态
for r in /sys/class/regulator/regulator.*/; do
    name=$(cat ${r}name 2>/dev/null)
    state=$(cat ${r}state 2>/dev/null)
    uv=$(cat ${r}microvolts 2>/dev/null)
    echo "$name: $state @ ${uv}uV"
done

# 预期输出:
# vdd_npu: enabled @ 900000uV
# vcc3v3_sys: enabled @ 3300000uV
# vcc_ddr: enabled @ 1100000uV
# vdd_logic: enabled @ 900000uV
# vcc_1v8: enabled @ 1800000uV
```

---

## 四、Runtime PM

### 4.1 Runtime PM 概念

Runtime PM 是 per-device 的动态电源管理：设备空闲时自动断电/关时钟，使用时自动恢复。

```
设备使用计数 (usage_count):
  pm_runtime_get_sync(dev)  → usage_count++ → 如果从 0→1, 恢复设备 (resume)
  pm_runtime_put(dev)       → usage_count-- → 如果从 1→0, 休眠设备 (suspend)

状态机:
  ACTIVE (usage_count > 0)
    ↓ pm_runtime_put (usage_count → 0)
  SUSPENDED (时钟关闭, 可选断电)
    ↓ pm_runtime_get_sync (usage_count → 1)
  ACTIVE
```

### 4.2 Runtime PM API

```c
#include <linux/pm_runtime.h>

/* probe 中启用 Runtime PM */
static int my_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;

    /* 使能 runtime pm */
    pm_runtime_enable(dev);

    /* 获取设备 (resume 回调被调用, 时钟打开) */
    pm_runtime_get_sync(dev);

    /* 初始化硬件... */
    my_hw_init(dev);

    /* 释放设备 (autosuspend delay 后 suspend 回调被调用, 时钟关闭) */
    pm_runtime_put(dev);

    return 0;
}

/* 操作前获取, 操作后释放 */
static int my_read_data(struct device *dev)
{
    pm_runtime_get_sync(dev);     /* 开时钟 */

    /* 读写硬件寄存器... */
    u32 val = ioread32(base + DATA_REG);

    pm_runtime_put(dev);          /* 关时钟 (延迟) */
    return val;
}

/* suspend/resume 回调 */
static int my_runtime_suspend(struct device *dev)
{
    /* 关闭时钟 */
    clk_disable_unprepare(my_clk);
    /* 可选: 断电 */
    regulator_disable(my_supply);
    return 0;
}

static int my_runtime_resume(struct device *dev)
{
    /* 恢复供电 */
    regulator_enable(my_supply);
    /* 恢复时钟 */
    clk_prepare_enable(my_clk);
    /* 恢复硬件状态 */
    my_hw_restore(dev);
    return 0;
}

/* 设备驱动中声明回调 */
static const struct dev_pm_ops my_pm_ops = {
    SET_RUNTIME_PM_OPS(my_runtime_suspend, my_runtime_resume, NULL)
};

static struct platform_driver my_driver = {
    .driver = {
        .pm = &my_pm_ops,
    },
};
```

### 4.3 autosuspend (延迟休眠)

```c
/* 设置 autosuspend 延迟 (毫秒) */
pm_runtime_set_autosuspend_delay(dev, 100);  /* 100ms 后休眠 */
pm_runtime_use_autosuspend(dev);             /* 启用 autosuspend 模式 */

/* 用 put_autosuspend 代替 put */
pm_runtime_put_autosuspend(dev);
/* 100ms 后如果没有人 get, 才真正 suspend */
```

> **为什么需要 autosuspend**：频繁开关时钟的开销可能大于持续供电。autosuspend 给一个"冷却期"，避免短时间内反复 suspend/resume。

---

## 五、PM Domain

### 5.1 PM Domain 概念

PM Domain 是一组共享同一电源域的设备的集合。当 domain 内所有设备都 suspend 时，整个 domain 断电。

```
RV1126B PM Domain 拓扑:

  PMU @0x20838000
    ├── VD_NPU domain
    │   └── RV1126B_PD_NPU  → NPU 相关 IP 核
    │       (断电后 NPU 完全不工作)
    │
    ├── VD_VDO domain (always-on)
    │   └── RV1126B_PD_VDO  → VOP / RKVDEC / JPEG / 解码
    │       (rockchip,always-on → 不断电)
    │
    └── VD_AISP domain
        └── RV1126B_PD_AISP → AI 音频 ISP
```

### 5.2 DTS 中的 PM Domain

```dts
/* rv1126b.dtsi */
pmu: power-management@20838000 {
    compatible = "rockchip,rv1126b-pmu", "syscon", "simple-mfd";
    reg = <0x20838000 0x400>;

    power: power-controller {
        compatible = "rockchip,rv1126b-power-controller";
        #power-domain-cells = <1>;
        status = "okay";

        power-domain@RV1126B_PD_NPU {
            reg = <RV1126B_PD_NPU>;
            clocks = <&cru HCLK_RKNN>;
        };
        power-domain@RV1126B_PD_VDO {
            reg = <RV1126B_PD_VDO>;
            clocks = <&cru ACLK_RKVDEC_ROOT>;
            rockchip,always-on;    /* 始终上电 */
        };
        power-domain@RV1126B_PD_AISP {
            reg = <RV1126B_PD_AISP>;
        };
    };
};
```

### 5.3 驱动中关联 PM Domain

```dts
/* 外设节点通过 power-domains 属性关联 */
rknpu: rknpu@22080000 {
    compatible = "rockchip,rknpu";
    power-domains = <&power RV1126B_PD_NPU>;  /* 关联 NPU 电源域 */
};
```

> 当 NPU 驱动调用 `pm_runtime_get_sync()` 时，PM Domain 框架会先确保 `RV1126B_PD_NPU` 域上电，然后才执行驱动的 runtime_resume 回调。

### 5.4 PM Domain 驱动实现

```c
/* drivers/soc/rockchip/pm_domains.c */
static const struct rockchip_domain_info rv1126b_pm_domains[] = {
    [RV1126B_PD_NPU]  = DOMAIN_RV1126B("npu",  BIT(0), BIT(8),  false),
    [RV1126B_PD_VDO]  = DOMAIN_RV1126B("vdo",  BIT(1), BIT(9),  false),
    [RV1126B_PD_AISP] = DOMAIN_RV1126B("aisp", BIT(2), BIT(10), false),
};
/* pwr = PMU 电源控制位, req = 上电请求位 */
```

---

## 六、CPU 调频 (CPUFreq)

### 6.1 OPP 表

```dts
/* rv1126b.dtsi */
cpu_opp_table: cpu0-opp-table {
    compatible = "operating-points-v2";
    opp-shared;

    opp-408000000 {
        opp-hz = /bits/ 64 <408000000>;
        opp-microvolt = <800000 800000 950000>;
    };
    opp-600000000 {
        opp-hz = /bits/ 64 <600000000>;
        opp-microvolt = <825000 825000 950000>;
    };
    /* ... */
    opp-1608000000 {
        opp-hz = /bits/ 64 <1608000000>;
        opp-microvolt = <950000 950000 1150000>;
    };
};

cpus {
    cpu0: cpu@0 {
        operating-points-v2 = <&cpu_opp_table>;
    };
};
```

| 频率 | 电压 | 说明 |
|------|------|------|
| 408 MHz | 0.80V | 最低功耗 |
| 600 MHz | 0.825V | — |
| 816 MHz | 0.875V | — |
| 1008 MHz | 0.900V | — |
| 1200 MHz | 0.900V | — |
| 1416 MHz | 0.925V | — |
| 1608 MHz | 0.950V | 最高性能 |

### 6.2 板端验证

```bash
# 查看 CPU 调频策略
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# 预期: ondemand / performance / powersave

# 查看当前频率
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
# 预期: 408000 ~ 1608000 (kHz)

# 查看可用频率
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies
# 预期: 408000 600000 816000 1008000 1200000 1416000 1608000

# 查看可用策略
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors

# 设置策略
echo performance | sudo tee /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# 跑满 1.6GHz
echo powersave | sudo tee /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# 降到 408MHz

# 手动设置频率
echo 600000 | sudo tee /sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed

# 查看 OPP 表 (含电压)
cat /sys/devices/system/cpu/cpu0/cpufreq/related_available
```

---

## 七、Suspend / Resume

### 7.1 休眠模式

| 模式 | 说明 | RV1126B 支持 |
|------|------|-------------|
| `standby` | 冻结用户空间, CPU 空闲 | ✅ |
| `mem` | 挂起到 RAM (Suspend-to-RAM) | ✅ (需 pm.config) |
| `disk` | 挂起到磁盘 (hibernation) | 通常不支持 |

### 7.2 启用 Suspend

当前 sportcam 配置未启用 Suspend，需添加 `rv1126b-pm.config`：

```bash
# 修改 SDK 配置
# device/rockchip/.chips/rv1126b/rv1126b_sportcam_defconfig
# 将:
#   RK_KERNEL_CFG_FRAGMENTS="rockchip_amp.config  rockchip_debug.config"
# 改为:
#   RK_KERNEL_CFG_FRAGMENTS="rockchip_amp.config  rockchip_debug.config  rv1126b-pm.config"

# rv1126b-pm.config 内容:
#   CONFIG_SUSPEND=y
#   CONFIG_PM_SLEEP=y
#   CONFIG_PM_GENERIC_DOMAINS_SLEEP=y
#   CONFIG_ROCKCHIP_SUSPEND_MODE=y
#   CONFIG_PM_WAKELOCKS=y

# 重新编译内核
./build.sh kernel
./build.sh firmware
```

### 7.3 Suspend 配置 (DTS)

```dts
/* rv1126b.dtsi */
rockchip_suspend: rockchip-suspend {
    compatible = "rockchip,pm-config";
    status = "disabled";    /* 板级 dts 中改为 okay */
    rockchip,sleep-mode-config = <
        (0
        | RKPM_SLP_ARMOFF_PMUOFF    /* 关闭 CPU */
        | RKPM_SLP_PMU_PMUALIVE_32K  /* PMU 以 32K 振荡器运行 */
        | RKPM_SLP_PMU_DIS_OSC       /* 关闭 24M 振荡器 */
        | RKPM_SLP_32K_EXT           /* 使用外部 32K */
        )
    >;
    rockchip,wakeup-config = <
        (0
        | RKPM_GPIO0_WKUP_EN         /* GPIO0 唤醒 */
        )
    >;
};
```

### 7.4 Suspend/Resume 流程

```
用户态: echo mem > /sys/power/state
    ↓
内核态:
  1. freeze_processes()        — 冻结所有用户进程
  2. suspend_devices()         — 逐设备调用 .suspend
     → 关闭外设时钟
     → 保存寄存器状态
  3. suspend_enter()           — 进入休眠
     → 关闭 CPU
     → PMU 进入低功耗模式
     → DDR 进入自刷新
     → 等待唤醒源 (GPIO/RTC/USB)

唤醒 (如按键):
  1. PMU 检测唤醒源
  2. 恢复 CPU 电源
  3. resume_devices()          — 逐设备调用 .resume
     → 恢复外设时钟
     → 恢复寄存器状态
  4. thaw_processes()          — 解冻用户进程
```

---

## 八、实验 1：查看时钟树

### 8.1 实验目标

从 `/sys/kernel/debug/clk/clk_summary` 读取 RV1126B 完整时钟树，画出主要路径。

### 8.2 操作步骤

```bash
# 板端:
# 查看完整时钟树
cat /sys/kernel/debug/clk/clk_summary

# 保存到文件
cat /sys/kernel/debug/clk/clk_summary > /tmp/clk_tree.txt

# 重点观察:
# 1. armclk 频率 (CPU 主频)
# 2. aclk_rkvdec / aclk_rkvenc (编解码时钟)
# 3. clk_i2c0~5 / clk_uart0~7 (外设时钟)
# 4. 各时钟的 enable_count (被引用次数)

# 查看特定时钟详细信息
ls /sys/kernel/debug/clk/ | grep i2c
cat /sys/kernel/debug/clk/clk_i2c5/clk_rate
cat /sys/kernel/debug/clk/clk_i2c5/clk_enable_count
cat /sys/kernel/debug/clk/clk_i2c5/clk_flags
```

### 8.3 预期结果

```
时钟名             频率         引用计数
xin24m             24MHz       8       (晶振)
clk_apll           1.608GHz    1       (CPU PLL)
armclk             1.608GHz    1       (CPU 主频)
clk_gpll           1.2GHz      9       (通用 PLL)
clk_cpll           1.0GHz      ?       (多媒体 PLL)
clk_i2c5           100MHz      1       (I2C5)
pclk_i2c5          100MHz      1       (I2C5 APB)
clk_uart0          150MHz      1       (UART0 调试串口)
aclk_rkvenc        ???MHz      ?       (硬件编码器)
aclk_rkvdec        ???MHz      ?       (硬件解码器)
```

---

## 九、实验 2：查看 Regulator

### 9.1 实验目标

列出所有 voltage regulator，确认 RK801 PMIC 的各路输出电压。

### 9.2 操作步骤

```bash
# 板端:
# 列出所有 regulator
for r in /sys/class/regulator/regulator.*/; do
    name=$(cat ${r}name 2>/dev/null)
    state=$(cat ${r}state 2>/dev/null)
    uv=$(cat ${r}microvolts 2>/dev/null)
    min_uv=$(cat ${r}min_microvolts 2>/dev/null)
    max_uv=$(cat ${r}max_microvolts 2>/dev/null)
    count=$(cat ${r}use_count 2>/dev/null)
    echo "$name: $state ${uv}uV (range: ${min_uv}~${max_uv}) use_count=$count"
done

# 预期输出:
# vdd_npu: enabled 900000uV (800000~1150000) use_count=1
# vcc3v3_sys: enabled 3300000uV use_count=2
# vcc_ddr: enabled 1100000uV use_count=1
# vdd_logic: enabled 900000uV use_count=1
# vcc_1v8: enabled 1800000uV use_count=1
```

---

## 十、实验 3：CPU 调频测试

### 10.1 实验目标

切换 CPU 调频策略，测量不同频率下的性能差异。

### 10.2 操作步骤

```bash
# 1. 查看当前状态
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq

# 2. 性能模式 (固定最高频率)
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
# 预期: 1608000

# 3. 节能模式 (固定最低频率)
echo powersave | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
# 预期: 408000

# 4. 动态模式 (按负载调频)
echo ondemand | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# 5. 性能对比: 运行简单计算
time dd if=/dev/zero bs=1M count=100 | md5sum
# 对比 performance 和 powersave 模式下的耗时
```

---

## 十一、实验 4：Runtime PM 观察

### 11.1 实验目标

观察设备 Runtime PM 状态变化，验证空闲时时钟自动关闭。

### 11.2 操作步骤

```bash
# 1. 查看设备 runtime pm 状态
# 找一个支持 runtime pm 的设备 (如 USB)
ls /sys/devices/platform/*usb*/power/

# 2. 查看 runtime status
cat /sys/devices/platform/*.usb3*/power/runtime_status
# 预期: active / suspended / suspending

# 3. 查看 usage count
cat /sys/devices/platform/*.usb3*/power/usage_count

# 4. 查看 autosuspend delay
cat /sys/devices/platform/*.usb3*/power/autosuspend_delay_ms

# 5. 手动控制
echo auto | sudo tee /sys/devices/platform/*.usb3*/power/control   # 自动管理
echo on | sudo tee /sys/devices/platform/*.usb3*/power/control     # 强制启用
echo disabled | sudo tee /sys/devices/platform/*.usb3*/power/control  # 禁用 rpm

# 6. 观察时钟变化
# USB suspend 前:
cat /sys/kernel/debug/clk/clk_summary | grep usb
# USB suspend 后 (如果 USB 时钟被关闭):
cat /sys/kernel/debug/clk/clk_summary | grep usb
# 预期: enable_count 减少
```

---

## 十二、实验 5：Suspend / Resume 测试 (需启用 pm.config)

### 12.1 实验目标

启用 Suspend 功能，测试挂起到 RAM 和唤醒。

### 12.2 启用 Suspend

```bash
# 1. 修改 SDK 配置 (在 PC 端)
# 编辑 device/rockchip/.chips/rv1126b/rv1126b_sportcam_defconfig
# 在 RK_KERNEL_CFG_FRAGMENTS 中添加 rv1126b-pm.config

# 2. 板级 DTS 中启用 rockchip_suspend
# 编辑 rv1126b-sportcam.dts (或 FET1126B-S.dtsi)
# &rockchip_suspend {
#     status = "okay";
# };

# 3. 重新编译
./build.sh kernel && ./build.sh firmware

# 4. 刷新内核
# dd if=output/firmware/boot.img of=/dev/mmcblk0p3
```

### 12.3 测试

```bash
# 板端:
# 确认 Suspend 支持
cat /sys/power/state
# 预期: freeze mem

# 进入休眠
echo mem > /sys/power/state
# 板子进入休眠, 串口无输出

# 唤醒 (按按键或短接 GPIO0 唤醒引脚)
# 预期: 系统恢复, 串口输出恢复

# 查看休眠日志
dmesg | grep -i suspend
dmesg | grep -i resume

# 查看休眠耗时
dmesg | grep "suspend\|resume" | grep "time"
```

---

## 十三、实验 6：在驱动中使用 Runtime PM

### 13.1 实验目标

在阶段四的 I2C 驱动中添加 Runtime PM 支持，观察时钟自动开关。

### 13.2 修改驱动

```c
/* 在 i2c_demo.c 中添加 Runtime PM */

static int i2c_demo_runtime_suspend(struct device *dev)
{
    struct i2c_demo_data *data = dev_get_drvdata(dev);

    /* 关闭 I2C 功能时钟 (如果可控) */
    /* 注意: I2C 控制器驱动本身已管理时钟, 这里仅做示例 */
    dev_info(dev, "runtime suspend\n");
    return 0;
}

static int i2c_demo_runtime_resume(struct device *dev)
{
    dev_info(dev, "runtime resume\n");
    return 0;
}

static const struct dev_pm_ops i2c_demo_pm_ops = {
    SET_RUNTIME_PM_OPS(i2c_demo_runtime_suspend,
                       i2c_demo_runtime_resume, NULL)
};

static struct i2c_driver i2c_demo_driver = {
    .driver = {
        .name = "i2c-demo",
        .of_match_table = of_match_ptr(i2c_demo_match),
        .pm = &i2c_demo_pm_ops,    /* 添加 PM ops */
    },
};

/* probe 中启用 */
static int i2c_demo_probe(struct i2c_client *client, ...)
{
    ...
    pm_runtime_enable(&client->dev);
    /* 设备初始状态为 active */
    pm_runtime_get_sync(&client->dev);
    ...
    /* autosuspend: 5 秒空闲后自动 suspend */
    pm_runtime_set_autosuspend_delay(&client->dev, 5000);
    pm_runtime_use_autosuspend(&client->dev);
    pm_runtime_put_autosuspend(&client->dev);
    ...
}
```

### 13.3 测试

```bash
# 加载驱动后观察
cat /sys/bus/i2c/devices/2-0048/power/runtime_status
# 预期: active (刚加载)

# 等待 5 秒后
cat /sys/bus/i2c/devices/2-0048/power/runtime_status
# 预期: suspended
# dmesg 预期: "runtime suspend"

# 访问设备 (触发 resume)
cat /sys/bus/i2c/devices/2-0048/reg_read
# dmesg 预期: "runtime resume"
cat /sys/bus/i2c/devices/2-0048/power/runtime_status
# 预期: active
```

---

## 十四、思考题

1. `clk_prepare_enable` 包含 `clk_prepare` 和 `clk_enable` 两步。为什么不能在硬中断上下文中调用 `clk_prepare`？`clk_enable` 可以吗？

2. Regulator 的 `regulator_set_voltage(supply, 900000, 1100000)` 设置的是范围而非精确值。为什么内核不直接设为 900000？PMIC 硬件如何处理这个范围？

3. Runtime PM 的 `pm_runtime_get_sync` 返回后，设备一定处于 active 状态吗？如果 resume 回调返回错误码，会怎样？

4. RV1126B 的 `RV1126B_PD_VDO` 域标记了 `rockchip,always-on`。为什么视频解码域要始终上电？如果取消 always-on，可能出什么问题？

5. Suspend 流程中 `freeze_processes` 会冻结所有用户进程。如果一个进程正在持有文件锁，另一个进程在等待锁，冻结时会发生什么？

---

## 十五、踩坑记录

| 日期 | 问题 | 原因 | 解决方案 |
|------|------|------|----------|
| | clk_get 返回 -ENOENT | DTS 中 clock-names 不匹配 | 确认 name 字符串完全一致 |
| | regulator_get 返回 -EPROBE_DEFER | PMIC 驱动还未 probe | 确保 rk808 驱动先加载 |
| | pm_runtime_get_sync 返回 -EACCES | 未调用 pm_runtime_enable | probe 中先 pm_runtime_enable |
| | Suspend 后无法唤醒 | 唤醒源未在 DTS 中配置 | rockchip,wakeup-config 添加 GPIO |
| | CPU 频率无法改变 | governor 为 userspace | 先切换到 ondemand/performance |
| | clk_summary 中某时钟 enable_count > 预期 | 驱动未正确 disable | 检查 remove/suspend 路径 |

---

## 十六、下阶段预告

阶段六：**Capstone — 完整 BSP 驱动项目**
- 从零设计一个完整 platform driver (I2C 温度传感器)
- 完整生命周期：DTS → probe → IRQ → Runtime PM → sysfs → remove
- 集成前 5 阶段所有知识：设备模型 + 设备树 + 中断 + I2C + PM
- 单元测试 + Ftrace 验证 + 性能基准 + 设计文档

---

## 相关笔记

- [[MOC-嵌入式Linux]] — 学习路线总览
- [[bsp-boot-flow]] — 阶段一：Bootloader + 启动流程
- [[bsp-device-model-dtb]] — 阶段二：设备模型 + 设备树
- [[bsp-interrupt-concurrency]] — 阶段三：中断处理 + 并发
- [[bsp-peripheral-drivers]] — 阶段四：外设驱动 I2C/SPI/UART
- [[kernel-debug-env]] — 附录A：内核调试环境
