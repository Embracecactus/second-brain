---
tags:
  - embedded-linux
  - bsp
  - i2c
  - spi
  - uart
  - peripheral
  - rockchip
  - driver
category: embedded-linux
created: 2026-06-22
updated: 2026-06-22
status: active
soc: Rockchip RV1126B
kernel: Linux 6.1.141
---

# 阶段四：外设驱动 — I2C + SPI + UART

> **JD对标**：理解常见外设接口协议，如UART、I2C、SPI等
>
> 本章深入 Linux 三大外设子系统。I2C 和 SPI 是 BSP 开发中最常用的总线，几乎所有传感器、PMIC、EEPROM 都挂在这上面。UART 是调试串口的基础。本章目标：能独立编写 I2C/SPI 从设备驱动。

---

## 一、I2C 子系统

### 1.1 I2C 协议基础

```
I2C 总线物理层:
  SDA (数据线) ───── 上拉电阻 ──── VCC
  SCL (时钟线) ───── 上拉电阻 ──── VCC

  主机 (Master/Adapter)    从机 (Slave/Client)
    │ SDA                    │ SDA
    │ SCL                    │ SCL
    └────────────────────────┘

传输时序:
  Start: SDA 在 SCL 高电平时下降沿
  Stop:  SDA 在 SCL 高电平时上升沿
  数据:  SDA 在 SCL 低电平时变化, SCL 高电平时采样
  ACK:   第 9 个时钟, 从机拉低 SDA 表示应答

帧格式 (7-bit 地址):
  [Start] [7-bit addr + R/W] [ACK] [data] [ACK] ... [Stop]
```

### 1.2 Linux I2C 子系统架构

```
应用层
  └── i2c-dev (/dev/i2c-X, 用户态直接访问)

内核层
  ├── i2c-core (核心层, 提供标准 API)
  │   ├── i2c_transfer()        — 主机传输
  │   ├── i2c_smbus_read_*()    — SMBus 读
  │   └── i2c_smbus_write_*()   — SMBus 写
  │
  ├── i2c adapter 驱动 (控制器驱动, SoC 厂商提供)
  │   └── i2c-rk3x.c (RV1126B I2C 控制器)
  │       └── rk3x_i2c_probe() → 注册 i2c_adapter
  │
  └── i2c client 驱动 (从设备驱动, 你要写的)
      └── 例: rk808.c (PMIC), 某传感器驱动
          └── i2c_driver.probe() → i2c_smbus_read_byte_data()

设备树
  &i2c0 { rk801: pmic@27 { ... }; }
  → of_i2c_register_device() 自动创建 i2c_client
  → i2c_driver.of_match_table 匹配 → probe()
```

### 1.3 核心数据结构

```c
/* i2c_adapter — I2C 控制器 (主机) */
struct i2c_adapter {
    struct module *owner;
    unsigned int class;
    const struct i2c_algorithm *algo;  /* 传输算法 */
    struct device dev;
    int nr;                             /* 总线编号 /dev/i2c-N */
    char name[48];
};

/* i2c_algorithm — 控制器传输方法 */
struct i2c_algorithm {
    int (*master_xfer)(struct i2c_adapter *, struct i2c_msg *, int);
    int (*smbus_xfer)(struct i2c_adapter *, u16, unsigned short,
                      char, u8, int, union i2c_smbus_data *);
};

/* i2c_client — I2C 从设备 (挂在总线上的芯片) */
struct i2c_client {
    unsigned short flags;
    unsigned short addr;       /* 7-bit I2C 地址, 如 0x27 */
    char name[I2C_NAME_SIZE];
    struct i2c_adapter *adapter; /* 所属控制器 */
    struct device dev;
    int irq;                   /* 中断号 (如有) */
};

/* i2c_driver — 从设备驱动 */
struct i2c_driver {
    int (*probe)(struct i2c_client *client, const struct i2c_device_id *id);
    int (*remove)(struct i2c_client *client);
    struct device_driver driver;
    const struct i2c_device_id *id_table;
};

/* i2c_msg — 单次传输消息 */
struct i2c_msg {
    __u16 addr;     /* 从设备地址 */
    __u16 flags;    /* I2C_M_RD=读, 0=写 */
    __u16 len;      /* 数据长度 */
    __u8 *buf;      /* 数据缓冲区 */
};
```

### 1.4 I2C 传输 API

