---
tags:
  - embedded-linux
  - bsp
  - device-model
  - device-tree
  - platform-driver
  - rockchip
  - kernel-module
category: embedded-linux
created: 2026-06-22
updated: 2026-06-22
status: active
soc: Rockchip RV1126B
kernel: Linux 6.1.141
dts: rv1126b.dtsi (4048 lines)
---

# 阶段二：Linux 设备模型 + 设备树

> **JD对标**：Linux设备模型、Kernel移植、驱动开发基础
>
> 本章是 BSP 驱动开发的核心基础。设备树描述"硬件有什么"，设备模型定义"驱动怎么和硬件匹配"，platform driver 是最常见的驱动框架。理解这三者，才能在阶段三~六中写出真正的驱动。

---

## 一、Linux 设备模型

### 1.1 三要素：Bus / Device / Driver

```
┌──────────────────────────────────────────┐
│              Linux 设备模型               │
├──────────────────────────────────────────┤
│                                          │
│   Bus (总线)                              │
│   ├── platform_bus (虚拟总线, 最常用)      │
│   ├── i2c_bus                             │
│   ├── spi_bus                             │
│   ├── usb_bus                             │
│   └── pci_bus                             │
│                                          │
│   Device (设备)                           │
│   ├── struct platform_device              │
│   ├── struct i2c_client                   │
│   ├── struct spi_device                   │
│   └── ... (每个硬件外设一个实例)            │
│                                          │
│   Driver (驱动)                           │
│   ├── struct platform_driver              │
│   ├── struct i2c_driver                   │
│   ├── struct spi_driver                   │
│   └── ... (每个驱动一个实例)               │
│                                          │
│   匹配机制: bus->match(device, driver)    │
│   → driver->probe(device)                 │
│                                          │
└──────────────────────────────────────────┘
```

### 1.2 匹配流程

```
设备树加载 → 为每个 DTS 节点创建 platform_device
驱动注册 → platform_driver_register → 挂到 platform_bus

匹配条件:
  1. of_match_table 中 compatible 字符串匹配 DTS 节点的 compatible
  2. 或 name 字段匹配 (旧式, 不推荐)

匹配成功 → 调用 driver->probe(platform_device)
          → probe 中初始化硬件、注册字符设备等
```

### 1.3 核心数据结构

```c
/* platform_driver — 驱动侧 */
struct platform_driver {
    int     (*probe)(struct platform_device *pdev);    /* 匹配后调用 */
    int     (*remove)(struct platform_device *pdev);   /* 卸载时调用 */
    void    (*shutdown)(struct platform_device *pdev); /* 关机时调用 */
    int     (*suspend)(struct platform_device *pdev, pm_message_t state);
    int     (*resume)(struct platform_device *pdev);
    struct device_driver  driver;                      /* 继承自 device_driver */
    const struct platform_device_id *id_table;         /* 旧式 ID 匹配 */
};

/* device_driver — 所有驱动的基类 */
struct device_driver {
    const char              *name;
    const struct of_device_id  *of_match_table;  /* 设备树匹配表 */
    struct module           *owner;
    struct driver_private   *p;                   /* 内核内部数据 */
};

/* platform_device — 设备侧 (通常由内核从 DTS 自动创建) */
struct platform_device {
    const char  *name;
    int         id;
    struct device dev;
    u32         num_resources;
    struct resource *resource;     /* 内存/中断资源 */
};
```

### 1.4 sysfs 中的设备模型

```bash
# 板端：查看设备模型层级
ls /sys/bus/platform/devices/     # 所有 platform 设备
ls /sys/bus/platform/drivers/     # 所有 platform 驱动

# 查看具体设备
ls /sys/bus/platform/devices/i2c0/
# 预期: uevent, modalias, of_node, driver -> ..., ...

# 查看 modalias (内核用来匹配驱动的字符串)
cat /sys/bus/platform/devices/i2c0/modalias
# 预期: of:NplatformT<Crockchip,rv1126b-i2c>

# 查看设备和驱动的绑定关系
ls -la /sys/bus/platform/devices/i2c0/driver
# 预期: -> ../../../../bus/platform/drivers/rockchip-i2c
```

---

## 二、设备树 (Device Tree)

### 2.1 设备树文件层次

