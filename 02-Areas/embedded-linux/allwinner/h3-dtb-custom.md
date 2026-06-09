---
tags:
  - h3
  - dtb
  - custom
  - allwinner
  - orangepi
  - device-tree
  - arm
  - embedded-linux
category: embedded-linux/allwinner
created: 2026-06-09
board: Orange Pi PC / PC Plus
SoC: Allwinner H3 (sun8i-h3)
arch: ARM Cortex-A7 quad-core
---

# Orange Pi H3 设备树 (DTB/DTS) 完整分析

## 项目概述

本项目包含 **Xunlong Orange Pi PC** 和 **Orange Pi PC Plus** 两款开发板的 Device Tree Source (DTS) 文件及其编译产物 (DTB)。两款开发板均基于 **Allwinner H3 (sun8i-h3)** SoC，采用四核 ARM Cortex-A7 架构。DTS 文件是 Linux 内核用于硬件描述的关键配置，定义了从 CPU 频率策略到 GPIO 引脚映射的所有硬件信息。

### 文件清单

| 文件名 | 说明 |
|--------|------|
| `sun8i-h3-orangepi-pc.dts` | Orange Pi PC 的完整设备树源码 |
| `sun8i-h3-orangepi-pc.dtb` | Orange Pi PC 编译后的设备树二进制 |
| `sun8i-h3-orangepi-pc-plus.dtb` | Orange Pi PC Plus 编译后的设备树二进制 |
| `sun8i-h3-orangepi-plus.dts` | Orange Pi PC Plus 的完整设备树源码 |
| `sun8i-h3-orangepi-pc - 副本.txt` | PC 版 DTS 的文本备份（内容与 .dts 一致） |

---

## 关键知识点

### 1. DTS 与 DTB 的关系

- **DTS (Device Tree Source)**: 人类可读的设备树源文件，文本格式
- **DTB (Device Tree Blob)**: 编译后的二进制格式，由 bootloader 加载并传递给内核
- 编译命令: `dtc -I dts -O dtb -o output.dtb input.dts`
- 反编译命令: `dtc -I dtb -O dts -o output.dts input.dtb`

### 2. 两款开发板的核心差异

| 特性 | Orange Pi PC | Orange Pi PC Plus |
|------|-------------|-------------------|
| `compatible` 字符串 | `xunlong,orangepi-pc` | `xunlong,orangepi-pc-plus` |
| 板载 eMMC (MMC2) | disabled | **enabled** (8-bit, non-removable, cap-mmc-hw-reset) |
| SDIO WiFi (MMC1) | disabled | **enabled** (RTL8189FTV, non-removable) |
| WiFi Power Sequence | 无 | `wifi_pwrseq` (mmc-pwrseq-simple) |
| OV5640 摄像头 | **已配置** (i2c2 + CSI) | 未配置 (i2c2 disabled) |
| DE2 clock reg 大小 | `0x10000` | `0x100000` |
| `aliases` | ethernet0, serial0 | ethernet0, serial0, **ethernet1** (SDIO WiFi) |

### 3. CPU 与 DVFS (动态电压频率调节)

两款板子共享相同的 OPP (Operating Performance Points) 表:

| 频率 | 电压 (uV) | 频率 (hex) | 电压 (hex) |
|------|-----------|-----------|-----------|
| 480 MHz | 1,044,000 (1.044V) | `0x1c9c3800` | `0xfde80` |
| 648 MHz | 1,044,000 | `0x269fb200` | `0xfde80` |
| 816 MHz | 1,104,000 (1.104V) | `0x30a32c00` | `0x10c8e0` |
| 960 MHz | 1,200,000 (1.2V) | `0x39387000` | `0x124f80` |
| 1008 MHz | 1,200,000 | `0x3c14dc00` | `0x124f80` |
| 1104 MHz | 1,320,000 (1.32V) | `0x41cdb400` | `0x142440` |
| 1200 MHz | 1,320,000 | `0x47868c00` | `0x142440` |
| 1296 MHz | 1,340,000 (1.34V) | `0x4d3f6400` | `0x147260` |
| **1368 MHz** | **1,400,000 (1.4V)** | `0x518a0600` | `0x155cc0` |

