---
tags:
  - temperature
  - sensor
  - adc
  - spi
  - rtd
  - pt100
  - pt1000
  - linux-driver
  - orangepi
category: mcu/common
created: 2026-06-09
---

# MAX31865 RTD温度传感器驱动开发笔记

## 项目概述

本项目基于 **LibDriver MAX31865** 开源驱动库 (v2.0, MIT License, Author: Shifeng Li)，实现了对 Maxim Integrated MAX31865 RTD-to-Digital 芯片的完整驱动。MAX31865 是一款高精度 RTD (Resistance Temperature Detector) 数字转换器，通过 SPI 接口与主控通信，支持 PT100/PT1000 铂电阻温度传感器，适用于工业级高精度温度测量场景。

目标硬件平台: **Orange Pi PC Plus** (Allwinner H3 SoC)

SPI 引脚连接: SCLK=PC2, MOSI=PC0, MISO=PC1, CS=PC3

---

## 关键知识点

### 1. MAX31865 芯片特性

| 参数 | 值 |
|---|---|
| 芯片全称 | Maxim Integrated MAX31865 |
| 通信接口 | SPI |
| 供电电压 | 3.0V ~ 3.6V |
| 最大电流 | 5.75mA |
| 工作温度范围 | -40.0C ~ 85.0C |
| 驱动版本 | 2.0 |

### 2. RTD 传感器类型

- **PT100**: 0度时电阻为 100 ohm, 适合一般精度场景
- **PT1000**: 0度时电阻为 1000 ohm, 灵敏度更高, 适合远距离传输

### 3. 接线方式 (Wire Configuration)

- **2-wire**: 最简单, 精度最低, 无导线补偿
- **3-wire**: 中等精度, 可补偿导线电阻
- **4-wire**: 最高精度, 完全消除导线电阻影响 (推荐)

### 4. 温度转换算法

使用 Callendar-Van Dusen 方程进行 RTD 电阻到温度的转换:

- **正温度 (>=0C)**: 使用二次方程求解, 涉及系数 RTD_A = 3.9083e-3, RTD_B = -5.775e-7
- **负温度 (<0C)**: 使用五阶多项式拟合, 系数从 -242.02 到 1.5243e-10

### 5. 故障检测机制

MAX31865 内置多种故障检测:
- RTD High/Low Threshold 超限检测
- REFIN > 0.85 * VBIAS (参考电阻过高)
- REFIN < 0.85 * VBIAS (参考电阻过低/断路)
- RTDIN < 0.85 * VBIAS (传感器断路)
- Over/Under Voltage 过欠压检测

---

## 技术细节

### 驱动架构分层

```
+---------------------------+
|     main.c (CLI入口)      |
+---------------------------+
|  example/basic + shot     |  <-- 应用示例层
+---------------------------+
|  driver_max31865.c/h      |  <-- 核心驱动层
+---------------------------+
|  interface (SPI HAL)      |  <-- 硬件抽象层
+---------------------------+
|  Linux SPI / IIO 子系统   |  <-- 操作系统层
+---------------------------+
```

### 核心数据结构

**Handle 结构体** (`max31865_handle_t`):
- `spi_init` / `spi_deinit`: SPI 总线初始化/反初始化函数指针
- `spi_read` / `spi_write`: SPI 读写函数指针 (reg, buf, len)
- `delay_ms`: 毫秒延时函数指针
- `debug_print`: 调试打印函数指针
- `inited`: 初始化标志
- `resistor`: PT 电阻类型 (100/1000)
- `ref_resistor`: 参考电阻值 (float)

**Info 结构体** (`max31865_info_t`):
- 芯片名称、厂商、接口、电压范围、电流、温度范围、驱动版本

### 寄存器映射