```
kernel-6.1/arch/arm64/boot/dts/rockchip/
├── rv1126b.dtsi          # SoC 级定义 (4048行, 所有 IP 核)
├── rv1126b-amp.dtsi      # AMP 架构补充
├── FET1126B-S.dtsi       # 载板级定义 (PMIC, eMMC, 外设使能)
└── rv1126b-sportcam.dts  # 板级入口 (顶层, include 以上三个)
```

```
编译产物:
  rv1126b-sportcam.dts → rv1126b-sportcam.dtb
  打包进 boot.img (FIT 镜像)
```

### 2.2 设备树语法

```dts
/* 节点基本格式 */
[label:] node-name[@unit-address] {
    property-name = <value>;           /* 32位整数 */
    property-name = "string";          /* 字符串 */
    property-name = <0x1 0x2 0x3>;    /* 整数数组 */
    #address-cells = <1>;              /* 子节点 reg 中地址占几个 cell */
    #size-cells = <1>;                 /* 子节点 reg 中大小占几个 cell */

    child-node {
        ...
    };
};

/* phandle 引用 */
clocks = <&cru CLK_I2C0>;   /* &cru 引用 cru 节点, CLK_I2C0 是宏 */
interrupts = <GIC_SPI 53 IRQ_TYPE_LEVEL_HIGH>;
```

### 2.3 关键属性速查

| 属性 | 说明 | 示例 |
|------|------|------|
| `compatible` | 驱动匹配字符串 (最重要) | `"rockchip,rv1126b-i2c"` |
| `reg` | 寄存器基址 + 大小 | `<0x21140000 0x1000>` |
| `interrupts` | 中断号 + 触发类型 | `<GIC_SPI 53 IRQ_TYPE_LEVEL_HIGH>` |
| `clocks` | 引用的时钟 | `<&cru CLK_I2C5>` |
| `clock-names` | 时钟名称 | `"i2c", "pclk"` |
| `pinctrl-0` | 引脚复用配置 | `<&i2c5m0_pins>` |
| `status` | 使能/禁用 | `"okay"` 或 `"disabled"` |
| `dma-names` | DMA 通道名 | `"rx", "tx"` |
| `resets` | 复位控制器引用 | `<&cru SRST_I2C5>` |
| `#address-cells` | 子节点地址 cell 数 | `<1>` |
| `#size-cells` | 子节点大小 cell 数 | `<0>` |

### 2.4 interrupts 宏定义

```c
// dt-bindings/interrupt-controller/arm-gic.h
#define GIC_SPI 0    // Shared Peripheral Interrupt (共享外设中断)
#define GIC_PPI 1    // Private Peripheral Interrupt (私有外设中断)

// dt-bindings/interrupt-controller/irq.h
#define IRQ_TYPE_NONE           0
#define IRQ_TYPE_EDGE_RISING    1
#define IRQ_TYPE_EDGE_FALLING   2
#define IRQ_TYPE_EDGE_BOTH      (1 | 2)
#define IRQ_TYPE_LEVEL_HIGH     4
#define IRQ_TYPE_LEVEL_LOW      8
```

示例：`interrupts = <GIC_SPI 53 IRQ_TYPE_LEVEL_HIGH>` = SPI 中断号 53，高电平触发。

### 2.5 compatible 匹配规则

```dts
/* DTS 节点 */
i2c5: i2c@21140000 {
    compatible = "rockchip,rv1126b-i2c";  /* 驱动用这个字符串匹配 */
    ...
};
```

```c
/* 驱动中的 of_match_table */
static const struct of_device_id rk3x_i2c_match[] = {
    { .compatible = "rockchip,rv1126b-i2c" },   /* 精确匹配 */
    { .compatible = "rockchip,rv1126-i2c" },    /* 兼容旧型号 */
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, rk3x_i2c_match);

static struct platform_driver rk3x_i2c_driver = {
    .probe  = rk3x_i2c_probe,
    .remove = rk3x_i2c_remove,
    .driver = {
        .name  = "rockchip-i2c",
        .of_match_table = of_match_ptr(rk3x_i2c_match),
    },
};
```

> **匹配规则**：内核从 DTS 节点的 `compatible` 列表中，从左到右依次尝试匹配驱动的 `of_match_table`。所以 `compatible = "rockchip,rv1126b-i2c", "rockchip,rv1126-i2c"` 表示先匹配 rv1126b 专用驱动，如果没有则退而匹配 rv1126 通用驱动。