- CPU 供电芯片: **Silergy SY8106A** (I2C 地址 0x65, 通过 r_i2c 总线)
- 固定电压: 1,200,000 uV (1.2V)
- 电压范围: 1,000,000 ~ 1,400,000 uV (1.0V ~ 1.4V)
- Ramp delay: 200 us (0xC8)

### 4. 热管理策略

**Orange Pi PC** 有 6 级温度保护:

| Trip Point | 温度 | 滞后 | 类型 |
|-----------|------|------|------|
| cpu_warm | 75,000 mC (75C) | 2,000 mC | passive |
| cpu_hot_pre | 80,000 mC (80C) | 2,000 mC | passive |
| cpu_hot | 85,000 mC (85C) | 2,000 mC | passive |
| cpu_very_hot_pre | 90,000 mC (90C) | 2,000 mC | passive |
| cpu_very_hot | 95,000 mC (95C) | 2,000 mC | passive |
| **cpu_crit** | **105,000 mC (105C)** | **2,000 mC** | **critical** |

**Orange Pi PC Plus** 简化为 2 级:

| Trip Point | 温度 | 类型 |
|-----------|------|------|
| cpu-hot | 80,000 mC (80C) | passive |
| cpu-very-hot | 100,000 mC (100C) | **critical** |

---

## 技术细节

### SoC 外设地址映射

```
0x01000000  DE2 Clock (display engine 2 clock)
0x01100000  DE2 Mixer 0 (display mixer)
0x01C00000  System Control / SRAM
0x01C02000  DMA Controller
0x01C0C000  TCON-TV (LCD controller for HDMI/TVE)
0x01C0E000  Video Engine (硬件视频编解码)
0x01C0F000  MMC0 (SD card)
0x01C10000  MMC1 (SDIO WiFi, PC Plus only)
0x01C11000  MMC2 (eMMC, PC Plus only)
0x01C14000  SID EEPROM (含温度传感器校准数据)
0x01C19000  USB OTG (MUSB controller)
0x01C19400  USB PHY (4端口: pmu0~pmu3)
0x01C1A000  EHCI0 / 0x01C1A400  OHCI0 (USB Host 0)
0x01C1B000  EHCI1 / 0x01C1B400  OHCI1 (USB Host 1)
0x01C1C000  EHCI2 / 0x01C1C400  OHCI2 (USB Host 2)
0x01C1D000  EHCI3 / 0x01C1D400  OHCI3 (USB Host 3)
0x01C20000  CCU (Clock Control Unit)
0x01C20800  PIO (Pin Controller / GPIO)
0x01C20C00  Timer
0x01C20CA0  Watchdog
0x01C21000  SPDIF
0x01C21400  PWM
0x01C22000  I2S0 / 0x01C22400  I2S1 / 0x01C22800  I2S2
0x01C22C00  Audio Codec (模拟音频)
0x01C25000  THS (Thermal Sensor)
0x01C28000  UART0 / 0x01C28400  UART1 / 0x01C28800  UART2 / 0x01C28C00  UART3
0x01C2AC00  I2C0 / 0x01C2B000  I2C1 / 0x01C2B400  I2C2
0x01C30000  EMAC (Ethernet MAC, 内置 PHY)
0x01C40000  Mali-400 GPU
0x01C68000  SPI0 / 0x01C69000  SPI1
0x01C81000  GIC-400 (中断控制器)
0x01CB0000  CSI (Camera Serial Interface)
0x01EE0000  HDMI Controller (DesignWare)
0x01EF0000  HDMI PHY
0x01F00000  RTC
0x01F01400  R_CCU (R_域时钟控制)
0x01F015C0  Codec Analog (R_域)
0x01F02000  IR Receiver (红外接收)
0x01F02400  R_I2C (R_域 I2C, 连接 SY8106A)
0x01F02C00  R_PIO (R_域 GPIO)
0x01D00000  SRAM C (512KB, Video Engine 使用)
```

### UART 串口配置

| UART | 基地址 | 引脚 (TX/RX) | 默认状态 |
|------|--------|-------------|---------|
| UART0 | 0x01C28000 | PA4/PA5 | **okay** (debug console) |
| UART1 | 0x01C28400 | PG6/PG7 | disabled |
| UART2 | 0x01C28800 | PA0/PA1 | disabled |
| UART3 | 0x01C28C00 | PA13/PA14 | disabled |

