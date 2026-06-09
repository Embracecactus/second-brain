---
tags:
  - h3
  - dtb
  - orangepi
  - allwinner
  - devicetree
  - arm
category: embedded-linux/allwinner
created: 2026-06-09
board: Xunlong Orange Pi Plus / Plus 2
soc: Allwinner H3 (sun8i-h3)
dtb_file: sun8i-h3-orangepi-plus.dtb
---

# Orange Pi Plus / Plus 2 -- H3 DTB 设备树分析

## 概述

本文档基于 `sun8i-h3-orangepi-plus.dtb` 反编译得到的 Device Tree Source (DTS) 文件，对迅龙 (Xunlong) Orange Pi Plus / Plus 2 开发板的硬件配置进行全面分析。该板搭载 Allwinner H3 (sun8i-h3) SoC，四核 ARM Cortex-A7 架构，内置 Mali-400 GPU，支持 HDMI 输出、千兆以太网、WiFi (SDIO)、USB、红外遥控等丰富外设。

DTS 文件为 `dtc` 反编译产物，phandle 引用为十六进制数值形式，需对照 `__symbols__` 节点理解实际硬件映射关系。

## 关键知识点

- **SoC**: Allwinner sun8i-h3，四核 Cortex-A7 (ARMv7)
- **compatible**: `"xunlong,orangepi-plus\0allwinner,sun8i-h3"`
- **调试串口**: `serial0` 即 UART0 (`serial@1c28000`)，115200n8，PA4/PA5 引脚
- **CPU 频率调节 (DVFS)**: 支持 9 档 OPP，480MHz ~ 1368MHz
- **CPU 供电**: 通过 I2C 总线 (r_i2c, `i2c@1f02400`) 上的 SY8106A PMIC 芯片提供 vdd-cpux，电压范围 1.0V ~ 1.36V
- **存储接口**:
  - MMC0 (`1c0f000`): SD 卡槽，4-bit bus，带卡检测 GPIO
  - MMC1 (`1c10000`): SDIO WiFi 模块 (RTL8189)，4-bit bus，non-removable
  - MMC2 (`1c11000`): eMMC，8-bit bus，non-removable，支持硬件复位
- **网络**: 千兆以太网 (EMAC, RGMII 模式)，通过 MDIO MUX 同时支持内部 PHY 和外部 RGMII PHY
- **显示**: HDMI 1.4 输出 (DesignWare HDMI TX)，DE2 display engine，支持 TCON-TV
- **GPU**: ARM Mali-400 MP1/MP2 (`gpu@1c40000`)，核心时钟 384MHz
- **音频**: I2S2 用于 HDMI 音频输出，板载 codec (`codec@1c22c00`) 支持 LINEOUT/MIC1
- **USB**: 4 组 EHCI/OHCI 控制器 (USB1/USB3 启用)，USB0 支持 OTG (当前 disabled)
- **GPIO Keys**: SW2 (KEY_POWER, code 0x101)、SW4 (KEY_MENU, code 0x100)
- **LED**: 红色 STATUS LED (PA15)、绿色 PWR LED (PL10)，PWR LED 默认常亮
- **红外**: CIR 接收器 (`ir@1f02000`)，PL11 引脚
- **温度监控**: THS 传感器，6 级温控阈值，从 75C (warm) 到 105C (critical)

## 技术细节

### CPU OPP 频率/电压表

| 频率 (MHz) | 频率 (Hex) | 电压 (uV) | 电压 (Hex) |
|:-----------:|:----------:|:---------:|:----------:|
| 480 | 0x1c9c3800 | 1,044,000 | 0xfde80 |
| 648 | 0x269fb200 | 1,044,000 | 0xfde80 |
| 816 | 0x30a32c00 | 1,100,000 | 0x10c8e0 |
| 960 | 0x39387000 | 1,200,000 | 0x124f80 |
| 1008 | 0x3c14dc00 | 1,200,000 | 0x124f80 |
| 1104 | 0x41cdb400 | 1,320,000 | 0x142440 |
| 1200 | 0x47868c00 | 1,320,000 | 0x142440 |
| 1296 | 0x4d3f6400 | 1,340,000 | 0x147260 |
| 1368 | 0x518a0600 | 1,400,000 | 0x155cc0 |