```c
/* SMBus 简易 API (最常用) */

/* 读一个字节寄存器 */
s32 i2c_smbus_read_byte_data(const struct i2c_client *client, u8 command);
// 等价: [Start][addr+W][reg][Restart][addr+R][data][NACK][Stop]

/* 写一个字节寄存器 */
s32 i2c_smbus_write_byte_data(const struct i2c_client *client, u8 command, u8 value);

/* 读一个字 (2字节) 寄存器 */
s32 i2c_smbus_read_word_data(const struct i2c_client *client, u8 command);

/* 写一个字 (2字节) 寄存器 */
s32 i2c_smbus_write_word_data(const struct i2c_client *client, u8 command, u16 value);

/* 读一块数据 */
s32 i2c_smbus_read_i2c_block_data(const struct i2c_client *client,
                                   u8 command, u8 length, u8 *values);

/* 写一块数据 */
s32 i2c_smbus_write_i2c_block_data(const struct i2c_client *client,
                                    u8 command, u8 length, const u8 *values);

/* 底层 transfer (需要自定义协议时) */
int i2c_transfer(struct i2c_adapter *adap, struct i2c_msg *msgs, int num);
```

### 1.5 RV1126B I2C 控制器

```dts
/* rv1126b.dtsi — 6 路 I2C 控制器 */
i2c0: i2c@21100000 {
    compatible = "rockchip,rv1126b-i2c";
    reg = <0x21100000 0x1000>;
    interrupts = <GIC_SPI 48 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&cru CLK_I2C0>, <&cru PCLK_I2C0>;
    clock-names = "i2c", "pclk";
    pinctrl-0 = <&i2c0m0_pins>;
    status = "disabled";
};
/* i2c1~5 类似, 基址 0x21110000~0x21140000, IRQ 49~53 */
```

板级启用 (`FET1126B-S.dtsi`)：

```dts
&i2c0 {
    status = "okay";
    rk801: pmic@27 {
        compatible = "rockchip,rk801";
        reg = <0x27>;           /* I2C 7-bit 地址 0x27 */
        /* ... regulators ... */
    };
};
```

### 1.6 板端验证

```bash
# 查看所有 I2C 总线
ls /dev/i2c-*
# 预期: /dev/i2c-0 ~ /dev/i2c-5 (启用的才有)

# 安装 i2c-tools (如果 rootfs 中有)
which i2cdetect

# 扫描 I2C0 总线上的设备
sudo i2cdetect -y 0
# 预期输出:
#        0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
#  00:                         -- -- -- -- -- -- -- --
#  10:                         -- -- -- -- -- -- -- --
#  20:               -- -- -- -- -- -- 27 -- -- -- -- --
#  ...                                          ^
#                                          RK801 PMIC @ 0x27

# 读取 RK801 的 chip ID 寄存器 (地址 0x00)
sudo i2cget -y 0 0x27 0x00 b
# 预期: 0xXX (芯片 ID)

# 查看内核 I2C 设备
ls /sys/bus/i2c/devices/
# 预期: 0-0027 (总线0, 地址0x27 = RK801)

# 查看绑定关系
ls /sys/bus/i2c/devices/0-0027/driver
# 预期: -> ../../../../bus/i2c/drivers/rk808
```

---

## 二、I2C Client 驱动开发

### 2.1 I2C 驱动模板

```c
#include <linux/module.h>
#include <linux/i2c.h>
#include <linux/of.h>

struct my_i2c_data {
    struct i2c_client *client;
    struct mutex lock;
    u8 cached_reg;
};

/* 读寄存器 */
static int my_i2c_read_reg(struct my_i2c_data *data, u8 reg, u8 *val)
{
    s32 ret;
    mutex_lock(&data->lock);
    ret = i2c_smbus_read_byte_data(data->client, reg);
    mutex_unlock(&data->lock);
    if (ret < 0)
        return ret;
    *val = (u8)ret;
    return 0;
}

/* 写寄存器 */
static int my_i2c_write_reg(struct my_i2c_data *data, u8 reg, u8 val)
{
    int ret;
    mutex_lock(&data->lock);
    ret = i2c_smbus_write_byte_data(data->client, reg, val);
    mutex_unlock(&data->lock);
    return ret;
}

static int my_i2c_probe(struct i2c_client *client,
                         const struct i2c_device_id *id)
{
    struct my_i2c_data *data;
    u8 chip_id;
    int ret;

    data = devm_kzalloc(&client->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;

    data->client = client;
    mutex_init(&data->lock);
    i2c_set_clientdata(client, data);

    /* 读取芯片 ID 验证通信 */
    ret = my_i2c_read_reg(data, 0x00, &chip_id);
    if (ret) {
        dev_err(&client->dev, "failed to read chip ID: %d\n", ret);
        return ret;
    }
    dev_info(&client->dev, "chip ID: 0x%02x\n", chip_id);

    return 0;
}

static int my_i2c_remove(struct i2c_client *client)
{
    dev_info(&client->dev, "removed\n");
    return 0;
}

static const struct of_device_id my_i2c_match[] = {
    { .compatible = "my,i2c-sensor" },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, my_i2c_match);

static struct i2c_driver my_i2c_driver = {
    .probe   = my_i2c_probe,
    .remove  = my_i2c_remove,
    .driver  = {
        .name = "my-i2c-sensor",
        .of_match_table = of_match_ptr(my_i2c_match),
    },
};

module_i2c_driver(my_i2c_driver);
MODULE_LICENSE("GPL");
```