- 所有 UART 基于 DesignWare APB UART IP (`snps,dw-apb-uart`)
- `reg-shift = <0x02>`: 寄存器地址偏移 4 字节 (32-bit 寄存器)
- `reg-io-width = <0x04>`: 32-bit I/O 宽度
- Debug console 输出: `stdout-path = "serial0:115200n8"` (115200 baud, 8N1)

### USB 子系统

H3 的 USB 架构包含:
- **USB OTG (MUSB)**: 0x01C19000, dr_mode = "otg", 使用 PHY port 0
- **USB PHY**: 4 端口控制器 (usb0~usb3), 含 PMU 管理
- **EHCI/OHCI Host**: 4 组 EHCI+OHCI 对 (Host 0~3)
  - Host 0: USB Type A 口 (靠近以太网口)
  - Host 1~3: 其余 USB 口

VBUS 供电 (5V, regulator-fixed):
- `usb0-vbus`: GPIO PL2, status = okay
- `usb1-vbus`: GPIO PG6, status = disabled (PC 默认)
- `usb2-vbus`: GPIO PG3, status = disabled (PC 默认)

### 网络

**EMAC (有线以太网)**:
- 控制器: `allwinner,sun8i-h3-emac` (基于 Synopsys DWC Ethernet MAC)
- PHY 模式: `mii` (内置 100M PHY)
- 使用 MDIO MUX 架构: internal_mdio (地址 1) + external_mdio (地址 2)
- 内置 PHY: `ethernet-phy-ieee802.3-c22`, 地址 1
- LED 配置: `allwinner,leds-active-low`

**SDIO WiFi (仅 PC Plus)**:
- 芯片: **RTL8189FTV** (Realtek)
- 连接: MMC1 总线, SDIO 地址 1
- 电源序列: `mmc-pwrseq-simple`, reset GPIO PL7 (active-low)
- `non-removable` 属性, `bus-width = <0x04>`

### GPIO 引脚分配汇总

#### PA 端口 (Port A)
| 引脚 | 功能 | 备注 |
|------|------|------|
| PA0/PA1 | UART2 TX/RX | |
| PA2/PA3 | UART2 RTS/CTS | |
| PA4/PA5 | UART0 TX/RX | **Debug Console** |
| PA11/PA12 | I2C0 SDA/SCL | |
| PA13/PA14 | SPI1 CS0/CLK / UART3 TX/RX | 复用 |
| PA15/PA16 | SPI1 MOSI/MISO / UART3 RTS/CTS | 复用 |
| PA17 | SPDIF TX | |
| PA18/PA19 | I2C1 SDA/SCL / I2S0 SCK/WS | 复用 |
| PA20/PA21 | I2S0 SD/SDI | |

#### PC 端口 (Port C)
| 引脚 | 功能 | 备注 |
|------|------|------|
| PC0~PC3 | SPI0 CLK/MOSI/MISO/CS | |
| PC5~PC16 | MMC2 8-bit | **仅 PC Plus (eMMC)** |

#### PD 端口 (Port D)
| 引脚 | 功能 | 备注 |
|------|------|------|
| PD0~PD17 | EMAC RGMII | 有线以太网 (drive-strength 40mA) |

#### PE 端口 (Port E)
| 引脚 | 功能 | 备注 |
|------|------|------|
| PE0~PE11 | CSI (摄像头) | 仅 PC 默认配置 |
| PE1 | CSI MCLK | 摄像头主时钟 |
| PE12/PE13 | I2C2 SDA/SCL | (bias-pull-up) |

#### PF 端口 (Port F)
| 引脚 | 功能 | 备注 |
|------|------|------|
| PF0~PF5 | MMC0 | SD 卡 (drive-strength 30mA, bias-pull-up) |

#### PG 端口 (Port G)
| 引脚 | 功能 | 备注 |
|------|------|------|
| PG0~PG5 | MMC1 | SDIO WiFi (仅 PC Plus) |
| PG6/PG7 | UART1 TX/RX | |
| PG8/PG9 | UART1 RTS/CTS | |
| PG10~PG13 | I2S1 | |

