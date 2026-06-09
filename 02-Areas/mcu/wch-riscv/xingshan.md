---
tags:
  - mcu
  - riscv
  - wch
  - bs2x
  - bs21e
  - cfbb
  - deveco
  - liteos
  - sle
  - bluetooth
  - nfc
  - carkey
  - smart-home
category: mcu/wch-riscv
created: 2026-06-09
status: active
project: xingshan
sdk_version: BS2X_1.0.16
cfbb_version: 0.9.0.5
chip: BS21E (BS21-N1100E)
board: BS21-N1100E-STANDARD
---

# xingshan - BS2X CFBB 嵌入式项目

## 项目概述

xingshan 是一个基于海思 (HiSilicon) BS2X 系列 RISC-V MCU 的嵌入式项目，使用 CFBB (Connectivity Framework Building Block) SDK v1.0.16 开发。该项目是一个智能车钥匙 (CARKEY) 应用，通过 SLE (星闪近距连接) 协议实现 BLE 设备扫描、白名单管理、测距定位等功能，结合 CMT2300A 射频模块和外设驱动，运行在 LiteOS 实时操作系统上。

## 技术栈

- **芯片**: 海思 BS21E (RISC-V 架构, BS21-N1100E 模组)
- **SDK**: CFBB (Connectivity Framework Building Block) v0.9.0.5, SDK 版本 BS2X 1.0.16
- **RTOS**: LiteOS (华为轻量级实时操作系统)
- **构建系统**: CMake + Python build.py + Ninja/Kconfig
- **IDE**: DevEco Device Tool (华为设备开发工具)
- **工具链**: RISC-V GCC (`gcc_riscv32_win_env`) / RISC-V CC (`cc_riscv32_win_env`)
- **协议栈**: BLE, SLE (星闪近距连接), NFC, Wi-Fi, Cat1/LTE
- **外设驱动**: GPIO, SPI, I2C, UART, PWM, ADC, DMA, Timer, Watchdog, QDEC, KeyScan, USB, CAN-FD, I2S/PCM, PSRAM, SFC (Flash)
- **安全**: GmSSL 3.0, TRNG, PKE (公钥引擎), Cipher, KM (密钥管理), OTP/EFUSE
- **调试工具**: JLink, BurnTool (烧录), HSO (调试数据库)
- **开发环境**: Python 3.8.10, CMake 3.18.4, Ninja 1.9.0, ccache 4.7.4, kconfiglib 14.1.0

## 架构与关键设计

### 分层架构

项目采用典型的嵌入式分层架构：

```
application/     -- 应用层 (demo, samples, 车钥匙应用)
  ├── bs20/      -- BS20 芯片应用入口
  ├── bs21/      -- BS21 芯片应用入口
  ├── bs21e/     -- BS21E 芯片应用入口 (main.c)
  ├── bs22/      -- BS22 芯片应用入口
  ├── demo/      -- 演示代码 (CMT2300A, SPI, Flash, Timer)
  ├── samples/   -- 官方示例 (BLE, SLE, NFC, 外设)
  └── ux/        -- UX 通信层
protocol/        -- 协议层
  ├── bt/        -- 蓝牙 (controller/host/algorithm/posalg)
  ├── nfc/       -- NFC
  ├── wifi/      -- Wi-Fi
  ├── cat1/      -- Cat1/LTE
  └── slp/       -- SLP 窄带协议
kernel/          -- 内核层
  ├── liteos/    -- LiteOS 内核
  ├── non_os/    -- 非 OS 模式支持
  ├── osal/      -- 操作系统抽象层
  └── osal_adapt/ -- OSAL 适配层
drivers/         -- 驱动层
  ├── chips/     -- 芯片级驱动 (bs20/bs21/bs21e/bs22/bs2x)
  ├── boards/    -- 板级驱动
  └── drivers/   -- 通用驱动 (HAL/Driver)
middleware/      -- 中间件
  ├── chips/     -- 芯片中间件 (NV/分区/镜像信息)
  ├── services/  -- 服务 (WiFi/TIoT Host)
  └── utils/     -- 工具库 (AT/DFX/PM/USB/Codec/算法)
open_source/     -- 开源组件 (GmSSL 3.0)
```

