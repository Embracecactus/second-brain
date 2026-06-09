---
tags: [fpga]
category: hardware-design/fpga
created: 2026-06-09
---
Now I have all the information needed. Here is the comprehensive Obsidian note:

---
tags:
  - fpga
  - gowin
  - verilog
  - led
  - hardware-design
category: hardware-design/fpga
created: 2026-06-09
status: complete
---

# FPGA LED 流水灯项目

## 项目概述

基于高云（Gowin）GW1N-1 系列 FPGA 开发板的 LED 流水灯实验项目，使用 Verilog HDL 实现三色 LED 灯循环切换，是 FPGA 数字逻辑设计的入门级实践。

## 技术栈

- **FPGA 厂商**：Gowin Semiconductor（高云半导体）
- **FPGA 型号**：GW1N-LV1QN48C6/I5（GW1N-1 系列）
- **硬件描述语言**：Verilog (IEEE 1364-2001)
- **开发工具**：Gowin EDA V1.9.12 (64-bit)
- **综合工具**：GowinSyn（内置综合器）
- **约束格式**：CST（物理约束文件）
- **IO 标准**：LVCMOS 1.8V
- **封装**：QN48（48 引脚 QFN）

## 项目结构

```
fpga_project/
├── fpga_project.gprj          # 项目主文件（XML 格式）
├── fpga_project.gprj.user     # 用户配置
├── impl/
│   ├── fpga_project_process_config.json  # 实现配置
│   ├── gwsynthesis/            # 综合输出
│   ├── pnr/                    # 布局布线输出
│   └── temp/                   # 临时文件
└── src/
    ├── led.v                   # 顶层 Verilog 源文件
    └── fpga_project.cst        # 引脚约束文件
```

## 架构与设计决策

### 顶层模块设计

LED 模块采用单一时钟域、异步复位架构：

```verilog
module led (
    input sys_clk,          // 系统时钟，Pin 35
    input sys_rst_n,        // 低有效复位，Pin 15
    output reg [2:0] led    // 三色 LED (B=bit2, R=bit1, G=bit0)
);
```

### 核心设计参数

- **时钟频率**：24MHz（由开发板晶振提供，计数到 11_999_999 产生 0.5 秒延迟）
- **计数器宽度**：24 位（2^24 = 16,777,216 > 12,000,000）
- **LED 切换逻辑**：循环左移 `{led[1:0], led[2]}`

### 时序逻辑设计

设计包含两个 `always` 块，均使用 `posedge sys_clk or negedge sys_rst_n` 异步复位模式：

1. **计数器块**：24 位计数器在 0 ~ 11,999,999 之间循环计数
2. **LED 控制块**：当计数器达到最大值时，LED 进行循环左移

### 引脚约束

| 信号 | 引脚 | IO 类型 | 说明 |
|------|------|---------|------|
| sys_clk | 35 | LVCMOS18 | 系统时钟输入 |
| sys_rst_n | 15 | LVCMOS18 | 低有效复位（上拉） |
| led[0] | 16 | LVCMOS18 | 绿色 LED |
| led[1] | 17 | LVCMOS18 | 红色 LED |
| led[2] | 18 | LVCMOS18 | 蓝色 LED |

## 关键实现细节

### 计数器分频原理

```verilog
// 0.5 秒延迟计算：
// 24MHz * 0.5s = 12,000,000 个时钟周期
// 计数范围 0 ~ 11,999,999
else if (counter < 24'd1199_9999)
    counter <= counter + 1'b1;
```

### LED 循环移位模式

初始状态 `3'b110`（仅蓝灯亮），每 0.5 秒循环左移：

```
110 (B) -> 101 (R) -> 011 (G) -> 110 (B) -> ...
```

实现使用位拼接操作：`{led[1:0], led[2]}`，这是 Verilog 中实现循环移位的经典写法。

## 构建与运行

1. 使用 Gowin EDA（教育版或商业版）打开 `fpga_project.gprj`
2. 执行 Synthesis（综合）-> Place & Route（布局布线）-> Generate Bitstream（生成比特流）
3. 通过 JTAG 下载到 GW1N-1 开发板
4. 上电后观察三色 LED 以 0.5 秒间隔循环切换

## 学习要点与经验

- **异步复位**：GW1N 系列 FPGA 推荐使用异步复位同步释放模式，本项目简化为纯异步复位
- **时序约束**：对于简单设计，Gowin 工具可自动推断时钟约束；复杂设计需添加 SDC 文件
- **IO 电平**：GW1N-1 Bank VCCIO 设为 1.8V，使用 LVCMOS18 标准
- **Verilog 编码风格**：使用 `always @(posedge clk or negedge rst_n)` 是推荐的异步复位写法
- **计数器上限**：`24'd1199_9999` 使用下划线分隔提高可读性，这是 Verilog-2001 标准特性

## 相关概念

- [[FPGA]] - 现场可编程门阵列基础
- [[Verilog HDL]] - 硬件描述语言
- [[Gowin FPGA]] - 高云半导体 FPGA 平台
- [[数字逻辑设计]] - 数字电路设计基础
- [[时序分析]] - FPGA 时序约束与分析
- [[LED 驱动电路]] - LED 硬件接口设计
- [[时钟分频]] - FPGA 时钟管理技术

## 相关笔记

- [[wenan]] — 嵌入式学习笔记合集（含 FPGA 学习）
- [[xmind-notes]] — XMind 知识库（含 FPGA 点灯笔记）
- [[wch]] — WCH CH32V RISC-V MCU 项目
