---
title: STM32G0B1 Makefile 工程模板
tags:
  - stm32
  - stm32g0
  - mcu
  - embedded
  - bare-metal
  - hal
  - makefile
  - arm-cortex-m0plus
category: mcu/stm32
created: 2026-06-09
status: active
target_mcu: STM32G0B1RET6
board: NUCLEO-G0B1RE
toolchain: GCC (arm-none-eabi)
---

# STM32G0B1 Makefile 工程模板

## 项目概述

基于 STM32CubeMX 生成的 STM32G0B1RET6 Makefile 工程，目标平台为 NUCLEO-G0B1RE 开发板。工程使用 STM32 HAL 库进行裸机开发，通过 USART2 实现 printf 重定向输出，是一个标准的 STM32 裸机项目起点模板。

## 技术栈

| 类别 | 详情 |
|------|------|
| MCU | STM32G0B1RET6 (Cortex-M0+, 512KB Flash, 144KB RAM) |
| 开发板 | NUCLEO-G0B1RE |
| HAL 库版本 | STM32Cube FW_G0 V1.6.2 |
| 代码生成器 | STM32CubeMX v6.14.1, projectgenerator v4.6.0.1-B1 |
| 工具链 | arm-none-eabi-gcc (GCC) |
| 构建系统 | Makefile |
| 调试接口 | SWD (Serial Wire Debug) |
| 串口 | USART2 @ 115200, 8N1 (PA2-TX, PA3-RX) |
| 时钟源 | HSI 16MHz (内部高速振荡器), PLL 可达 64MHz |

## 项目结构

```
stm32g0-make/
├── Core/
│   ├── Inc/                    # 头文件
│   │   ├── main.h              # 主程序头文件, 引脚定义
│   │   ├── gpio.h
│   │   ├── usart.h
│   │   ├── stm32g0xx_it.h
│   │   └── stm32g0xx_hal_conf.h
│   └── Src/                    # 源文件
│       ├── main.c              # 主程序入口
│       ├── gpio.c              # GPIO 初始化
│       ├── usart.c             # USART2 初始化
│       ├── stm32g0xx_it.c      # 中断服务函数
│       ├── stm32g0xx_hal_msp.c # HAL MSP 初始化
│       ├── system_stm32g0xx.c  # 系统初始化
│       ├── sysmem.c            # 内存管理
│       └── syscalls.c          # 系统调用桩函数
├── Drivers/
│   ├── STM32G0xx_HAL_Driver/   # HAL 驱动库
│   └── CMSIS/                  # ARM CMSIS 核心支持
├── Makefile                    # 构建脚本
├── STM32G0B1XX_FLASH.ld       # 链接脚本
├── startup_stm32g0b1xx.s       # 启动汇编文件
├── stm32g0-make.ioc            # STM32CubeMX 工程文件
└── build/                      # 编译输出目录
```

## 架构与关键设计决策

### 1. 系统时钟配置

使用内部 HSI 振荡器 (16MHz) 作为系统时钟源，未启用 PLL 和外部晶振。这是一种简化配置，适合快速原型验证。

```c
// SystemClock_Config 核心配置
RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSI;
RCC_OscInitStruct.HSIState = RCC_HSI_ON;
RCC_OscInitStruct.HSIDiv = RCC_HSI_DIV1;
RCC_OscInitStruct.PLL.PLLState = RCC_PLL_NONE;  // PLL 未使用
RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_HSI;  // SYSCLK = 16MHz
```

### 2. printf 重定向到 UART

通过重写 `_write` 系统调用实现 printf 输出到 USART2，这是嵌入式开发中非常常见的模式：

```c
// 重写 _write 使 printf 输出到串口
int _write(int fd, char *ch, int len)
{
  HAL_UART_Transmit(&huart2, (uint8_t*)ch, len, 0xFFFF);
  return len;
}
```

### 3. GPIO 外设配置

| 引脚 | 功能 | 模式 | 说明 |
|------|------|------|------|
| PA5 | LED_GREEN | GPIO_Output, Push-Pull | 板载绿色 LED |
| PC13 | B1 | EXTI Rising Edge | 蓝色用户按钮 (外部中断) |
| PA2 | USART2_TX | AF1 | 串口发送 (STLK_TX) |
| PA3 | USART2_RX | AF1 | 串口接收 (STLK_RX) |
| PA13 | SYS_SWDIO | Serial Wire | SWD 调试数据 |
| PA14 | SYS_SWCLK | Serial Wire | SWD 调试时钟 |
| PF0 | RCC_OSC_IN | HSE External Clock | 外部时钟输入 (MCO) |

### 4. 内存布局

| 区域 | 起始地址 | 大小 | 用途 |
|------|----------|------|------|
| FLASH | 0x08000000 | 512KB | 程序代码 + 常量数据 |
| RAM | 0x20000000 | 144KB | 数据段 + BSS + 堆栈 |