### 2.2 设备树配置

```dts
/* 在板级 DTS 中添加 */
&i2c2 {
    status = "okay";

    my_sensor: sensor@48 {
        compatible = "my,i2c-sensor";
        reg = <0x48>;          /* I2C 7-bit 地址 */
        interrupt-parent = <&gpio0>;
        interrupts = <5 IRQ_TYPE_LEVEL_LOW>;
        vdd-supply = <&vcc3v3>;
        status = "okay";
    };
};
```

### 2.3 真实示例：RK801 PMIC 驱动

`drivers/mfd/rk808.c` 是 RV1126B 板上 RK801 PMIC 的 MFD 驱动：

```c
/* rk808.c 匹配 RK801 */
static const struct of_device_id rk808_of_match[] = {
    { .compatible = "rockchip,rk801" },
    ...
};

/* probe 中通过 I2C 读取芯片 ID */
static int rk808_probe(struct i2c_client *client, ...)
{
    /* 读取 PMIC ID 寄存器 */
    msb = i2c_smbus_read_byte_data(client, pmic_id_msb);
    lsb = i2c_smbus_read_byte_data(client, pmic_id_lsb);
    ...
    /* 注册子设备: regulator, rtc, gpio 等 */
    devm_mfd_add_devices(...)
}
```

---

## 三、SPI 子系统

### 3.1 SPI 协议基础

```
SPI 总线物理层 (4 线):
  MOSI (Master Out Slave In)  — 主机→从机数据
  MISO (Master In Slave Out)  — 从机→主机数据
  SCLK (Serial Clock)         — 主机输出的时钟
  CS   (Chip Select)          — 片选, 低有效

SPI 4 种模式 (CPOL + CPHA):
  Mode 0: CPOL=0, CPHA=0  — 空闲低, 第一个边沿采样 (最常用)
  Mode 1: CPOL=0, CPHA=1  — 空闲低, 第二个边沿采样
  Mode 2: CPOL=1, CPHA=0  — 空闲高, 第一个边沿采样
  Mode 3: CPOL=1, CPHA=1  — 空闲高, 第二个边沿采样

全双工: MOSI 和 MISO 同时传输
```

### 3.2 Linux SPI 子系统架构

```
内核层
  ├── spi-core (核心层)
  │   ├── spi_sync()      — 同步传输
  │   ├── spi_async()     — 异步传输
  │   └── spi_message / spi_transfer
  │
  ├── spi master 驱动 (控制器驱动)
  │   └── spi-rockchip.c (RV1126B SPI 控制器)
  │       └── rockchip_spi_probe() → 注册 spi_master
  │
  └── spi device 驱动 (从设备驱动)
      └── 例: 某 SPI Flash / SPI 传感器

设备树
  &spi0 { flash@0 { ... }; }
  → of_register_spi_device() 自动创建 spi_device
  → spi_driver.of_match_table 匹配 → probe()
```

### 3.3 核心数据结构

