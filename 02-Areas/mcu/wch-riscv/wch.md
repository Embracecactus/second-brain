---
tags: [wch, risc-v, ch32v208, ch582, kconfig]
category: mcu/wch-riscv
created: 2026-06-09
---
```markdown
---
title: WCH CH32V RISC-V MCU 项目
tags:
  - mcu
  - wch
  - risc-v
  - ch32v208
  - ch582
  - embedded
  - freertos
  - ble
  - usb
category: mcu/wch-riscv
created: 2026-06-09
status: active
---

# WCH CH32V RISC-V MCU 项目

## 项目概述

基于 WCH（南京沁恒）CH32V 系列 RISC-V MCU 的嵌入式开发项目集合，涵盖 CH32V208 和 CH582 两款芯片，实现了 USB 自定义设备、BLE 蓝牙键盘、矩阵键盘扫描、编码器测速等功能。项目采用 Kconfig 配置管理系统（移植自 Zephyr），通过 Makefile 构建，支持 FreeRTOS 实时操作系统。

## 技术栈

| 类别 | 技术 |
|------|------|
| 芯片 | CH32V208 (RISC-V RV32IMAC), CH582 |
| 工具链 | RISC-V Embedded GCC 12 (`riscv-wch-elf-gcc`) |
| 构建系统 | GNU Make + Kconfig (menuconfig) |
| RTOS | FreeRTOS (可选), TMOS (WCH BLE 协议栈调度器) |
| 协议 | USB 2.0 Full-Speed, BLE (HID Profile), UART |
| SDK | WCH CH32V208 SDK (git 存储库), CH582 EVT SDK |
| 版本管理 | Git + Git LFS（编译器大文件） |

## 架构与关键设计

### 目录结构

```
wch/
├── Kconfig                          # 顶层 Kconfig（芯片选择、SDK 路径、FreeRTOS 使能）
├── Makefile                         # 主构建脚本
├── configs/                         # defconfig 预定义配置
├── ch32v208-project-windowsAndlinux/  # CH32V208 完整项目（Windows/Linux 双平台）
│   ├── main/main.c                  # 应用入口
│   ├── app/                         # 应用层模块
│   │   ├── keyboard_scan.*          # 独立按键 + 矩阵键盘扫描
│   │   ├── usb_keyboard.*           # USB HID 键盘
│   │   ├── ble_keyboard.*           # BLE HID 键盘
│   │   └── lcd_128064.*             # LCD 显示驱动
│   ├── drivers/                     # BSP 驱动抽象层
│   ├── sdk/ch32v208/               # WCH 官方 SDK（git clone）
│   ├── components/log/             # 日志组件
│   └── tools/compiler/             # 交叉编译工具链（Git LFS）
├── ch582-sdk/ (ch-project/)        # CH582 BLE SoC 项目
├── ch-p/                           # CH32V208 驱动开发验证目录
│   └── drivers/                    # 独立的 usart/clk 驱动模块
└── wch-project/                    # CH582 项目（含 BSP 抽象层、log 库）
```

### 构建系统设计

项目采用 **Kconfig + Makefile** 的 Linux 内核式配置管理：

1. **Kconfig** 定义配置项（芯片型号、SDK 源地址、外设使能）
2. **menuconfig** 生成 `.config` 文件
3. **update-config** 将 `.config` 转换为 `autoconf.h`（C 宏定义）
4. Makefile 通过 `-include .config` 和 `-include autoconf.h` 自动适配编译参数

### 驱动抽象层（BSP）

采用分层设计，驱动层与 SDK 解耦：
- `drv_clk.c/h` — 时钟初始化抽象
- `drv_usart.c/h` — UART 驱动抽象
- `drv_gpio.c/h` — GPIO 操作抽象
- `drv_usb.c/h` — USB 设备驱动
- `drv_tim.c/h` — 定时器驱动

SDK 路径通过 `CONFIG_SDK_LOCAL_PATH` 配置，支持不同芯片切换。

### 应用架构

应用层基于 **TMOS 事件调度器**（WCH BLE 协议栈内置）和 **FreeRTOS** 两套调度机制：

- **TMOS**：BLE 协议栈事件循环（`TMOS_SystemProcess()`），用于 BLE HID 键盘
- **FreeRTOS**：通用任务调度，用于 USB 键盘、编码器等
- **中断驱动**：定时器中断触发键盘扫描，GPIO EXTI 用于唤醒

## 核心知识点

### 1. USB 设备枚举与描述符

CH32V208 使用内置 USBFS 控制器实现 USB Full-Speed 设备。枚举流程：
1. `USBFS_RCC_Init()` — 使能 USB 时钟
2. `USBFS_Device_Init()` — 复位 SIE 引擎、配置中断、初始化端点、使能上拉电阻
3. 主机枚举：读取 Device/Configuration/Interface/Endpoint/String 描述符
4. 数据传输：通过 `USBFS_Endp_DataUp()` 发送，中断中接收

描述符结构：`Device(18B) -> Configuration(9B) -> Interface(9B) -> Endpoint(7B)x6 -> String`

### 2. USB 端点配置与 DMA

端点方向通过 `UEP4_1_MOD / UEP2_3_MOD / UEP5_6_MOD` 寄存器配置：
- EP1/3/5: OUT (Host->Device, RX)
- EP2/4/6: IN (Device->Host, TX)
- EP0: 控制传输（双向）

DMA 缓冲区需 4 字节对齐，双缓冲模式下 TX 偏移 64 字节。

### 3. BLE HID 键盘初始化流程

```
WCHBLE_Init() → HAL_Init() → GAPRole_PeripheralInit()
  → HidDev_Init() → HidEmu_Init() → GAPRole_StartDevice()