---

## 三、RV1126B 设备树解析

### 3.1 SoC 顶层结构 (rv1126b.dtsi)

```dts
/ {
    compatible = "rockchip,rv1126b";
    interrupt-parent = <&gic>;
    #address-cells = <1>;
    #size-cells = <1>;

    aliases {          /* 设备别名, 决定 /dev/video0, i2c-0 等编号 */
        serial0 = &uart0;
        i2c0 = &i2c0;
        ...
    };

    cpus { ... }       /* CPU 拓扑 (4× Cortex-A53) */
    psci { ... }       /* PSCI 电源管理接口 */

    /* SoC 内设 (通过 simple-bus 或直接定义) */
    cru: ...           /* Clock & Reset Unit (CRU) */
    gic: ...           /* GIC-400 中断控制器 */
    i2c0~5: ...        /* 6 路 I2C 控制器 */
    uart0~7: ...       /* 8 路 UART */
    spi0~1: ...        /* 2 路 SPI + 2 路 FSPI */
    gpio0~7: ...       /* 8 组 GPIO */
    emmc: ...          /* eMMC 控制器 */
    usb2phy: ...       /* USB2 PHY */
    ...
};
```

### 3.2 典型节点分析：I2C5

```dts
i2c5: i2c@21140000 {
    compatible = "rockchip,rv1126b-i2c";         /* 驱动匹配字符串 */
    reg = <0x21140000 0x1000>;                   /* 寄存器: 基址 0x21140000, 大小 0x1000 */
    interrupts = <GIC_SPI 53 IRQ_TYPE_LEVEL_HIGH>;  /* GIC SPI 53, 高电平触发 */
    #address-cells = <1>;                        /* 子设备地址占 1 cell */
    #size-cells = <0>;                           /* 子设备无 size */
    clocks = <&cru CLK_I2C5>, <&cru PCLK_I2C5>;  /* 两路时钟: 功能时钟 + APB 时钟 */
    clock-names = "i2c", "pclk";
    pinctrl-names = "default";
    pinctrl-0 = <&i2c5m0_pins>;                  /* 引脚复用: I2C5 mux-0 组 */
    status = "disabled";                         /* 默认禁用, 板级 dts 中开启 */
};
```

### 3.3 板级覆盖：FET1126B-S.dtsi

```dts
/* 板级 DTS 覆盖 SoC DTS 的 status */
&i2c0 {
    status = "okay";          /* 启用 I2C0 */

    rk801: pmic@27 {
        compatible = "rockchip,rk801";
        reg = <0x27>;         /* I2C 从设备地址 0x27 */
        ...
        regulators {
            vdd_npu: DCDC1 { ... }
            vcc_ddr: DCDC3 { ... }
        };
    };
};
```

> **理解 DTS 层叠机制**：SoC `rv1126b.dtsi` 定义所有 IP 核但默认 `status = "disabled"`，板级 DTS (如 `FET1126B-S.dtsi`) 通过 `&i2c0 { status = "okay"; }` 启用实际使用的外设，并添加板级特有设备（如 PMIC、传感器）。

---

## 四、Platform Driver 开发

### 4.1 最小 platform driver 模板

```c
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/of.h>

static const struct of_device_id hello_match[] = {
    { .compatible = "my,hello-device" },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, hello_match);

static int hello_probe(struct platform_device *pdev)
{
    dev_info(&pdev->dev, "probe: device matched and initialized\n");
    return 0;
}

static int hello_remove(struct platform_device *pdev)
{
    dev_info(&pdev->dev, "remove: device removed\n");
    return 0;
}

static struct platform_driver hello_driver = {
    .probe  = hello_probe,
    .remove = hello_remove,
    .driver = {
        .name = "hello-device",
        .of_match_table = of_match_ptr(hello_match),
    },
};

module_platform_driver(hello_driver);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Student");
MODULE_DESCRIPTION("Minimal platform driver demo");
```

### 4.2 module_platform_driver 宏展开

```c
/* module_platform_driver(hello_driver) 展开为: */
static int __init hello_driver_init(void)
{
    return platform_driver_register(&hello_driver);
}
module_init(hello_driver_init);

static void __exit hello_driver_exit(void)
{
    platform_driver_unregister(&hello_driver);
}
module_exit(hello_driver_exit);
```