堆栈配置:
- 最小堆空间 (Heap): 0x200 = 512 bytes
- 最小栈空间 (Stack): 0x400 = 1024 bytes

### 5. HAL 库裁剪

工程仅链接了必要的 HAL 模块，遵循最小化原则:
- HAL 核心 (`stm32g0xx_hal.c`)
- UART 驱动 (`stm32g0xx_hal_uart.c`, `*_ex.c`)
- RCC 时钟 (`stm32g0xx_hal_rcc.c`, `*_ex.c`, `stm32g0xx_ll_rcc.c`)
- GPIO (`stm32g0xx_hal_gpio.c`)
- DMA (`stm32g0xx_hal_dma.c`, `*_ex.c`)
- Flash (`stm32g0xx_hal_flash.c`, `*_ex.c`)
- PWR (`stm32g0xx_hal_pwr.c`, `*_ex.c`)
- Cortex (`stm32g0xx_hal_cortex.c`)
- EXTI (`stm32g0xx_hal_exti.c`)

## 关键学习与洞察

1. **STM32CubeMX 生成的 USER CODE 区域**: 代码中大量 `/* USER CODE BEGIN */` / `/* USER CODE END */` 标记是 CubeMX 的保护区机制。在这些区域内的代码在重新生成工程时会被保留，区域外的代码会被覆盖。自定义代码必须写在这些标记之间。

2. **_write 桩函数**: nano.specs 链接选项使用 newlib-nano，printf 底层调用 `_write`。重写此函数是嵌入式 printf 重定向的标准做法，比直接用 `HAL_UART_Transmit` 更灵活，允许直接使用 `printf`/`snprintf` 等标准库函数。

3. **Cortex-M0+ 无 FPU**: Makefile 中 `FPU` 和 `FLOAT-ABI` 变量为空，因为 Cortex-M0+ 不含硬件浮点单元。所有浮点运算由软件模拟，性能有限。

4. **错误处理模式**: `Error_Handler()` 中直接 `__disable_irq()` 并进入死循环，这是 HAL 库的标准错误处理模式。在生产代码中应考虑加入看门狗复位或 LED 故障指示。

5. **启动流程**: `startup_stm32g0b1xx.s` -> `Reset_Handler` -> `SystemInit` -> `main` -> `HAL_Init` -> `SystemClock_Config` -> 外设初始化 -> 主循环。

## 构建与运行

### 前置条件

- 安装 `arm-none-eabi-gcc` 工具链
- 安装 `make`
- (可选) STM32CubeProgrammer 用于烧录

### 编译

```bash
cd stm32g0-make
make clean
make all
```

编译产物在 `build/` 目录:
- `stm32g0-make.elf` - ELF 可执行文件
- `stm32g0-make.hex` - Intel HEX 格式
- `stm32g0-make.bin` - 二进制格式

### 烧录

```bash
# 使用 ST-Link (SWD)
STM32_Programmer_CLI -c port=SWD -w build/stm32g0-make.bin 0x08000000 -rst
```

### 调试

```bash
# 使用 GDB + OpenOCD
arm-none-eabi-gdb build/stm32g0-make.elf
(gdb) target remote :3333
(gdb) load
(gdb) continue
```

### 串口监控

```bash
# Linux
screen /dev/ttyACM0 115200
# 或
minicom -D /dev/ttyACM0 -b 115200
```

## 相关概念链接

- [[STM32 HAL 库]]
- [[STM32CubeMX 工程配置]]
- [[ARM Cortex-M0+ 架构]]
- [[嵌入式 printf 重定向]]
- [[STM32 时钟树配置]]
- [[SWD 调试接口]]
- [[STM32G0 系列产品]]
- [[GCC 交叉编译工具链]]
- [[嵌入式链接脚本 LD]]
- [[STM32 启动流程]]
- [[嵌入式 GPIO 配置]]
- [[UART 串口通信]]

## 参考资料

- [STM32G0B1RE 官方数据手册](https://www.st.com/en/microcontrollers-microprocessors/stm32g0b1re.html)
- [STM32Cube FW_G0 固件包](https://github.com/STMicroelectronics/STM32CubeG0)
- [NUCLEO-G0B1RE 开发板用户手册](https://www.st.com/resource/en/user_manual/um2398-stm32g0-nucleo-64-board-mb1360-stmicroelectronics.pdf)
- [STM32CubeMX 用户指南](https://www.st.com/resource/en/user_manual/um1718-stm32cubmx-for-stm32-configuration-and-initialization-c-code-generation-stmicroelectronics.pdf)

## 相关笔记

- [[studyzephyr]] — Zephyr RTOS 学习项目（同为 STM32 Nucleo 平台）
- [[make]] — Make 构建系统学习
- [[embedded-blog]] — 嵌入式技术博客（含 STM32 GPIO 文章）
- [[wenan]] — 嵌入式学习笔记合集