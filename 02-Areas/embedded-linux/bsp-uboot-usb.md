---
tags:
  - embedded-linux
  - bsp
  - bootloader
  - u-boot
  - usb
  - rockusb
  - dwc3
  - gadget
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

# U-Boot USB 子系统深度解析

> **前置笔记**：[[bsp-boot-flow]] — RockUSB 下载模式
>
> **前置笔记**：[[bsp-uboot-rktools]] — rkdeveloptool 烧录流程
>
> **核心文件**：`cmd/rockusb.c`, `drivers/usb/gadget/f_rockusb.c`, `drivers/usb/dwc3/`

---

## 一、RV1126B USB 硬件

| 控制器 | DTS node | 功能 |
|--------|---------|------|
| DWC3 0 | `usbdrd@21500000` | OTG (Host/Device 切换) |
| EHCI | `usb_host0_ehci@...` | USB 2.0 Host |
| OHCI | `usb_host0_ohci@...` | USB 2.0 Host (低功耗) |

U-Boot 烧录场景使用 **DWC3 的 Device (Gadget) 模式**。

---

## 二、U-Boot USB 双栈

```
USB Host 栈 (主机模式)           USB Gadget 栈 (设备模式)
───────────────────────────────────────────────────
UCLASS_USB_HUB                 UCLASS_USB_GADGET
  ├─ UCLASS_USB_DEVICE           ├─ UCLASS_USB_GADGET
  │   USB 存储/键盘              │   f_mass_storage (UMS)
  │                              │   f_rockusb (Rockchip 协议)
  ├─ ehci / ohci / xhci          │
  │   (主机控制器)                └─ dwc3 (双角色控制器)
  └─ dwc3 (OTG 模式)
```

### 2.1 RockUSB 协议

RockUSB 是 Rockchip 定义的私有 USB 烧录协议：

```
USB 枚举阶段:
  VID:PID = 0x2207:0x110f  (Rockchip gadget)
  厂商: "Rockchip"
  产品: "USB Download"

协议层 (基于 USB Bulk 传输):
  ┌─────────┬──────────┬──────────┐
  │ 命令头   │ 数据段   │ 校验段   │
  │ (512B)  │ (可变)   │ (可选)   │
  └─────────┴──────────┴──────────┘

支持的命令:
  CMD_ID    0x01     识别设备信息
  CMD_DL    0x02     下载数据到内存
  CMD_UL    0x03     从内存上传数据
  CMD_RUN   0x04     跳转到内存地址执行
  CMD_WL    0x05     写入 eMMC LBA
  CMD_RL    0x06     读取 eMMC LBA
  CMD_GPT   0x07     写入 GPT 分区表
  CMD_OTP   0x08     OTP 操作
  CMD_RST   0x09     重启设备
```

### 2.2 U-Boot RockUSB 命令

```bash
# 进入 RockUSB 设备模式 (最常用):
=> rockusb 0 0x20000000 0x20000000
# 参数: (USB 控制器索引) (内存基址) (内存大小)

# 此时 PC 端检测到设备:
lsusb | grep Rockchip
# Bus 001 Device 003: ID 2207:110f Rockchip

# PC 端即可用 rkdeveloptool:
rkdeveloptool db loader.bin
rkdeveloptool wl 0x40 uboot.img
```

### 2.3 UMS (USB Mass Storage)

```bash
# 将 eMMC 分区暴露为 U 盘 (给 PC 直接访问)
=> ums 0 mmc 0
# 参数: (USB 控制器) (存储类型) (存储设备)
# 此时 PC 端可以看到 eMMC 作为 U 盘挂载
# 适合: 生产测试, 日志导出
```

---

## 三、DWC3 控制器

### 3.1 DWC3 核心特性

```
USB 3.0 SuperSpeed (5Gbps)
USB 2.0 HighSpeed (480Mbps)
双角色: Host + Device 切换
32 个双向端点
支持: Control/Bulk/Interrupt/Isochronous
```

### 3.2 RV1126B DTS 配置

```dts
&usbdrd {
    status = "okay";
    dr_mode = "otg";                    // OTG 双角色
    extcon = <&usb2phy0>;               // ID 引脚检测 (Host/Device 切换)
    pinctrl-names = "default";
    pinctrl-0 = <&usb_drd>, <&usb_vcc>;

    // USB 2.0 PHY (芯动科技 USB2PHY)
    usb2-phy = <&usb2phy0>;
};

&usb2phy0 {
    status = "okay";
    phy-supply = <&vcc5v0_usb>;         // USB VBUS 电源
};
```

---

## 四、USB 启动流程

```
上电 → Maskrom → 检测 USB ID 引脚
  │
  ├─ ID 为低 (Host 模式) → 从 eMMC/SD 正常启动
  │
  └─ ID 为高 (Device 模式) → 进入 USB 下载模式
       │
       ├─ BootROM 枚举为 "Rockchip USB Download"
       │    (VID:PID = 0x2207:0x301a, Maskrom 状态)
       │
       ├─ PC 端: rkdeveloptool db loader.bin
       │    → 下载 loader 到 ISRAM 并执行
       │    → 设备切换到 Loader 模式
       │    → VID:PID 变为 0x2207:0x110f
       │
       └─ PC 端: rkdeveloptool wl 0x40 uboot.img
            → 通过 RockUSB 协议写入 eMMC
```

---

## 五、U-Boot USB Gadget 调试

```bash
# 查看 USB 控制器状态
=> usb start                     # 启动 USB 子系统 (Host 模式)
=> usb tree                      # 查看 USB 设备树
=> usb info                      # USB 设备信息
=> usb reset                     # 复位 USB 总线

# 测试 RockUSB 连接 (确保 gadget 可用)
=> rockusb 0 0x20000000 0x20000000
# 如果失败: "No USB gadget available" → 检查 DTS dr_mode

# 用 USB 下载模式调试
# U-Boot shell 中进入下载模式, 然后:
# PC: rkdeveloptool rfi       → 读取 Flash 信息
# PC: rkdeveloptool pt        → 读取分区表
```

---

## 六、思考题

1. **RockUSB vs UMS**：两种 USB 模式分别适用于什么场景？为什么 Rockchip 要自研 RockUSB 协议而不是只用标准 UMS？

2. **OTG 检测**：RV1126B 通过什么机制检测 USB ID 引脚决定 Host/Device 模式？如果 ID 引脚悬空会怎样？

3. **DWC3 端点管理**：RockUSB 使用几个 Bulk 端点？为什么需要至少 2 个端点（IN + OUT）？

---

## 相关笔记

- [[bsp-boot-flow]] — RockUSB 下载模式
- [[bsp-uboot-rktools]] — rkdeveloptool 烧录
- [[bsp-uboot-boottime]] — USB 初始化时间
- [[MOC-嵌入式Linux]] — 嵌入式 Linux 学习路线 MOC