### 4.3 probe 中获取 DTS 资源

```c
static int hello_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct resource *res;
    void __iomem *base;
    int irq;
    u32 reg_val;

    /* 1. 获取寄存器基地址 (从 DTS 的 reg 属性) */
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    if (!res) {
        dev_err(dev, "no memory resource\n");
        return -ENODEV;
    }
    base = devm_ioremap_resource(dev, res);
    if (IS_ERR(base))
        return PTR_ERR(base);

    /* 2. 获取中断号 (从 DTS 的 interrupts 属性) */
    irq = platform_get_irq(pdev, 0);
    if (irq < 0) {
        dev_err(dev, "no irq resource\n");
        return irq;
    }

    /* 3. 读取 DTS 自定义属性 */
    if (of_property_read_u32(dev->of_node, "my-custom-value", &reg_val)) {
        dev_warn(dev, "my-custom-value not set, using default\n");
        reg_val = 0;
    }
    dev_info(dev, "custom value = %u\n", reg_val);

    /* 4. 读取时钟并使能 */
    struct clk *clk = devm_clk_get(dev, "pclk");
    if (IS_ERR(clk))
        return PTR_ERR(clk);
    clk_prepare_enable(clk);

    return 0;
}
```

### 4.4 devm_ 资源管理

`devm_` 前缀的函数自动管理资源生命周期：

| 函数 | 自动释放 |
|------|---------|
| `devm_ioremap_resource()` | iounmap |
| `devm_kzalloc()` | kfree |
| `devm_clk_get()` | clk_put |
| `devm_request_irq()` | free_irq |
| `devm_gpio_request()` | gpio_free |

> **好处**：probe 失败或 remove 时，内核自动释放所有 `devm_` 分配的资源，无需手动 free。这是现代驱动开发的标准做法。

---

## 五、字符设备

### 5.1 字符设备注册流程

```
probe() 中:
  1. alloc_chrdev_region() 或 register_chrdev_region() → 获取主/次设备号
  2. cdev_init()  → 初始化 cdev 结构
  3. cdev_add()   → 注册到内核
  4. class_create()  → 创建设备类 (/sys/class/xxx)
  5. device_create() → 创建设备节点 (/dev/xxx, 由 udev 自动创建)
```

### 5.2 字符设备驱动模板

```c
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/platform_device.h>
#include <linux/of.h>
#include <linux/uaccess.h>

#define DEVICE_NAME "hello"
#define BUF_SIZE    256

struct hello_dev {
    struct cdev cdev;
    dev_t devno;
    struct class *cls;
    struct device *dev;
    char buf[BUF_SIZE];
};

static struct hello_dev *hello;

static int hello_open(struct inode *inode, struct file *filp)
{
    filp->private_data = hello;
    return 0;
}

static ssize_t hello_read(struct file *filp, char __user *buf,
                           size_t count, loff_t *ppos)
{
    struct hello_dev *dev = filp->private_data;
    int len = strlen(dev->buf);

    if (*ppos >= len)
        return 0;
    if (count > len - *ppos)
        count = len - *ppos;

    if (copy_to_user(buf, dev->buf + *ppos, count))
        return -EFAULT;

    *ppos += count;
    return count;
}

static ssize_t hello_write(struct file *filp, const char __user *buf,
                            size_t count, loff_t *ppos)
{
    struct hello_dev *dev = filp->private_data;

    if (count >= BUF_SIZE)
        count = BUF_SIZE - 1;

    if (copy_from_user(dev->buf, buf, count))
        return -EFAULT;

    dev->buf[count] = '\0';
    return count;
}

static const struct file_operations hello_fops = {
    .owner = THIS_MODULE,
    .open  = hello_open,
    .read  = hello_read,
    .write = hello_write,
};

static int hello_probe(struct platform_device *pdev)
{
    int ret;

    hello = devm_kzalloc(&pdev->dev, sizeof(*hello), GFP_KERNEL);
    if (!hello)
        return -ENOMEM;

    /* 1. 动态分配设备号 */
    ret = alloc_chrdev_region(&hello->devno, 0, 1, DEVICE_NAME);
    if (ret) {
        dev_err(&pdev->dev, "alloc_chrdev_region failed\n");
        return ret;
    }

    /* 2. 初始化并添加 cdev */
    cdev_init(&hello->cdev, &hello_fops);
    hello->cdev.owner = THIS_MODULE;
    ret = cdev_add(&hello->cdev, hello->devno, 1);
    if (ret) {
        unregister_chrdev_region(hello->devno, 1);
        return ret;
    }

    /* 3. 创建设备类和节点 (udev 自动创建 /dev/hello) */
    hello->cls = class_create(THIS_MODULE, DEVICE_NAME);
    hello->dev = device_create(hello->cls, &pdev->dev,
                                hello->devno, NULL, DEVICE_NAME);

    platform_set_drvdata(pdev, hello);
    dev_info(&pdev->dev, "hello device registered: major=%d minor=%d\n",
             MAJOR(hello->devno), MINOR(hello->devno));
    return 0;
}

static int hello_remove(struct platform_device *pdev)
{
    struct hello_dev *dev = platform_get_drvdata(pdev);

    device_destroy(dev->cls, dev->devno);
    class_destroy(dev->cls);
    cdev_del(&dev->cdev);
    unregister_chrdev_region(dev->devno, 1);
    dev_info(&pdev->dev, "hello device unregistered\n");
    return 0;
}

static const struct of_device_id hello_match[] = {
    { .compatible = "my,hello-device" },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, hello_match);

static struct platform_driver hello_driver = {
    .probe  = hello_probe,
    .remove = hello_remove,
    .driver = {
        .name = "hello-device",
        .of_match_table = of_match_ptr(hello_match),
    },
};

module_platform_driver(hello_driver);
MODULE_LICENSE("GPL");
```

