---
tags: [sensor-driver, sc-series, mvdf]
category: camera-isp/sensor-driver
created: 2026-06-09
---
I now have enough information to generate a comprehensive Obsidian note. Let me produce it.

```yaml
---
title: "GenICam 属性管理与专家系统"
category: camera-isp/sensor-driver
tags:
  - genicam
  - industrial-camera
  - xml
  - register-management
  - claude-code-skill
  - python
  - sfnc
  - pfnc
status: active
created: 2026-06-09
version: "2.0"
related:
  - "[[GenICam SFNC]]"
  - "[[GenICam PFNC]]"
  - "[[工业相机固件开发]]"
  - "[[寄存器地址管理]]"
---

# GenICam 属性管理与专家系统

## 项目概述

本项目是一套基于 GenICam 协议的工业相机属性管理工具，包含两个 Claude Code Skill：**genicam-property**（基础版）和 **genicam-expert**（专家版）。系统能够解析用户自然语言输入，自动验证属性名称与类型，分配不冲突的寄存器地址，并生成符合 GenICam 标准的 XML 属性定义和 C 语言寄存器地址宏定义。目标文件为大华（Dahua）HUARUI 系列工业相机的 XML 描述文件和寄存器头文件。

## 技术栈

- **语言**: Python 3.10
- **协议标准**: GenICam Standard v2.1.1、SFNC v2.7、PFNC v2.4
- **XML 处理**: xml.etree.ElementTree + minidom
- **工具形态**: Claude Code Skill + 独立 Python 模块
- **目标平台**: 大华 HUARUI USB3 工业相机
- **寄存器位宽**: 32-bit 地址空间，4 字节对齐

## 项目结构

```
/home/lijian/project/skill/
├── skills/
│   ├── genicam-property/          # 基础版 Skill
│   │   ├── skill.md               # Skill 定义（Claude Code 读取）
│   │   ├── implementation.py      # Python 实现（411行）
│   │   ├── README.md              # 完整文档
│   │   ├── USAGE.md               # 使用指南
│   │   └── COMPLETE.md            # 完成报告
│   └── genicam-expert/            # 专家版 Skill
│       ├── skill.md               # Skill 定义
│       ├── README.md              # 完整文档
│       ├── QUICKSTART.md          # 快速入门
│       ├── test_expert.py         # 测试脚本
│       ├── knowledge/             # 5个JSON知识库
│       └── resources/             # 3个PDF标准文档
├── recousure/
│   ├── general.xml                # GenICam XML 描述文件（36871行）
│   ├── reg_address.h              # 寄存器地址宏定义（566行）
│   ├── GenICam_SFNC_v2_7.pdf
│   ├── GenICam_PFNC_2_4.pdf
│   └── GenICam_Standard_v2_1_1.pdf
└── __pycache__/
```

## 架构与关键设计决策

### 双层 Skill 架构

项目采用 **基础版 + 专家版** 的分层设计：

- **genicam-property**: 轻量级属性管理，直接操作 XML 和头文件，不依赖 PDF 文档
- **genicam-expert**: 知识库驱动的专家系统，维护 5 个从 PDF 提取的 JSON 知识库，提供标准符合性检查、智能类别推荐和命名规范验证

### 寄存器地址空间规划

系统对寄存器地址进行了功能分区，这是工业相机固件中的典型设计：

| 地址范围 | 用途 |
|----------|------|
| `0x0001xxxx` | 设备控制、采集控制 |
| `0x2001xxxx` | 采集帧计数 |
| `0x2100xxxx` | Strobe、Binning |
| `0x3000xxxx` | 触发、文件访问、Multi-ROI、HDR |
| `0x4E05xxxx` | GenICam 标准属性（主要区域） |
| `0x5000xxxx` | 自定义寄存器区域（自动分配起始地址） |
| `0x5300xxxx` | FPGA Sequencer |
| `0x5400xxxx` | Trigger Ignore |
| `0xEFFxxxxx` | Flash、用户数据、FFC |
| `0xEFFFFxxx` | 更新、密码、设备ID |

### GenICam XML Category 结构

XML 文件遵循 GenICam SFNC 标准的功能类别划分：

- **DeviceControl** -- 设备信息与控制
- **ImageFormatControl** -- 图像格式（ROI、Binning、Decimation）
- **AcquisitionControl** -- 采集控制（触发、曝光、帧率）
- **AutoFunctionControl** -- 自动功能（自动曝光、自动增益）
- **DigitalIOControl** -- 数字 IO
- **AnalogControl** -- 模拟控制（增益、黑电平、白平衡）
- **LUTControl** -- 查找表
- **EventControl** -- 事件控制
- **TransportLayerControl** -- 传输层
- **UserSetControl** -- 用户集
- **ISPControl** -- 图像信号处理
- **SequencerFPGAControl** -- FPGA 序列器

### XML 生成模式

每个属性由两部分组成：**功能节点**（如 `<Integer>`）+ **寄存器节点**（`<IntReg>`），通过 `<pValue>` 引用关联：

```xml
<Integer Name="SensorTemperature" NameSpace="Standard">
    <pValue>SensorTemperatureReg</pValue>
    <Min>-40</Min>
    <Max>125</Max>