### 多芯片支持

SDK 通过条件编译和目录隔离支持多款芯片：BS20, BS21, BS21E, BS22。每个芯片有独立的 application 入口、驱动、中间件和板级配置。

### Kconfig 配置系统

使用标准 Kconfig 进行系统配置，支持以下关键选项：
- `SYSTEM_CONTROL_ENABLE` -- 电源管理 (LDO/BUCK, WFI 降频, 超深睡眠)
- `SAMPLE_ENABLE` -- 启用示例代码
- `SYSTEM_MOUSE_PIN_CONFIG` -- 鼠标引脚配置
- `DRIVER_SUPPORT_LITEOS` -- 启用 LiteOS 内核

### 启动流程

```
Flashboot Init -> Jump to 0x90115300 -> main_init() -> main() -> axk_main()
```

`main()` 函数位于 `application/bs21e/standard/main.c`，调用 `main_init()` 进行芯片初始化，随后通过 LiteOS 调度器启动多任务。

### 应用设计模式 (车钥匙)

应用使用多任务架构：

```c
// axk_main() 中创建三个任务
task1: sle_test_task1  -- LED 闪烁 (GPIO 控制)
task2: sle_test_task2  -- UART 命令解析 (StandbyRxData/ParseCommand)
task3: sle_test_task3  -- 测距功能 (SLE Measure Distance)
```

核心功能流程：
1. GPIO 初始化 -- 配置 RF 数据引脚、LED、按键
2. 定时器初始化
3. CMT2300A 射频模块初始化 (SPI)
4. SLE 扫描 -- 发现周边设备
5. 白名单过滤 -- 按设备名 "yk" 和 RSSI 阈值过滤
6. SLE 连接 -- 连接到白名单设备
7. 测距 -- 通过 SLE RSSI 计算距离 (0.076~0.330m 范围)

## 关键代码片段

### OSAL 任务创建模式

```c
osal_kthread_lock();
task_handle = osal_kthread_create((osal_kthread_handler)sle_test_task1,
                                  0, "sle_test_task1",
                                  SLE_TEST1_TASK_STACK_SIZE);
if (task_handle != NULL) {
    osal_kthread_set_priority(task_handle, SLE_TEST1_TASK_PRIO);
    osal_kfree(task_handle);
}
osal_kthread_unlock();
```

### 寄存器操作宏 (common_def.h)

```c
// 通用寄存器位操作
#define uapi_reg_setbits(addr, pos, bits, val) \
    (uapi_reg_read_val32(addr) = \
     (uapi_reg_read_val32(addr) & (~((((unsigned int)1 << (bits)) - 1) << (pos)))) | \
     ((unsigned int)((val) & (((unsigned int)1 << (bits)) - 1)) << (pos)))
```

### 错误码体系 (errcode.h)

SDK 使用 32 位错误码，按模块分段：
- `0x80000000-0x8000007F` -- 通用错误
- `0x80001000-0x80003000` -- 驱动错误 (GPIO/UART/SPI/I2C/DMA/ADC 等)
- `0x80003000-0x80004000` -- 中间件错误 (NV/Flash/PM/DIAG 等)
- `0x80004000-0x80005000` -- Wi-Fi 错误
- `0x80005800-0x80006000` -- NFC 错误
- `0x80006000-0x80007000` -- BT 错误

### OSAL 兼容层

OSAL (操作系统抽象层) 同时支持 Linux 内核模块和 LiteOS：
- `__KERNEL__` 宏区分 Linux 内核环境
- 非内核环境下，模块导出/初始化宏为空操作
- 统一使用 `osal_task`, `osal_kthread_create`, `osal_msleep` 等 API