---

## 六、实验 1：解析 RV1126B 设备树拓扑

### 6.1 实验目标

从 `rv1126b.dtsi` 提取所有 IP 核节点，画出 SoC 硬件拓扑图。

### 6.2 操作步骤

```bash
# PC 端: 解析 DTS 中的所有节点
# 提取所有有 compatible 属性的节点
grep -n "compatible" kernel-6.1/arch/arm64/boot/dts/rockchip/rv1126b.dtsi | \
  grep -v "//" | head -40

# 统计各类外设数量
grep "rockchip,rv1126b-" kernel-6.1/arch/arm64/boot/dts/rockchip/rv1126b.dtsi | \
  sort -u

# 板端: 用 /proc/device-tree 验证实际加载的 DTS
ls /proc/device-tree/
ls /proc/device-tree/i2c@21140000/
cat /proc/device-tree/i2c@21140000/compatible
# 预期: rockchip,rv1126b-i2c

# 板端: 查看哪些节点 status=okay
for node in /proc/device-tree/*/; do
    if [ -f "${node}status" ]; then
        status=$(tr -d '\0' < "${node}status")
        if [ "$status" = "okay" ]; then
            echo "ENABLED: $(basename $node)"
        fi
    fi
done
```

### 6.3 预期拓扑图

```
RV1126B SoC
├── CPU: 4× Cortex-A53 @1.6GHz (PSCI)
├── GIC-400 @0x21201000 (GICv2)
├── CRU @0x20000000 (Clock & Reset Unit)
├── PMU @0x20838000 (Power Management)
│
├── I2C ×6
│   ├── i2c0 @0x21100000 → RK801 PMIC @0x27
│   ├── i2c1 @0x21110000 (DMA)
│   ├── i2c2 @0x20800000
│   ├── i2c3 @0x21120000 (DMA)
│   ├── i2c4 @0x21130000
│   └── i2c5 @0x21140000
│
├── UART ×8
│   ├── uart0 @0x20810000 (调试串口, ttyFIQ0)
│   ├── uart1~7 @0x21160000~0x211c0000
│
├── SPI ×2 + FSPI ×2
│   ├── spi0 @0x211e0000
│   ├── spi1 @0x211f0000
│   ├── fspi0 @0x21460000 (SPI NAND Flash)
│   └── fspi1 @0x208c0000
│
├── USB
│   ├── usb2phy @0x21400000
│   ├── usb3phy @0x21410000
│   └── dwc3 (OTG) + ehci + ohci
│
├── 存储
│   ├── emmc @0x21470000 (8-bit HS200)
│   └── sdmmc1 (SDIO WiFi: RTL8821CS)
│
├── 多媒体
│   ├── RKISP1 (ISP)
│   ├── RKCIF (Camera Interface)
│   ├── MPP (VEPU511 编码 + VDPU384A 解码)
│   ├── RGA2 (2D 加速)
│   └── RKNPU (2.0 TOPS)
│
└── 其他
    ├── RTC @0x21280000
    ├── RNG @0x20950000
    ├── Crypto @0x20940000
    ├── TSADC @0x20bb0000
    └── Mailbox ×4 (MCU 通信)
```