clock-latency-ns = 245000 (0x3b9b0)

### 温控阈值 (Thermal Trips)

| 温度节点 | 温度 (mC) | 温度 (C) | 类型 |
|:--------:|:---------:|:--------:|:----:|
| cpu_warm | 75,000 | 75 | passive |
| cpu_hot_pre | 80,000 | 80 | passive |
| cpu_hot | 85,000 | 85 | passive |
| cpu_very_hot_pre | 90,000 | 90 | passive |
| cpu_very_hot | 95,000 | 95 | passive |
| cpu_crit | 105,000 | 105 | critical |

hysteresis = 2,000 mC (2C)。cooling-maps 通过降频实现热管理，最多可限制到最低频率 OPP。

### 电源域 (Regulators)

| 名称 | 电压 (V) | 用途 |
|:----:|:--------:|:----:|
| vdd-cpux | 1.0 ~ 1.36 (SY8106A) | CPU 核心供电 |
| vcc3v3 | 3.3 | 系统 3.3V |
| vcc3v0 | 3.0 | 系统 3.0V |
| vcc5v0 | 5.0 | 系统 5V |
| gmac-3v3 | 3.3 | 以太网 PHY 供电 (PD6 控制) |
| usb1-vbus | 5.0 | USB1 口供电 (PG13 控制) |
| usb3-vbus | 5.0 | USB3 口供电 (PG11 控制) |
| ahci-5v | 5.0 | SATA 供电 (已禁用) |

### 外设状态总览

| 外设 | 状态 | 说明 |
|:----:|:----:|:----:|
| UART0 | okay | 调试串口 |
| UART1-3 | disabled | 可通过 pinctrl 使能 |
| MMC0 (SD) | okay | SD 卡 |
| MMC1 (WiFi) | okay | SDIO RTL8189 |
| MMC2 (eMMC) | okay | 8-bit eMMC |
| EMAC | okay | RGMII 千兆以太网 |
| HDMI | okay | HDMI 输出 |
| I2C0-2 | disabled | 通用 I2C |
| r_i2c | okay | PMIC (SY8106A) |
| SPI0-1 | disabled | SPI 总线 |
| CIR | okay | 红外接收 |
| Codec | okay | 板载音频 |
| I2S0-1 | disabled | -- |
| I2S2 | okay (implicitly) | HDMI 音频 |
| CSI | disabled | 摄像头接口 |
| PWM | disabled | PWM 输出 |
| MGPU (Mali) | -- | 已声明，无 status |
| Video Engine | -- | VE SRAM 已声明 |

### GPIO Pin 分配速查

| 功能 | 引脚 | 备注 |
|:----:|:----:|:----:|
| UART0 TX/RX | PA4, PA5 | 调试串口 |
| I2C0 | PA11, PA12 | -- |
| I2C1 | PA18, PA19 | 与 I2S0 共用 |
| I2C2 | PE12, PE13 | -- |
| SPI0 | PC0-PC3 | -- |
| SPI1 | PA13-PA16 | 与 UART3 共用 |
| CSI | PE0-PE11 | 摄像头并行接口 |
| EMAC RGMII | PD0-PD17 | drive-strength 40mA |
| MMC0 | PF0-PF5 | SD 卡，drive-strength 30mA |
| MMC1 | PG0-PG5 | WiFi，drive-strength 30mA |
| MMC2 | PC5-PC16 | eMMC 8-bit，drive-strength 40mA |
| IR RX | PL11 | 红外遥控接收 |
| Status LED | PF15 (PA15) | 红色 |
| PWR LED | PL10 | 绿色，默认亮 |
| SW2 (Power) | PL4 | KEY_POWER |
| SW4 (Menu) | PL3 | KEY_MENU |