| 地址 | 名称 | 说明 |
|---|---|---|
| 0x00 | CONFIG | 配置寄存器 (VBIAS/Mode/Shot/Wire/Fault/Clear/Filter) |
| 0x01 | RTD_MSB | RTD 数据高字节 |
| 0x02 | RTD_LSB | RTD 数据低字节 |
| 0x03 | HIGH_FAULT_MSB | 高阈值高字节 |
| 0x04 | HIGH_FAULT_LSB | 高阈值低字节 |
| 0x05 | LOW_FAULT_MSB | 低阈值高字节 |
| 0x06 | LOW_FAULT_LSB | 低阈值低字节 |
| 0x07 | FAULT_STATUS | 故障状态寄存器 |

### CONFIG 寄存器位域

| Bit | 功能 |
|---|---|
| Bit 7 | VBIAS (0=off, 1=on) |
| Bit 6 | Conversion Mode (0=normally off, 1=auto) |
| Bit 5 | 1-shot (写1触发单次转换, 完成后自动清零) |
| Bit 4 | Wire (0=2/4-wire, 1=3-wire) |
| Bit 3:2 | Fault Detection Cycle Control |
| Bit 1 | Fault Status Clear |
| Bit 0 | Filter Select (0=60Hz, 1=50Hz) |

### SPI 写操作

写寄存器时地址需 OR 0x80 (WRITE_ADDRESS_MASK), 即最高位置1表示写操作。

### 读取流程 (Single Read)

1. 读取 CONFIG 寄存器
2. 设置 Fault Detection Clear, 启用 VBIAS, 设置 One-shot 模式
3. 写回 CONFIG 寄存器
4. 轮询等待 Bit 5 清零 (每次延时 63ms, 最多重试 5000 次)
5. 读取 RTD_MSB 和 RTD_LSB (2 bytes)
6. 检查 Bit 0 故障标志
7. 右移 1 位得到 15-bit raw ADC 值
8. 调用温度转换函数

### 读取流程 (Continuous Read)

1. 启动: 设置 VBIAS=1, Auto Mode=1, Shot=0
2. 直接读取 RTD 寄存器即可
3. 停止: 设置 VBIAS=0, Auto Mode=0, Shot=0

### 参考电阻

参考电阻值 (ref_resistor) 用于将 ADC raw 值转换为实际电阻值。默认值 430.0 ohm。计算公式:
```
rt = (raw / 32768) * ref_resistor
```

---

## 代码/配置片段

### 初始化流程 (Basic Example)

```c
// 1. 链接函数指针
DRIVER_MAX31865_LINK_INIT(&gs_handle, max31865_handle_t);
DRIVER_MAX31865_LINK_SPI_INIT(&gs_handle, max31865_interface_spi_init);
DRIVER_MAX31865_LINK_SPI_DEINIT(&gs_handle, max31865_interface_spi_deinit);
DRIVER_MAX31865_LINK_SPI_READ(&gs_handle, max31865_interface_spi_read);
DRIVER_MAX31865_LINK_SPI_WRITE(&gs_handle, max31865_interface_spi_write);
DRIVER_MAX31865_LINK_DELAY_MS(&gs_handle, max31865_interface_delay_ms);
DRIVER_MAX31865_LINK_DEBUG_PRINT(&gs_handle, max31865_interface_debug_print);

// 2. 初始化芯片
max31865_init(&gs_handle);

// 3. 配置参数
max31865_set_filter_select(&gs_handle, MAX31865_FILTER_SELECT_50HZ);
max31865_set_wire(&gs_handle, MAX31865_WIRE_4);
max31865_set_resistor(&gs_handle, MAX31865_RESISTOR_100PT);
max31865_set_reference_resistor(&gs_handle, 430.0f);
max31865_set_fault_detection_cycle_control(&gs_handle, MAX31865_FAULT_DETECTION_CYCLE_CONTROL_AUTOMATIC_DELAY);
max31865_set_high_fault_threshold(&gs_handle, 0xFFFEU);
max31865_set_low_fault_threshold(&gs_handle, 0x0000U);

// 4. 启动连续读取
max31865_start_continuous_read(&gs_handle);

// 5. 读取温度
float temp;
max31865_continuous_read(&gs_handle, &raw, &temp);

// 6. 停止并反初始化
max31865_stop_continuous_read(&gs_handle);
max31865_deinit(&gs_handle);
```