---

## 七、实验 2：编写第一个内核驱动 — /dev/hello

### 7.1 实验目标

编写一个完整的 platform driver + 字符设备，通过设备树匹配后创建 `/dev/hello`，支持 `read`/`write`。

### 7.2 源码

使用第五节的字符设备驱动模板。保存为 `hello_drv.c`。

### 7.3 设备树修改

```dts
/* 在 rv1126b-sportcam.dts 中添加自定义节点 */
/ {
    hello_device: hello-device {
        compatible = "my,hello-device";
        status = "okay";
        my-custom-value = <0x1234>;
    };
};
```

### 7.4 Makefile

```makefile
# Makefile — 外部模块编译
obj-m += hello_drv.o

KERNEL_DIR := $(SDK_ROOT)/kernel-6.1
ARCH := arm64
CROSS_COMPILE := aarch64-none-linux-gnu-

all:
    make -C $(KERNEL_DIR) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) \
         M=$(PWD) modules

clean:
    make -C $(KERNEL_DIR) ARCH=$(ARCH) M=$(PWD) clean
```

### 7.5 编译 & 部署

```bash
# PC 端编译
export PATH=$PWD/prebuilts/gcc/linux-x86/aarch64/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin:$PATH
make

# 部署到板子
scp hello_drv.ko rooter@192.168.1.109:/tmp/

# 重新编译包含 hello-device 节点的 dtb
# 修改 rv1126b-sportcam.dts 后:
./build.sh kernel
# 用 dd 刷写 boot 分区的 dtb (或重新打包 boot.img)

# 板端加载驱动
sudo insmod /tmp/hello_drv.ko
# 预期 dmesg: "hello device registered: major=XXX minor=0"

# 验证设备节点
ls -la /dev/hello
# 预期: crw------- 1 root root ...

# 测试读写
echo "Hello BSP" | sudo tee /dev/hello
cat /dev/hello
# 预期: Hello BSP

# 查看驱动绑定
ls -la /sys/bus/platform/drivers/hello-device/
# 预期: hello-device -> ../../../devices/platform/...
```

### 7.6 卸载

```bash
sudo rmmod hello_drv
# 预期 dmesg: "hello device unregistered"
ls /dev/hello
# 预期: No such file (设备节点自动删除)
```

---

## 八、实验 3：设备树属性读取

### 8.1 实验目标

在驱动 probe 中读取 DTS 自定义属性，验证 `of_property_read_*` 系列 API。

### 8.2 设备树

```dts
hello-device {
    compatible = "my,hello-device";
    status = "okay";
    my-custom-value = <0x1234>;
    my-string = "dji-bsp";
    my-array = <10 20 30 40 50>;
    my-flags = <1 0 1>;  /* enable, invert, debug */
};
```

### 8.3 probe 代码

```c
static int hello_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    u32 val, arr[5];
    const char *str;
    int i;

    /* 读取单个 u32 */
    of_property_read_u32(dev->of_node, "my-custom-value", &val);
    dev_info(dev, "my-custom-value = 0x%x\n", val);

    /* 读取字符串 */
    of_property_read_string(dev->of_node, "my-string", &str);
    dev_info(dev, "my-string = %s\n", str);

    /* 读取数组 */
    int count = of_property_count_u32_elems(dev->of_node, "my-array");
    dev_info(dev, "my-array has %d elements:\n", count);
    for (i = 0; i < count && i < 5; i++) {
        of_property_read_u32_index(dev->of_node, "my-array", i, &arr[i]);
        dev_info(dev, "  [%d] = %u\n", i, arr[i]);
    }

    return 0;
}
```

### 8.4 预期 dmesg

```
hello-device: my-custom-value = 0x1234
hello-device: my-string = dji-bsp
hello-device: my-array has 5 elements:
hello-device:   [0] = 10
hello-device:   [1] = 20
hello-device:   [2] = 30
hello-device:   [3] = 40
hello-device:   [4] = 50
```

---

## 九、实验 4：Ftrace 追踪 driver_probe_device