### 显示链路

```
DE2 Mixer0 --> TCON-TV --> DW-HDMI TX --> HDMI Connector (Type A)
```

- DE2 clock: `allwinner,sun8i-h3-de2-clk` @ 0x1000000
- Mixer0: `allwinner,sun8i-h3-de2-mixer-0` @ 0x1100000
- TCON-TV: `allwinner,sun8i-h3-tcon-tv` @ 0x1c0c000
- HDMI TX: `allwinner,sun8i-h3-dw-hdmi` @ 0x1ee0000
- HDMI PHY: `allwinner,sun8i-h3-hdmi-phy` @ 0x1ef0000
- Framebuffer: HDMI (disabled) / TVE (disabled)，由 bootloader 初始化

### 以太网 MDIO MUX 架构

```
EMAC (@1c30000)
  └── MDIO Bus
       └── MDIO MUX (sun8i-h3-mdio-mux)
            ├── mdio@1 (Internal)
            │    └── Internal PHY (int_mii_phy, reg=1)
            │        使用 bus clock + reset
            └── mdio@2 (External, RGMII)
                 └── ext_rgmii_phy (reg=0)  <-- 实际使用的 PHY
```

当前 phy-handle 指向外部 RGMII PHY (`ext_rgmii_phy`)，phy-mode = "rgmii"。

### USB 控制器映射

| 控制器 | 地址 | 类型 | 状态 | PHY |
|:------:|:----:|:----:|:----:|:---:|
| USB0 (OTG) | 1c19000 | MUSB | disabled | phy0 |
| USB1 EHCI | 1c1b000 | EHCI | **okay** | phy1 |
| USB1 OHCI | 1c1b400 | OHCI | disabled | phy1 |
| USB2 EHCI | 1c1c000 | EHCI | disabled | phy2 |
| USB2 OHCI | 1c1c400 | OHCI | disabled | phy2 |
| USB3 EHCI | 1c1d000 | EHCI | **okay** | phy3 |
| USB3 OHCI | 1c1d400 | OHCI | disabled | phy3 |

注意: 实际 USB 可用端口由 EHCI 驱动加载决定，OHCI 通常会作为 companion controller 自动关联。

## 代码/配置片段

### DTS 基本信息

```dts
/ {
    model = "Xunlong Orange Pi Plus / Plus 2";
    compatible = "xunlong,orangepi-plus", "allwinner,sun8i-h3";

    chosen {
        stdout-path = "serial0:115200n8";
    };
};
```

### CPU 节点 (四核 Cortex-A7)

```dts
cpus {
    #address-cells = <0x01>;
    #size-cells = <0x00>;

    cpu@0 {
        compatible = "arm,cortex-a7";
        device_type = "cpu";
        reg = <0x00>;
        clocks = <0x03 0x0e>;       /* ccu, CLK_CPUX */
        clock-names = "cpu";
        operating-points-v2 = <0x2f>;  /* opp_table0 */
        #cooling-cells = <0x02>;
        cpu-supply = <0x30>;        /* vdd-cpux (SY8106A) */
    };
    /* cpu@1, cpu@2, cpu@3 同结构，无独立 cpu-supply */
};
```

### SY8106A PMIC 配置 (I2C 地址 0x65)

```dts
i2c@1f02400 {    /* r_i2c, PL0/PL1, s_i2c */
    compatible = "allwinner,sun6i-a31-i2c";
    status = "okay";

    regulator@65 {
        compatible = "silergy,sy8106a";
        reg = <0x65>;
        regulator-name = "vdd-cpux";
        silergy,fixed-microvolt = <0x124f80>;  /* 1.2V default */
        regulator-min-microvolt = <0xf4240>;    /* 1.0V */
        regulator-max-microvolt = <0x155cc0>;   /* 1.36V */
        regulator-ramp-delay = <0xc8>;          /* 200 uV/us */
        regulator-boot-on;
        regulator-always-on;
    };
};
```