```c
/* spi_device — SPI 从设备 */
struct spi_device {
    struct device dev;
    struct spi_master *master;  /* 所属控制器 */
    u32 max_speed_hz;           /* 最大时钟频率 */
    u8 chip_select;             /* 片选编号 */
    u8 bits_per_word;           /* 每字比特数 (通常 8) */
    u8 mode;                    /* SPI 模式 (SPI_MODE_0 等) */
    char modalias[SPI_NAME_SIZE];
    int irq;                    /* 中断号 */
};

/* spi_driver — SPI 从设备驱动 */
struct spi_driver {
    int (*probe)(struct spi_device *spi);
    int (*remove)(struct spi_device *spi);
    struct device_driver driver;
};

/* spi_transfer — 单次传输 */
struct spi_transfer {
    const void *tx_buf;         /* 发送缓冲区 (NULL 表示发 0) */
    void *rx_buf;               /* 接收缓冲区 (NULL 表示忽略接收) */
    unsigned len;               /* 传输长度 */
    unsigned speed_hz;          /* 传输速率 (0=用设备默认) */
    u8 bits_per_word;           /* 每字比特数 (0=用设备默认) */
    u8 cs_change;               /* 传输后是否改变 CS */
};

/* spi_message — 一组传输 (原子执行) */
struct spi_message {
    struct list_head transfers;
    ...
};
```

### 3.4 SPI 传输 API

```c
/* 同步传输 (阻塞直到完成) */
int spi_sync(struct spi_device *spi, struct spi_message *message);

/* 异步传输 (完成后回调) */
int spi_async(struct spi_device *spi, struct spi_message *message);

/* 简化 API: 单次写 */
int spi_write(struct spi_device *spi, const void *buf, size_t len);

/* 简化 API: 单次读 (先发 cmd, 再读数据) */
int spi_read(struct spi_device *spi, void *buf, size_t len);

/* 组合传输: 先写命令, 再读数据 */
static int spi_read_reg(struct spi_device *spi, u8 reg, u8 *buf, int len)
{
    struct spi_message msg;
    struct spi_transfer xfer[2];
    u8 tx = reg | 0x80;  /* 假设 bit7=1 表示读 */

    spi_message_init(&msg);

    /* 第一段: 发送寄存器地址 */
    xfer[0].tx_buf = &tx;
    xfer[0].len = 1;
    spi_message_add_tail(&xfer[0], &msg);

    /* 第二段: 接收数据 */
    xfer[1].rx_buf = buf;
    xfer[1].len = len;
    spi_message_add_tail(&xfer[1], &msg);

    return spi_sync(spi, &msg);
}
```

### 3.5 RV1126B SPI 控制器

```dts
/* rv1126b.dtsi */
spi0: spi@211e0000 {
    compatible = "rockchip,rv1126b-spi", "rockchip,rk3066-spi";
    reg = <0x211e0000 0x1000>;
    interrupts = <GIC_SPI 192 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&cru CLK_SPI0>, <&cru PCLK_SPI0>;
    clock-names = "spiclk", "apb_pclk";
    dmas = <&dmac 40>, <&dmac 41>;
    dma-names = "rx", "tx";
    num-cs = <2>;                 /* 2 个片选 */
    status = "disabled";
};
```

---

## 四、UART 子系统

### 4.1 UART 协议基础

```
UART (异步串行通信):
  TX (发送) ──────────────── RX (接收)
  RX (接收) ──────────────── TX (发送)
  GND (共地) ─────────────── GND

帧格式:
  [Start bit] [5~9 data bits] [Parity] [1~2 Stop bits]
  常用: 8N1 = 8 数据位, 无校验, 1 停止位

波特率: 9600, 115200, 1500000(RV1126B 调试串口)
```

### 4.2 Linux UART 子系统架构

```
用户层
  /dev/ttyS0~N (标准 8250 设备)
  /dev/ttyFIQ0 (RV1126B 调试串口, FIQ debugger)

内核层
  ├── tty 核心 (drivers/tty/tty_io.c)
  │   └── 管理所有 tty 设备
  │
  ├── serial core (drivers/tty/serial/serial_core.c)
  │   └── struct uart_driver / uart_port
  │
  └── 8250 驱动 (drivers/tty/serial/8250/)
      ├── 8250_core.c  — 8250 通用逻辑
      ├── 8250_dw.c    — DesignWare APB UART (RV1126B 使用)
      └── 8250_of.c    — OF (设备树) 平台驱动
```

### 4.3 RV1126B UART 驱动

RV1126B 的 8 路 UART 都是 DesignWare APB UART (8250 兼容)：