</Integer>
<IntReg Name="SensorTemperatureReg">
    <Address>0x50000008</Address>
    <Length>4</Length>
    <AccessMode>RO</AccessMode>
    <pPort>Device</pPort>
    <Endianess>LittleEndian</Endianess>
</IntReg>
```

对应的 C 宏定义：
```c
#define SENSORTEMPERATURE_ADDR            (0x50000008)
```

## 关键实现细节

### 属性管理器核心类 `GenICamPropertyManager`

```python
class GenICamPropertyManager:
    def __init__(self, xml_file="recousure/general.xml",
                 reg_header_file="recousure/reg_address.h"):
        self.existing_addresses = set()    # 已占用地址集合
        self.existing_properties = {}      # 属性名 -> 地址映射
        self._load_existing_data()         # 启动时加载现有数据

    def find_available_address(self, base_address=0x50000000) -> int:
        """从基地址开始，按4字节步长查找可用地址"""
        addr = base_address
        while addr < base_address + 0x10000000:
            if addr not in self.existing_addresses:
                self.existing_addresses.add(addr)
                return addr
            addr += 4
        raise Exception("无法找到可用的寄存器地址")
```

### 验证机制

系统实现了多层验证：

1. **名称验证**: 正则 `^[A-Za-z][A-Za-z0-9_]*$`，保留字检查
2. **类型验证**: 支持 Integer / Float / StringReg / Boolean / Enumeration / Command
3. **访问模式验证**: RO / WO / RW
4. **地址冲突检测**: 从 `reg_address.h` 和 `general.xml` 双源加载已有地址
5. **名称冲突检测**: 检查 `{NAME}_ADDR` 是否已存在

### 支持的数据类型

| 类型 | 必需参数 | 可选参数 | 说明 |
|------|----------|----------|------|
| Integer | min, max | inc, unit, representation | 整数寄存器 |
| Float | min, max | inc, unit, representation | 浮点寄存器 |
| Enumeration | enum_entries[] | -- | 枚举类型 |
| Boolean | -- | -- | 布尔值 |
| StringReg | -- | length (默认64) | 字符串寄存器 |
| Command | -- | command_value | 命令类型 |

## 关键学习与洞察

1. **GenICam 的 "功能-寄存器" 分离设计**: 每个用户可见属性（如 `ExposureTime`）与其底层寄存器（`ExposureTimeReg`）是独立的 XML 节点，通过 `<pValue>` 引用连接。这种设计允许同一个寄存器被多个功能节点引用，也支持通过表达式动态计算值。

2. **寄存器地址的空间隔离**: 不同功能域使用不同的地址段（`0x0001` 设备控制、`0x4E05` 标准属性、`0x5000` 自定义等），这使得固件侧的地址解码可以按高位地址进行快速路由。

3. **Cachable 与 pInvalidator 机制**: GenICam XML 中的 `<Cachable>WriteThrough</Cachable>` 和 `<pInvalidator>` 标签用于控制客户端的寄存器缓存策略，`pInvalidator` 指向一个"失效触发器"寄存器，当该寄存器被写入时，客户端需要重新读取相关寄存器。

4. **Claude Code Skill 的知识封装模式**: 项目展示了如何将领域知识（PDF 标准文档）结构化为 JSON 知识库，再通过 `skill.md` 定义 Agent 行为，实现"知识 + 工具 + 行为"的三层封装。

5. **自然语言到硬件寄存器的映射**: 用户输入"添加一个温度传感器属性，只读，-40到125度"，系统能自动完成属性命名、类型选择、地址分配、XML 生成和头文件更新的全链路工作。

## 使用方式

### 作为 Claude Code Skill

```
/genicam-property 添加SensorTemperature属性，整数，只读，-40到125度，单位°C
/genicam-expert 添加GainAuto枚举属性
```

### 作为 Python 模块

```python
from skills.genicam_property.implementation import add_property_from_dict

success, result = add_property_from_dict({
    "name": "SensorTemperature",
    "type": "Integer",
    "description": "传感器温度",
    "min": -40, "max": 125,
    "access_mode": "RO"
})
```

## 参考标准

| 文档 | 版本 | 大小 | 内容 |
|------|------|------|------|
| GenICam_SFNC | v2.7 | 8.1MB | Standard Features Naming Convention -- 命名规范与标准属性定义 |
| GenICam_PFNC | v2.4 | 927KB | Pixel Format Naming Convention -- 像素格式命名 |
| GenICam Standard | v2.1.1 | 1014KB | 核心标准 -- XML 架构、寄存器访问、传输层 |

## 相关概念链接

- [[GenICam SFNC]] -- 标准功能命名规范
- [[GenICam PFNC]] -- 像素格式命名规范
- [[工业相机固件寄存器设计]]
- [[USB3 Vision 协议]]
- [[Claude Code Skill 开发]]
- [[ISP 图像信号处理]]
```
## 相关笔记

- [[myskills]] — CamSkills 工业相机固件开发技能包
- [[camera-diag-skills]] — Camera 诊断技能
- [[work]] — SC-Sensor-Driver 专家系统
- [[rv1126b]] — RV1126B 运动相机项目