### 9.1 实验目标

用 Ftrace 追踪从驱动注册到 probe 被调用的完整流程。

### 9.2 操作步骤

```bash
# 板端:
# 设置 function_graph 追踪
echo function_graph | sudo tee /sys/kernel/tracing/current_tracer
echo '*driver_probe*' | sudo tee /sys/kernel/tracing/set_ftrace_filter
echo '*platform*' | sudo tee -a /sys/kernel/tracing/set_ftrace_filter
echo '*really_probe*' | sudo tee -a /sys/kernel/tracing/set_ftrace_filter
echo '*of_match*' | sudo tee -a /sys/kernel/tracing/set_ftrace_filter
echo 128 | sudo tee /sys/kernel/tracing/max_graph_depth

# 清缓冲
echo | sudo tee /sys/kernel/tracing/trace

# 加载驱动 (触发 probe)
sudo insmod /tmp/hello_drv.ko

# 保存追踪
sudo cat /sys/kernel/tracing/trace > /tmp/probe_trace.log
echo nop | sudo tee /sys/kernel/tracing/current_tracer
```

### 9.3 预期调用链

```
platform_driver_register()
  → driver_register()
    → bus_add_driver()
      → driver_attach()
        → bus_for_each_dev()
          → __driver_attach()
            → driver_match_device()       ← of_match_table 匹配
              → of_driver_match_device()
                → of_match_node()
                  → __of_match_node()     ← 遍历 compatible
            → driver_probe_device()       ← 匹配成功, 开始 probe
              → really_probe()
                → platform_driver.probe() ← 调用你的 hello_probe
```

---

## 十、思考题

1. 设备树中的 `compatible` 为什么可以有多个值（如 `"rockchip,rv1126b-uart", "snps,dw-apb-uart"`）？匹配时内核从哪个开始尝试？这种设计有什么好处？

2. `platform_bus` 是一条虚拟总线，它不对应任何物理硬件。为什么 Linux 需要虚拟总线？哪些设备应该挂在 platform_bus 上，哪些不应该？

3. `devm_` 资源管理函数和普通函数相比有什么优势？如果 probe 在 `devm_ioremap_resource` 之后、`devm_request_irq` 之前失败，前者分配的资源会自动释放吗？

4. 板级 DTS（`FET1126B-S.dtsi`）通过 `&i2c0 { status = "okay"; }` 覆盖了 SoC DTS 中的 `status = "disabled"`。如果 SoC DTS 中某节点没有定义 `status`，默认值是什么？

5. 字符设备注册中，`device_create()` 创建的 `/dev/hello` 节点是内核自动创建的，还是需要 udev/mdev 守护进程配合？如果文件系统没有 udev，设备节点会怎样？

---

## 十一、踩坑记录

| 日期 | 问题 | 原因 | 解决方案 |
|------|------|------|----------|
| | insmod 报 "Invalid module format" | 内核版本不匹配或 CONFIG 不一致 | 确保用当前内核源码编译模块 |
| | probe 未被调用 | DTS compatible 字符串不匹配 | 检查 of_match_table 和 DTS compatible 完全一致 |
| | /dev/hello 未创建 | 没有 udev/mdev 或 class_create 失败 | 检查 dmesg，确保 rootfs 有 udev |
| | cdev_add 返回 -EBUSY | 设备号已被占用 | 用 `cat /proc/devices` 检查冲突 |
| | of_property_read_u32 返回 -EINVAL | 属性不存在或类型不对 | DTS 中确认属性名和类型 |
| | 模块编译报找不到头文件 | KERNEL_DIR 路径不对 | 指向 SDK 的 kernel-6.1 目录 |

---

## 十二、下阶段预告

阶段三：**中断处理 + 并发**
- GIC-400 中断控制器深度解析
- `request_irq` / `request_threaded_irq`
- Top half vs Bottom half (workqueue / tasklet / threaded_irq)
- spinlock / mutex / completion 并发原语
- Lockdep / irqsoff tracer 实战调试

---

## 相关笔记

- [[MOC-嵌入式Linux]] — 学习路线总览
- [[bsp-boot-flow]] — 阶段一：Bootloader + 启动流程
- [[kernel-debug-env]] — 附录A：内核调试环境 (Ftrace 使用)
- [[rv1126b]] — RV1126B 运动相机项目