```dts
/* rv1126b.dtsi */
uart0: serial@20810000 {
    compatible = "rockchip,rv1126b-uart", "snps,dw-apb-uart";
    reg = <0x20810000 0x100>;
    interrupts = <GIC_SPI 56 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&cru SCLK_UART0>, <&cru PCLK_UART0>;
    clock-names = "baudclk", "apb_pclk";
    reg-shift = <2>;            /* 寄存器间隔 = 4 字节 */
    reg-io-width = <4>;         /* 寄存器宽度 = 32 位 */
    status = "disabled";
};
```

驱动匹配 (`8250_dw.c`)：

```c
static const struct of_device_id dw8250_of_match[] = {
    { .compatible = "snps,dw-apb-uart", .data = &dw8250_dw_apb },
    ...
};

/* RV1126B 通过 compatible = "snps,dw-apb-uart" 匹配 */
```

### 4.4 板端验证

```bash
# 查看所有串口设备
ls /dev/ttyS* /dev/ttyFIQ*
# 预期: /dev/ttyFIQ0 (调试串口), /dev/ttyS1~7

# 查看当前控制台
cat /sys/class/tty/console/active
# 预期: ttyFIQ0

# 查看串口波特率
stty -F /dev/ttyS1
# 预期: speed 115200 baud, 8N1

# 设置串口参数
stty -F /dev/ttyS1 115200 cs8 -cstopb -parenb

# 简单回环测试 (短接 TX-RX)
echo "hello" > /dev/ttyS1
cat /dev/ttyS1
```

---

## 五、实验 1：I2C 总线扫描 + PMIC 寄存器读取

### 5.1 实验目标

用 `i2c-tools` 扫描 RV1126B 的 I2C 总线，找到 RK801 PMIC，读取其寄存器验证通信。

### 5.2 操作步骤

```bash
# 板端:

# 1. 查看可用的 I2C 总线
ls /dev/i2c-*
# 预期: /dev/i2c-0 (至少 I2C0 启用, 因为 RK801 挂在上面)

# 2. 扫描 I2C0
sudo i2cdetect -y 0
# 预期: 0x27 位置有设备 (RK801 PMIC)

# 3. 读取 RK801 所有寄存器 (0x00~0xFF)
sudo i2cdump -y 0 0x27 b
# 预期: 输出 256 字节寄存器映射表

# 4. 读取特定寄存器 (例: 芯片 ID, 假设在 0x00)
sudo i2cget -y 0 0x27 0x00 b
# 预期: 0xXX

# 5. 写入并验证 (小心! 不要改关键寄存器)
# 读取一个可安全修改的寄存器
sudo i2cget -y 0 0x27 0xXX b
sudo i2cset -y 0 0x27 0xXX 0xYY b
sudo i2cget -y 0 0x27 0xXX b
# 预期: 返回刚写入的 0xYY
```

### 5.3 分析问题

1. RK801 的 I2C 地址是 0x27，为什么不是 0x4E (0x27 左移 1 位)？
   > 提示: Linux I2C 子系统使用 7-bit 地址，i2c-tools 也是 7-bit。有些硬件手册写 8-bit 地址 (左移 1 位 + R/W bit)。
2. `i2cdetect` 扫描时有些地址显示 `UU`，是什么含义？
   > 提示: `UU` 表示该地址设备已被内核驱动占用，用户态无法直接访问。
3. 如果扫描不到 0x27，可能的原因有哪些？
   > 提示: DTS 中 i2c0 status 未开启 / PMIC 未上电 / I2C 引脚复用配置错误。

---

## 六、实验 2：编写 I2C Client 驱动

### 6.1 实验目标

编写一个 I2C client 驱动，通过 sysfs 暴露 RK801 的寄存器读写接口。

### 6.2 设备树

```dts
/* 如果 RK801 已有 mfd 驱动占用, 可用另一个 I2C 设备做实验 */
&i2c2 {
    status = "okay";

    my_i2c_dev: my-i2c-dev@48 {
        compatible = "my,i2c-test";
        reg = <0x48>;
        status = "okay";
    };
};
```

> **注意**：如果板上没有可用的 I2C 设备，可以用一个 I2C GPIO 扩展器 (如 PCF8574) 或 EEPROM (如 24C02) 做实验。或者修改一个已存在设备的 compatible。

### 6.3 驱动源码 (i2c_demo.c)