```

关键注意事项：
- FreeRTOS 下中断必须使用**软件压栈**（`__attribute__((interrupt()))`），不能使用硬件压栈（`"WCH-Interrupt-fast"`）
- SysTick 定时器被 FreeRTOS 占用，USB 延迟需改用其他定时器
- Link.ld 的 RAM 大小必须与下载器配置一致

### 4. 键盘扫描与编码器

- **独立按键**：TIM3 中断（10ms 周期）触发 `KB_Scan()`，二次读取消抖
- **矩阵键盘**：5x5 矩阵，逐行拉低扫描列状态
- **旋转编码器**：TIM2 编码器模式，通过 DIR 位和溢出中断计算圈数和速度

### 5. 低功耗与唤醒

通过 EXTI Event 模式（非中断模式）配合 `PWR_EnterSTOPMode(PWR_STOPEntry_WFE)` 实现低功耗，唤醒后需重新初始化系统时钟和 USB。

## 重要代码片段

### USB 数据发送函数（核心接口）

```c
uint8_t USBFS_Endp_DataUp(uint8_t endp, uint8_t *pbuf, uint16_t len, uint8_t mod)
{
    // 1. 检查端点合法性 (EP1-EP7)
    // 2. 检查端点是否空闲 (Endp_Busy)
    // 3. 获取端点模式 (TX_EN/RX_EN)
    // 4. 双缓冲偏移: 同时RX+TX时偏移64字节
    // 5. DMA模式: 直接设置DMA地址; 拷贝模式: memcpy到缓冲区
    // 6. 标记端点忙, 设置发送长度, 响应ACK
}
```

### 键盘扫描状态机

```c
void KB_Scan(void) {
    scan_cnt++;
    if (scan_cnt == FIRST_READ_CNT)   // 第20次: 首次采样
        current_scan_result = read_key_gpio();
    else if (scan_cnt == SECOND_READ_CNT) { // 第40次: 二次采样
        scan_cnt = 0;
        if (current_scan_result == temp_status) { // 消抖成功
            tmos_set_event(usb_keyboard_TaskID, KEY_SCAN_EVENT);
        }
    }
}
```

### Kconfig 配置生成 autoconf.h

```c
// .config → autoconf.h 自动生成
#define CONFIG_CHIP "ch32v208"
#define CONFIG_CHIP_CH32V208 1
#define CONFIG_USER_FREERTOS_ENABLE 1
#define CONFIG_DRV_USART_ENABLE 1
#define CONFIG_DRV_UART_PORT 3
```

## 构建/运行方法

### 环境准备

```bash
# 解压工具链
cd tools/compiler
tar -xvf MRS_Toolchain_Linux_x64_V210.tar.xz
export PATH="$PWD/RISC-V-Embedded-GCC12/bin:$PATH"
riscv-wch-elf-gcc -v
```

### 编译流程

```bash
# 加载 defconfig
make ch32v208_defconfig.config

# 配置（可选）
make menuconfig

# 完整编译
make all

# 清理
make clean
```

### 编译产物

| 文件 | 说明 |
|------|------|
| `build/ch32V208-project.elf` | ELF 可执行文件 |
| `build/ch32V208-project.bin` | 二进制烧录文件 |
| `build/ch32V208-project.hex` | Intel HEX 烧录文件 |
| `build/ch32V208-project.map` | 链接映射文件 |
| `build/ch32V208-project.list` | 反汇编列表 |

### 编译架构参数

```
-march=rv32imac -mabi=ilp32 -mcmodel=medany
-msmall-data-limit=8 -mno-save-restore
--specs=nano.specs --specs=nosys.specs
```

## 相关笔记链接

- [[RISC-V 嵌入式开发基础]]
- [[USB 协议与设备描述符]]
- [[BLE HID 协议栈]]
- [[FreeRTOS RISC-V 移植]]
- [[Kconfig 配置系统]]
- [[嵌入式 Makefile 构建系统]]
- [[WCH CH32V208 数据手册]]
- [[旋转编码器接口设计]]
- [[矩阵键盘扫描算法]]
```

## 相关笔记

- [[xingshan]] — BS2X CFBB 嵌入式项目（同为 RISC-V MCU）
- [[wenan]] — 嵌入式学习笔记合集（含 CH32V208 开发）
- [[fpga]] — FPGA LED 流水灯项目（同为数字逻辑入门）