#### PL 端口 (R_域, Port L)
| 引脚 | 功能 | 备注 |
|------|------|------|
| PL0/PL1 | R_I2C SDA/SCL | 连接 SY8106A DCDC |
| PL2 | USB0 VBUS 控制 GPIO | r_pio 控制 |
| PL3 | SW4 按键 GPIO | GPIO-KEY (linux,code = 0x74 = KEY_POWER) |
| PL7 | WiFi reset GPIO | 仅 PC Plus |
| PL10 | PWR LED (绿色) | 默认亮 |
| PL11 | IR 接收 | s_cir_rx 功能 |

### 显示子系统

**显示管线**: DE2 Mixer0 -> TCON-TV -> HDMI -> Connector

- **DE2 Clock**: `allwinner,sun8i-h3-de2-clk`
- **DE2 Mixer 0**: `allwinner,sun8i-h3-de2-mixer-0` (Display Engine 2)
- **TCON-TV**: `allwinner,sun8i-h3-tcon-tv` (LCD controller, HDMI 输出通道)
- **HDMI Controller**: `allwinner,sun8i-h3-dw-hdmi` (DesignWare HDMI IP)
- **HDMI PHY**: `allwinner,sun8i-h3-hdmi-phy` (自研 HDMI PHY)
- **Connector**: HDMI Type A
- **Simple Framebuffer**: HDMI 管线 (`mixer0-lcd0-hdmi`) 默认 disabled, 由 U-Boot 传递

**GPU**: ARM Mali-400 MP2 (GP + 2xPP), 核心时钟 384 MHz (`0x16e36000`)

### 音频子系统

- **HDMI Audio**: `simple-audio-card`, I2S 格式, 256 MCLK/FS ratio
- **Analog Codec**: `allwinner,sun8i-h3-codec` (Line Out + MIC1)
- **I2S2**: 8 通道播放 (用于 HDMI 数字音频)
- Audio routing: `Line Out -> LINEOUT`, `MIC1 -> Mic -> MBIAS`

### 摄像头 (仅 Orange Pi PC)

- **Sensor**: OmniVision OV5640 (5MP, I2C 地址 0x3C)
- **CSI 控制器**: `allwinner,sun8i-h3-csi`
- **总线宽度**: 8-bit, data-shift = 2
- **同步极性**: hsync-active = high, vsync-active = low, pclk-sample = rising
- **电源**:
  - AVDD: 2.8V (`cam-avdd`, 来自 vcc3v3)
  - DOVDD: 1.8V (`cam-dovdd`, 来自 vcc3v3)
  - DVDD: 1.5V (`cam-dvdd`, 来自 vcc3v3)

### 电源域 (Regulators)

| 名称 | 电压 | 类型 | 用途 |
|------|------|------|------|
| vdd-cpux | 1.2V (可调 1.0~1.4V) | SY8106A (I2C DCDC) | CPU 核心供电 |
| vcc3v0 | 3.0V | fixed | 板级 3.0V |
| vcc3v3 | 3.3V | fixed | 板级 3.3V |
| vcc5v0 | 5.0V | fixed | 板级 5.0V |
| usb0-vbus | 5.0V | fixed (GPIO 控制) | USB OTG VBUS |
| usb1-vbus | 5.0V | fixed (GPIO 控制) | USB Host 1 VBUS |
| usb2-vbus | 5.0V | fixed (GPIO 控制) | USB Host 2 VBUS |
| ahci-5v | 5.0V | fixed (GPIO 控制) | SATA (disabled) |

### 特殊硬件

- **IR 接收器**: `allwinner,sun6i-a31-ir`, 引脚 PL11, status = okay
- **Watchdog**: `allwinner,sun6i-a31-wdt`
- **Video Engine**: 硬件视频编解码, 使用 SRAM C (512KB)
- **SID**: Security ID EEPROM, 含 thermal sensor 校准数据 (offset 0x34)

---

## 代码/配置片段

### DTS 基础结构

```dts
/dts-v1/;

/ {
    interrupt-parent = <0x01>;  /* GIC-400 */
    #address-cells = <0x01>;
    #size-cells = <0x01>;
    model = "Xunlong Orange Pi PC";
    compatible = "xunlong,orangepi-pc\0allwinner,sun8i-h3";
    /* ... */
};
```

### Debug Console 配置 (UART0)