```c
#include <linux/module.h>
#include <linux/i2c.h>
#include <linux/of.h>
#include <linux/sysfs.h>
#include <linux/device.h>

struct i2c_demo_data {
    struct i2c_client *client;
    struct mutex lock;
};

/* sysfs: 读取寄存器 (echo "reg" > reg_read) */
static ssize_t reg_read_show(struct device *dev,
                              struct device_attribute *attr, char *buf)
{
    struct i2c_demo_data *data = dev_get_drvdata(dev);
    s32 val;

    mutex_lock(&data->lock);
    val = i2c_smbus_read_byte_data(data->client, 0x00);
    mutex_unlock(&data->lock);

    if (val < 0)
        return sprintf(buf, "read error: %d\n", val);
    return sprintf(buf, "0x%02x\n", val);
}

/* sysfs: 写入寄存器 (echo "reg value" > reg_write) */
static ssize_t reg_write_store(struct device *dev,
                                struct device_attribute *attr,
                                const char *buf, size_t count)
{
    struct i2c_demo_data *data = dev_get_drvdata(dev);
    u8 reg, val;
    int ret;

    if (sscanf(buf, "%hhx %hhx", &reg, &val) != 2)
        return -EINVAL;

    mutex_lock(&data->lock);
    ret = i2c_smbus_write_byte_data(data->client, reg, val);
    mutex_unlock(&data->lock);

    if (ret)
        return ret;
    return count;
}

static DEVICE_ATTR_RO(reg_read);
static DEVICE_ATTR_WO(reg_write);

static struct attribute *i2c_demo_attrs[] = {
    &dev_attr_reg_read.attr,
    &dev_attr_reg_write.attr,
    NULL,
};
ATTRIBUTE_GROUPS(i2c_demo);

static int i2c_demo_probe(struct i2c_client *client,
                           const struct i2c_device_id *id)
{
    struct i2c_demo_data *data;
    s32 chip_id;

    data = devm_kzalloc(&client->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;

    data->client = client;
    mutex_init(&data->lock);
    i2c_set_clientdata(client, data);

    /* 验证 I2C 通信 */
    chip_id = i2c_smbus_read_byte_data(client, 0x00);
    if (chip_id < 0) {
        dev_err(&client->dev, "I2C read failed: %d\n", chip_id);
        return chip_id;
    }
    dev_info(&client->dev, "I2C device probed, reg[0]=0x%02x\n", chip_id);

    /* 创建 sysfs 接口 */
    devm_device_add_groups(&client->dev, i2c_demo_groups);

    return 0;
}

static int i2c_demo_remove(struct i2c_client *client)
{
    dev_info(&client->dev, "removed\n");
    return 0;
}

static const struct of_device_id i2c_demo_match[] = {
    { .compatible = "my,i2c-test" },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, i2c_demo_match);

static struct i2c_driver i2c_demo_driver = {
    .probe   = i2c_demo_probe,
    .remove  = i2c_demo_remove,
    .driver  = {
        .name = "i2c-demo",
        .of_match_table = of_match_ptr(i2c_demo_match),
    },
};

module_i2c_driver(i2c_demo_driver);
MODULE_LICENSE("GPL");
```

### 6.4 测试

```bash
# 加载驱动
sudo insmod /tmp/i2c_demo.ko
# 预期: "I2C device probed, reg[0]=0xXX"

# 通过 sysfs 读取寄存器 0x00
cat /sys/bus/i2c/devices/2-0048/reg_read
# 预期: 0xXX

# 通过 sysfs 写入寄存器
echo "0x10 0xAB" | sudo tee /sys/bus/i2c/devices/2-0048/reg_write

# 验证写入
sudo i2cget -y 2 0x48 0x10 b
# 预期: 0xab
```

---

## 七、实验 3：Ftrace 追踪 I2C 传输路径

### 7.1 实验目标

用 Ftrace 追踪 `i2c_smbus_read_byte_data` 从用户态到硬件寄存器读写的完整调用链。

### 7.2 操作步骤

```bash
# 板端:
echo function_graph | sudo tee /sys/kernel/tracing/current_tracer
echo '*i2c*' | sudo tee /sys/kernel/tracing/set_ftrace_filter
echo '*rk3x*' | sudo tee -a /sys/kernel/tracing/set_ftrace_filter
echo 128 | sudo tee /sys/kernel/tracing/max_graph_depth

echo | sudo tee /sys/kernel/tracing/trace

# 触发 I2C 读
sudo i2cget -y 0 0x27 0x00 b

# 保存追踪
sudo cat /sys/kernel/tracing/trace > /tmp/i2c_trace.log
echo nop | sudo tee /sys/kernel/tracing/current_tracer
```