## 构建与烧录

### 构建命令

```bash
# 使用 DevEco Device Tool (推荐)
hos run --project-dir <project_path> --environment BS21-N1100E-STANDARD

# 使用 Python 脚本直接构建
python build.py <target_name>        # 编译
python build.py -c <target_name>     # clean 后编译
python build.py -j8 <target_name>    # 指定线程数
python build.py -ninja <target_name> # 使用 Ninja 生成器
python build.py -def=XXX,YYY <target_name>  # 添加编译宏
```

### 构建产物

- `.elf` 文件 -- 可执行文件
- `.bin` 文件 -- 固件二进制 (通过 objcopy 生成)
- `_rom.bin` -- ROM 镜像 (如有 ROM_COMPONENT)
- `BOOTLOADERFlsDrv.signed.s19` -- Flash 驱动 (SFC 独立模式)
- HSO 数据库 -- 调试信息

### 烧录工具

- **BurnTool**: Windows 烧录工具 (`DevTools_CFBB_V1.0.7/BurnTool/BurnTool.exe`)
- **JLink**: 支持 JLink 调试烧录 (需配置 `ConnectCore1.JLinkScript`)
- **DevEco Upload**: IDE 内一键烧录

### 调试

- 串口日志: 115200 波特率
- 异常信息: 包含寄存器 dump、任务栈信息、backtrace、CPU Trace
- 内存统计: 堆使用量、任务栈峰值使用率

## 关键学习与洞察

1. **SLE (星闪近距连接)** 是该项目的核心无线协议，用于设备发现、连接和测距，是华为自研的近距通信技术，类似 BLE 但针对中国标准优化。

2. **白名单机制** 实现了设备过滤：通过 NV 存储持久化白名单 (最多 20 个设备)，扫描时按设备名 "yk" 前缀和 RSSI 阈值进行过滤，按下按键进入 10 秒配对模式。

3. **测距功能** 基于 SLE RSSI 实现，精度在 0.076~0.330m 范围内，用于车钥匙的接近检测场景。

4. **异常处理** 系统包含完整的 crash dump：寄存器状态 (mepc/mcause/mtval)、任务栈信息、backtrace、CPU Trace，便于问题定位。

5. **NV (非易失性存储)** 用于存储白名单、学习位置等持久化数据，错误码 `0x80003081` 表示 key 未找到 (首次启动正常)。

6. **CMT2300A** 是外挂的 Sub-GHz 射频收发器，通过 SPI 接口控制，用于额外的无线通信通道。

7. **多芯片统一代码库** 的设计使得同一套 SDK 可以支持 BS20/BS21/BS21E/BS22 多款芯片，通过 `CHIP` 变量和条件编译区分。

8. **内存管理** 使用 LiteOS 堆管理，任务栈大小需要仔细规划 (1024~6144 bytes)，系统总堆约 53KB (`0xcf34`)。

## 相关文档

- `BS2XV100 IDE工具 使用指南_01.pdf` -- IDE 使用指南
- `BS2XV100 NV存储 用户指南.html` -- NV 存储说明
- `DevTools_CFBB_V1.0.7/Toolkit Usage Guide.pdf` -- 工具链使用指南
- `application/samples/nfc/README.md` -- NFC 示例说明

## 相关概念链接

- [[MCU开发]]
- [[RISC-V架构]]
- [[LiteOS]]
- [[BLE蓝牙开发]]
- [[SLE星闪协议]]
- [[NFC近场通信]]
- [[嵌入式C语言]]
- [[Kconfig配置系统]]
- [[CMake构建系统]]
- [[华为DevEco]]

## 相关笔记

- [[wch]] — WCH CH32V RISC-V MCU 项目
- [[wenan]] — 嵌入式学习笔记合集
- [[TuyaOpen]] — 涂鸦 AI+IoT 开发框架（同为 IoT 平台）