### 千兆以太网 (EMAC + RGMII)

```dts
ethernet@1c30000 {
    compatible = "allwinner,sun8i-h3-emac";
    reg = <0x1c30000 0x10000>;
    interrupts = <0x00 0x52 0x04>;
    phy-mode = "rgmii";
    phy-handle = <0x16>;           /* ext_rgmii_phy */
    phy-supply = <0x18>;           /* gmac-3v3, PD6 控制 */
    pinctrl-0 = <0x17>;            /* emac-rgmii-pins, drive 40mA */
    allwinner,leds-active-low;
    status = "okay";
};
```

### eMMC 配置 (8-bit 总线)

```dts
mmc@1c11000 {
    compatible = "allwinner,sun7i-a20-mmc";
    reg = <0x1c11000 0x1000>;
    bus-width = <0x08>;
    non-removable;
    cap-mmc-hw-reset;
    vmmc-supply = <0x0b>;          /* vcc3v3 */
    pinctrl-0 = <0x0f>;            /* mmc2-8bit-pins, drive 40mA */
    status = "okay";
};
```

### HDMI 音频链路

```dts
sound {
    compatible = "simple-audio-card";
    simple-audio-card,format = "i2s";
    simple-audio-card,name = "allwinner-hdmi";
    simple-audio-card,mclk-fs = <0x100>;   /* 256 fs */

    simple-audio-card,codec {
        sound-dai = <0x04>;        /* hdmi@1ee0000 */
    };

    simple-audio-card,cpu {
        sound-dai = <0x05>;        /* i2s@1c22800 (I2S2) */
        dai-tdm-slot-num = <0x02>;
        dai-tdm-slot-width = <0x20>;   /* 32-bit */
    };
};
```

### WiFi 电源序列

```dts
wifi_pwrseq {
    compatible = "mmc-pwrseq-simple";
    reset-gpios = <0x3c 0x00 0x07 0x01>;  /* r_pio, PL7, active-low */
};

mmc@1c10000 {
    mmc-pwrseq = <0x0e>;          /* wifi_pwrseq */
    non-removable;
    bus-width = <0x04>;

    sdio_wifi@1 {
        reg = <0x01>;             /* SDIO function 1 */
    };
};
```

### 板载 Codec 音频路由

```dts
codec@1c22c00 {
    compatible = "allwinner,sun8i-h3-codec";
    status = "okay";
    allwinner,pa-gpios = <0x0c 0x00 0x10 0x00>;  /* pio, PA16, 功放使能 */
    allwinner,audio-routing =
        "Speaker", "LINEOUT",
        "MIC1", "Mic",
        "Mic", "MBIAS";
};
```

### 编译 DTB / 反编译 DTS 命令参考

```bash
# DTS 编译为 DTB
dtc -I dts -O dtb -o sun8i-h3-orangepi-plus.dtb sun8i-h3-orangepi-plus.dts

# DTB 反编译为 DTS
dtc -I dtb -O dts -o sun8i-h3-orangepi-plus.dts sun8i-h3-orangepi-plus.dtb

# 查看 DTB 中的字符串表
strings sun8i-h3-orangepi-plus.dtb | head -50
```

## 相关链接

- Allwinner H3 Mainline Linux DTS 上游: `arch/arm/boot/dts/allwinner/sun8i-h3-orangepi-plus.dts`
- Orange Pi 官方镜像与文档: http://www.orangepi.org/
- Linux Kernel DT binding 文档: https://www.kernel.org/doc/Documentation/devicetree/bindings/
- Allwinner sunxi 社区 Wiki: https://linux-sunxi.org/
- SY8106A Datasheet: Silergy Corp. (用于 vdd-cpux 动态调压)
- DesignWare HDMI TX Binding: `Documentation/devicetree/bindings/display/bridge/dw_hdmi.yaml`
- Mali-400 DT Binding: `Documentation/devicetree/bindings/gpu/arm,mali-400.yaml`