### IIO (Industrial I/O) 方式读取

通过 Linux IIO 子系统读取 (适用于内核已加载 MAX31865 驱动的情况):

```c
#include <fcntl.h>
#include <unistd.h>

int fd = open("/sys/bus/iio/devices/iio:device0/in_temp_raw", O_RDONLY);
char buffer[16];
read(fd, buffer, sizeof(buffer) - 1);
int raw_value = atoi(buffer);
close(fd);

// 手动转换: rtd_nominal=100, ref_resistor=430
float temperature = a_max31865_temperature_conversion((float)raw_value, 100.0f, 430.0f);
```

### 温度转换核心算法

```c
#define RTD_A  3.9083e-3f
#define RTD_B  -5.775e-7f

static float a_max31865_temperature_conversion(float rt, float rtd_nominal, float ref_resistor)
{
    float z1, z2, z3, z4, temp;
    float rpoly;

    rt /= 32768;                        // ADC raw -> 比例
    rt *= ref_resistor;                 // 比例 -> 实际电阻
    z1 = -RTD_A;
    z2 = RTD_A * RTD_A - (4 * RTD_B);
    z3 = (4 * RTD_B) / rtd_nominal;
    z4 = 2 * RTD_B;
    temp = z2 + (z3 * rt);
    temp = (sqrtf(temp) + z1) / z4;

    if (temp >= 0) return temp;         // 正温度直接返回

    // 负温度: 五阶多项式拟合
    rt /= rtd_nominal;
    rt *= 100;                          // 归一化到 100 ohm
    rpoly = rt;
    temp = -242.02f;
    temp += 2.2228f * rpoly;    rpoly *= rt;
    temp += 2.5859e-3f * rpoly; rpoly *= rt;
    temp -= 4.8260e-6f * rpoly; rpoly *= rt;
    temp -= 2.8183e-8f * rpoly; rpoly *= rt;
    temp += 1.5243e-10f * rpoly;

    return temp;
}
```

### Orange Pi PC Plus 编译方法

**Makefile 方式:**
```shell
sudo apt-get install libgpiod-dev pkg-config cmake -y
cd orangepi-pc-plus
make
sudo make install
```

**CMake 方式:**
```shell
mkdir build && cd build
cmake ..
make
sudo make install
```

### CLI 命令示例

```shell
# 查看芯片信息
./max31865 -i

# 查看引脚连接
./max31865 -p

# 运行寄存器测试
./max31865 -t reg

# 运行读取测试 (4线, PT100, 参考电阻430ohm, 测3次)
./max31865 -t read --wire=4 --type=100 --resistor=430.0 --times=3

# 连续读取示例
./max31865 -e read --wire=4 --type=100 --resistor=430.0 --times=3

# 单次读取示例
./max31865 -e shot --wire=4 --type=100 --resistor=430.0 --times=3
```

### 默认配置常量

```c
#define MAX31865_BASIC_DEFAULT_FILTER_SELECT                MAX31865_FILTER_SELECT_50HZ
#define MAX31865_BASIC_DEFAULT_FAULT_DETECTION_CYCLE_CONTROL MAX31865_FAULT_DETECTION_CYCLE_CONTROL_AUTOMATIC_DELAY
#define MAX31865_BASIC_DEFAULT_HIGH_FAULT_THRESHOLD         0xFFFEU
#define MAX31865_BASIC_DEFAULT_LOW_FAULT_THRESHOLD          0x0000U
```

---

## 相关链接

- **源码位置**: `C:\Users\lijian\Downloads\temperature\`
- **目标平台**: Orange Pi PC Plus (Allwinner H3, SPI: PC0-PC3)
- **依赖库**: libgpiod-dev, pkg-config, cmake
- **驱动库来源**: LibDriver MAX31865 (https://github.com/libdriver/max31865)
- **芯片 Datasheet**: Maxim Integrated MAX31865 RTD-to-Digital Converter
- **许可协议**: MIT License