### 7.3 预期调用链

```
用户态:
  i2cget → ioctl(/dev/i2c-0, I2C_SMBUS, ...)

内核态:
  i2cdev_ioctl()
    → i2c_smbus_xfer()
      → i2c_smbus_xfer_emulated()    (如果控制器不支持 SMBUS 硬件)
        → i2c_transfer()
          → __i2c_transfer()
            → adapter->algo->master_xfer()  ← rk3x_i2c_xfer
              → rk3x_i2c_start()
              → rk3x_i2c_write_reg()        (写寄存器地址)
              → wait_for_completion()        (等待中断)
                → rk3x_i2c_irq()             (I2C 控制器中断)
                  → complete()
              → rk3x_i2c_read()              (读数据)
              → wait_for_completion()
                → rk3x_i2c_irq()
              → rk3x_i2c_stop()
```

> **关键发现**：`i2c_transfer` 是同步阻塞的，内部通过 `wait_for_completion` 等待 I2C 控制器中断。这说明 I2C 驱动是"中断驱动"的——发起传输后 CPU 睡眠，硬件完成后中断唤醒 CPU。

---

## 八、实验 4：SPI 设备驱动 (如有 SPI 外设)

### 8.1 实验目标

编写 SPI 设备驱动，通过 `spi_sync` 读写 SPI 从设备寄存器。

### 8.2 设备树

```dts
&spi0 {
    status = "okay";
    #address-cells = <1>;
    #size-cells = <0>;

    my_spi_dev: spi-device@0 {
        compatible = "my,spi-test";
        reg = <0>;                    /* CS 0 */
        spi-max-frequency = <1000000>; /* 1MHz */
        spi-cpol = <0>;               /* CPOL=0 */
        spi-cpha = <0>;               /* CPHA=0 → Mode 0 */
        status = "okay";
    };
};
```

### 8.3 驱动源码 (spi_demo.c)

```c
#include <linux/module.h>
#include <linux/spi/spi.h>
#include <linux/of.h>

struct spi_demo_data {
    struct spi_device *spi;
    struct mutex lock;
};

/* 读 SPI 寄存器: 先发 reg|0x80, 再读 1 字节 */
static int spi_demo_read_reg(struct spi_demo_data *data, u8 reg, u8 *val)
{
    u8 tx[2] = { reg | 0x80, 0xFF };  /* 0xFF 是 dummy 用于读 */
    u8 rx[2];
    struct spi_message msg;
    struct spi_transfer xfer = {
        .tx_buf = tx,
        .rx_buf = rx,
        .len = 2,
    };
    int ret;

    mutex_lock(&data->lock);
    spi_message_init(&msg);
    spi_message_add_tail(&xfer, &msg);
    ret = spi_sync(data->spi, &msg);
    mutex_unlock(&data->lock);

    if (ret)
        return ret;
    *val = rx[1];
    return 0;
}

/* 写 SPI 寄存器: 先发 reg&0x7F, 再发 value */
static int spi_demo_write_reg(struct spi_demo_data *data, u8 reg, u8 val)
{
    u8 tx[2] = { reg & 0x7F, val };
    return spi_write(data->spi, tx, 2);
}

static int spi_demo_probe(struct spi_device *spi)
{
    struct spi_demo_data *data;
    u8 chip_id;
    int ret;

    data = devm_kzalloc(&spi->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;

    data->spi = spi;
    mutex_init(&data->lock);
    spi_set_drvdata(spi, data);

    /* 设置 SPI 模式 (覆盖设备树) */
    spi->mode = SPI_MODE_0;
    spi->bits_per_word = 8;
    ret = spi_setup(spi);
    if (ret)
        return ret;

    /* 读取芯片 ID */
    ret = spi_demo_read_reg(data, 0x00, &chip_id);
    if (ret) {
        dev_err(&spi->dev, "SPI read failed: %d\n", ret);
        return ret;
    }
    dev_info(&spi->dev, "SPI device probed, chip_id=0x%02x\n", chip_id);

    return 0;
}

static int spi_demo_remove(struct spi_device *spi)
{
    dev_info(&spi->dev, "removed\n");
    return 0;
}

static const struct of_device_id spi_demo_match[] = {
    { .compatible = "my,spi-test" },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, spi_demo_match);

static struct spi_driver spi_demo_driver = {
    .probe  = spi_demo_probe,
    .remove = spi_demo_remove,
    .driver = {
        .name = "spi-demo",
        .of_match_table = of_match_ptr(spi_demo_match),
    },
};

module_spi_driver(spi_demo_driver);
MODULE_LICENSE("GPL");
```