```dts
/* chosen 节点中的串口输出 */
chosen {
    stdout-path = "serial0:115200n8";
};

/* UART0 硬件节点 */
serial@1c28000 {
    compatible = "snps,dw-apb-uart";
    reg = <0x1c28000 0x400>;
    interrupts = <0x00 0x00 0x04>;
    reg-shift = <0x02>;
    reg-io-width = <0x04>;
    clocks = <0x03 0x3e>;
    resets = <0x03 0x31>;
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <0x19>;  /* uart0-pa-pins: PA4/PA5 */
};

/* UART0 引脚定义 */
uart0-pa-pins {
    pins = "PA4\0PA5";
    function = "uart0";
};
```

### CPU OPP 表 (DVFS)

```dts
opp_table0 {
    compatible = "operating-points-v2";
    opp-shared;

    opp-480000000 {
        opp-hz = <0x00 0x1c9c3800>;       /* 480 MHz */
        opp-microvolt = <0xfde80 0xfde80 0x13d620>;  /* min/target/max = 1.044V */
        clock-latency-ns = <0x3b9b0>;      /* 244 us */
    };

    opp-1368000000 {
        opp-hz = <0x00 0x518a0600>;       /* 1368 MHz (最高频) */
        opp-microvolt = <0x155cc0 0x155cc0 0x155cc0>;  /* 1.4V */
        clock-latency-ns = <0x3b9b0>;
    };
};
```

### SY8106A CPU 电压调节器 (R_I2C)

```dts
/* R_域 I2C 总线 (PL0/PL1) */
i2c@1f02400 {
    compatible = "allwinner,sun6i-a31-i2c";
    reg = <0x1f02400 0x400>;
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <0x2d>;  /* r-i2c-pins: PL0/PL1 */

    regulator@65 {
        compatible = "silergy,sy8106a";
        reg = <0x65>;                          /* I2C 地址 0x65 */
        regulator-name = "vdd-cpux";
        silergy,fixed-microvolt = <0x124f80>;  /* 固定 1.2V */
        regulator-min-microvolt = <0xf4240>;   /* 最低 1.0V */
        regulator-max-microvolt = <0x155cc0>;   /* 最高 1.4V */
        regulator-ramp-delay = <0xc8>;         /* 200 us */
        regulator-boot-on;
        regulator-always-on;
    };
};
```

### Orange Pi PC Plus 板载 eMMC 配置

```dts
/* MMC2 - 板载 eMMC */
mmc@1c11000 {
    reg = <0x1c11000 0x1000>;
    compatible = "allwinner,sun7i-a20-mmc";
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <0x0f>;          /* mmc2-8bit-pins */
    bus-width = <0x08>;           /* 8-bit 数据总线 */
    non-removable;                /* 不可移除 */
    cap-mmc-hw-reset;             /* 支持硬件复位 */
    vmmc-supply = <0x0b>;         /* vcc3v3 */
};

/* MMC2 引脚 (8-bit) */
mmc2-8bit-pins {
    pins = "PC5\0PC6\0PC8\0PC9\0PC10\0PC11\0PC12\0PC13\0PC14\0PC15\0PC16";
    function = "mmc2";
    drive-strength = <0x28>;      /* 40mA (PC Plus 略高于 PC 的 30mA) */
    bias-pull-up;
};
```

### Orange Pi PC Plus SDIO WiFi 配置

```dts
/* MMC1 - SDIO WiFi */
mmc@1c10000 {
    compatible = "allwinner,sun7i-a20-mmc";
    status = "okay";
    bus-width = <0x04>;
    non-removable;
    mmc-pwrseq = <0x0e>;         /* 引用 wifi_pwrseq */
    vmmc-supply = <0x0b>;

    sdio_wifi@1 {
        reg = <0x01>;             /* SDIO 地址 1 = RTL8189FTV */
    };
};

/* WiFi 电源序列 */
wifi_pwrseq {
    compatible = "mmc-pwrseq-simple";
    reset-gpios = <0x34 0x00 0x07 0x01>;  /* PL7, active-low */
};
```

### OV5640 摄像头配置 (仅 Orange Pi PC)