---

## 九、三大总线对比

| 维度 | I2C | SPI | UART |
|------|-----|-----|------|
| 线数 | 2 (SDA+SCL) | 4+ (MOSI+MISO+SCLK+CS) | 2 (TX+RX) |
| 双工 | 半双工 | 全双工 | 全双工 |
| 速率 | 100K/400K/1MHz | 通常 1~50MHz | 115200~1500000 baud |
| 寻址 | 7-bit 地址 | CS 片选 | 无寻址 (点对点) |
| 拓扑 | 多从机 (地址区分) | 多从机 (CS 区分) | 点对点 |
| Linux 驱动类型 | `i2c_driver` | `spi_driver` | `uart_driver` (通常不需写) |
| 设备树节点 | `i2c@XX { device@YY }` | `spi@XX { device@0 }` | `serial@XX` |
| 传输 API | `i2c_smbus_read/write_*` | `spi_sync / spi_write / spi_read` | `tty API` (用户态) |
| 是否需要写控制器驱动 | 否 (SoC 厂商已写) | 否 (SoC 厂商已写) | 否 (SoC 厂商已写) |
| **你需要写什么** | **从设备驱动** | **从设备驱动** | **通常不用写 (用户态操作)** |

---

## 十、思考题

1. I2C 的 `i2c_smbus_read_byte_data` 内部实际发送了几次 Start 信号？画出完整的 SDA 时序图。

2. SPI 的 `spi_sync` 是同步阻塞的，如果 SPI 传输数据量很大（如 4MB），会阻塞很久。如何优化？提示：`spi_async` + completion。

3. RV1126B 的 UART 驱动用的是 8250/DesignWare 通用驱动 (`8250_dw.c`)，而不是 Rockchip 自研的。这说明什么？如果你要给 RV1126B 添加一个自定义 UART IP 核，需要写哪些代码？

4. I2C 驱动中用 `mutex` 保护 `i2c_smbus_read_byte_data` 调用，但 I2C 控制器驱动内部已经有锁。这两层锁的关系是什么？去掉外层 mutex 会有什么风险？

5. 设备树中 I2C 子节点的 `reg = <0x27>` 是 7-bit 地址。但有些芯片手册写 `0x4E`（8-bit，左移 1 位）。如果你在 DTS 中写了 `reg = <0x4E>`，会发生什么？如何排查？

---

## 十一、踩坑记录

| 日期 | 问题 | 原因 | 解决方案 |
|------|------|------|----------|
| | i2cdetect 扫不到设备 | DTS status=disabled 或引脚复用错误 | 确认 `&i2cX { status = "okay"; }` |
| | i2c_smbus_read 返回 -ENXIO | 地址错误或设备未上电 | 检查 `reg = <addr>` 和手册地址 |
| | i2c_smbus_read 返回 -ETIMEDOUT | I2C 时钟过快或上拉电阻太大 | 降低 clock-frequency 或检查硬件 |
| | probe 未调用 | of_match_table compatible 不匹配 | 确认 DTS 和驱动 compatible 完全一致 |
| | SPI 传输数据错乱 | SPI mode (CPOL/CPHA) 不对 | 对照芯片手册设置 spi-cpol/spi-cpha |
| | i2cdetect 显示 UU | 设备已被内核驱动占用 | 正常现象, 用 sysfs 而非 i2c-dev 访问 |

---

## 十二、下阶段预告

阶段五：**电源管理**
- Clock 框架 (CCF)：`clk-rv1126b.c`、`clk_prepare_enable`
- Regulator 框架：`rk801-regulator.c`、`regulator_enable`
- Runtime PM：`pm_runtime_get_sync` / `pm_runtime_put`
- PM Domains：`pm_domains.c`、`RV1126B_PD_NPU/VDO/AISP`
- Suspend/Resume 测试

---

## 相关笔记

- [[MOC-嵌入式Linux]] — 学习路线总览
- [[bsp-boot-flow]] — 阶段一：Bootloader + 启动流程
- [[bsp-device-model-dtb]] — 阶段二：设备模型 + 设备树
- [[bsp-interrupt-concurrency]] — 阶段三：中断处理 + 并发
- [[kernel-debug-env]] — 附录A：内核调试环境 (Ftrace 使用)