```dts
/* I2C2 上的 OV5640 摄像头 */
camera@3c {
    compatible = "ovti,ov5640";
    reg = <0x3c>;                 /* I2C 地址 0x3C */
    pinctrl-names = "default";
    pinctrl-0 = <0x20>;          /* csi-mclk-pin: PE1 */
    clocks = <0x03 0x6b>;
    clock-names = "xclk";
    reset-gpios = <0x0c 0x04 0x0e 0x01>;
    AVDD-supply = <0x21>;        /* 2.8V */
    DOVDD-supply = <0x22>;       /* 1.8V */
    DVDD-supply = <0x23>;        /* 1.5V */

    port {
        endpoint {
            remote-endpoint = <0x24>;   /* 连接到 CSI 控制器 */
            bus-width = <0x08>;
            data-shift = <0x02>;
            hsync-active = <0x01>;
            vsync-active = <0x00>;
            pclk-sample = <0x01>;
        };
    };
};
```

### LED 和按键

```dts
leds {
    compatible = "gpio-leds";
    pwr_led {
        label = "orangepi:green:pwr";
        gpios = <0x3c 0x00 0x0a 0x00>;  /* PL10 */
        default-state = "on";
    };
    status_led {
        label = "orangepi:red:status";
        gpios = <0x0c 0x00 0x0f 0x00>;  /* PA15 */
    };
};

r_gpio_keys {
    compatible = "gpio-keys";
    sw4 {
        label = "sw4";
        linux,code = <0x74>;              /* KEY_POWER */
        gpios = <0x3c 0x00 0x03 0x01>;    /* PL3, active-low */
    };
};
```

### DTB 编译与反编译操作

```bash
# DTS 编译为 DTB
dtc -I dts -O dtb -o sun8i-h3-orangepi-pc.dtb sun8i-h3-orangepi-pc.dts

# DTB 反编译为 DTS (可读)
dtc -I dtb -O dts -o output.dts sun8i-h3-orangepi-pc.dtb

# 验证 DTB 完整性
fdtdump sun8i-h3-orangepi-pc.dtb

# 与主线内核 DTB 对比
dtc -I dtb -O dts /boot/dtb/allwinner/sun8i-h3-orangepi-pc.dtb > upstream.dts
diff upstream.dts custom.dts
```

### 常用 DTB 调试技巧

```bash
# 查看当前运行系统的设备树
ls /proc/device-tree/
cat /proc/device-tree/model
cat /proc/device-tree/compatible

# 查看设备树中所有 status = "okay" 的节点
find /proc/device-tree -name status -exec cat {} \;

# 检查特定外设是否启用
cat /proc/device-tree/soc/serial@1c28000/status    # UART0
cat /proc/device-tree/soc/ethernet@1c30000/status   # Ethernet

# 在 U-Boot 中加载自定义 DTB
setenv fdtfile sun8i-h3-orangepi-pc-custom.dtb
saveenv
```

---

## 相关链接

### 官方资源
- Orange Pi 官方资料: http://www.orangepi.org/
- Allwinner H3 Datasheet: 需从厂商获取 (NDA)
- Linux 内核 Allwinner DTS 源码: `arch/arm/boot/dts/allwinner/sun8i-h3-*.dts`

### Linux 内核文档
- Device Tree 说明: https://www.kernel.org/doc/html/latest/devicetree/
- Allwinner SoC 绑定文档: `Documentation/devicetree/bindings/arm/allwinner/sunxi.yaml`
- OPP 绑定: `Documentation/devicetree/bindings/opp/opp-v2.yaml`
- SY8106A 调节器: `Documentation/devicetree/bindings/regulator/silergy,sy8106a.yaml`

### 社区资源
- linux-sunxi Wiki (H3): https://linux-sunxi.org/H3
- Armbian Orange Pi PC: https://www.armbian.com/orange-pi-pc/
- mainline Linux DT 源码树: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm/boot/dts/allwinner

### DTB 工具
- Device Tree Compiler (dtc): https://git.kernel.org/pub/scm/utils/dtc/dtc.git
- fdtdump 工具: 随内核源码 `scripts/dtc/` 提供
- 在线 DTS 编辑器: https://devicetree.org/

### 芯片参考
- Allwinner H3 SoC: 4x Cortex-A7 @ 最高 1.368GHz, Mali-400 MP2, HDMI, CSI, 4x USB Host, EMAC
- SY8106A DCDC: https://www.silergy.com/ (CPU 核心供电)
- OV5640 Camera: https://www.ovt.com/ (5MP CMOS sensor)
- RTL8189FTV WiFi: https://www.realtek.com/ (SDIO 802.11b/g/n)
