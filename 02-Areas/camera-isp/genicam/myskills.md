---
tags: [genicam, industrial-camera, sensor, fpga]
category: camera-isp/genicam
created: 2026-06-09
---
Now I have enough information to generate a comprehensive Obsidian note. Let me compile all the knowledge extracted from this project.

```markdown
---
title: CamSkills - AI辅助工业相机固件开发技能包
tags:
  - camera-isp
  - genicam
  - industrial-camera
  - firmware
  - ai-skill
  - cline
  - sensor-driver
  - isp-algorithm
  - fpga
category: camera-isp/genicam
created: 2026-06-09
status: active
project_path: /home/lijian/project/myskills/camskills
related_projects:
  - MVDF-Drv
---

# CamSkills - AI辅助工业相机固件开发技能包

## 项目概述

CamSkills 是一套面向工业相机固件开发（MVDF-Drv）的 AI 辅助技能集合，基于 Cline AI 编程助手运行。该技能包将工业相机固件开发中高频、重复的工作流程提炼为标准化自动化 Skill，覆盖从协议规范、传感器驱动、ISP 算法到产品配置的完整开发链路，将单环节开发时间从 1-2 小时缩短至 5-10 分钟。

## 技术栈

- **AI 平台**: Cline（VS Code AI 编程助手）/ Claude Code
- **技能格式**: Markdown（SKILL.md + YAML frontmatter）
- **知识资源**: GenICam 标准 PDF 文档 3 份（SFNC v2.7、PFNC 2.4、Core Standard v2.1）
- **知识库格式**: JSON（GenICam 专家系统 5 个）、YAML（传感器信息/bug 数据库）
- **脚本语言**: Python 2.7（sc-sensor-driver 脚本）、Python 3（genicam-expert 实现）
- **Python 依赖**: PyYAML、xlrd
- **适用项目**: MVDF-Drv 工业相机固件工程
- **硬件平台**: 安路(Anlogic) / Xilinx FPGA，USB3 / GigE / CXP 接口

## 架构设计

### 核心设计哲学：Skill-as-Prompt

Skills 不是编译运行的软件，而是结构化的 Markdown 文档（SKILL.md），指导 AI 助手完成特定开发工作流。每个 Skill 通过 YAML frontmatter 声明 `name` 和 `description`，正文包含逐步工作流指令。

### 标准 Skill 目录结构

```
{skill-name}/
├── SKILL.md           # Skill 定义（必需，含 YAML frontmatter）
├── knowledge/         # 结构化知识库（YAML/JSON）
├── resources/         # 原始参考资料（PDF datasheet、INI 文件）
├── references/        # 代码模板和参考文档
├── scripts/           # 辅助 Python 脚本
├── cases/             # 适配案例记录
└── checklists/        # 代码审查清单
```

### Skill 流水线（开发流程顺序）

```
genicam-expert → genicam-xml-builder → general.xml / reg_address.h
sc-sensor-driver → sensor-driver-adapter → MVDF-Drv 框架
isp-algorithm-developer → alg_main.c
product-config-creator → configure / CameraVersion.csv
seq-1n-mapper → FPGA 序列配置（XML 节点来自 genicam-xml-builder）
camera-add-feature / camera-customization-function → FPGA 寄存器特性
```

### 技能间协作关系

- `sc-sensor-driver` 生成驱动代码后，需调用 `sensor-driver-adapter` 完成 4 处框架注册
- `genicam-expert` 提供标准验证，`genicam-xml-builder` 基于验证结果生成 XML 代码
- `product-config-creator` 创建新产品配置时，需配合传感器和 FPGA 驱动配置
- `seq-1n-mapper` 修改的 FPGA 序列功能，其 XML 节点由 `genicam-xml-builder` 生成

## 7 个已完成技能

| 序号 | 技能名称 | 功能定位 | 核心价值 |
|------|---------|---------|---------|
| 1 | genicam-expert | GenICam 协议专家系统 | 基于 3 份官方 PDF 标准，提供属性命名验证、类别分配、数据类型校验 |
| 2 | genicam-xml-builder | GenICam XML 节点构建工具 | 自动生成 XML 节点定义、寄存器地址分配、C 回调代码 |
| 3 | sc-sensor-driver | SmartSens SC 系列驱动开发 | 解析 .ini 文件，自动生成 SC130/SC136/SC235/SC535/SC950 驱动代码 |
| 4 | sensor-driver-adapter | 传感器驱动适配注册 | 新增传感器时自动完成 compile.mk / drv_sensor.c / commonDefine.h / configure 4 处联动 |
| 5 | isp-algorithm-developer | ISP 算法模块开发 | 自动完成 ModuleFlag 注册、alg_main.c 调度接入、FPGA 寄存器映射 |
| 6 | product-config-creator | 产品型号配置 | 新增产品型号时自动创建 configure 文件、更新 CameraVersion.csv |
| 7 | seq-1n-mapper | SequencerFPGA 1N 映射 | 将 2N 配对模式改为 1N 单循环，支持 16 组独立曝光参数 |

## 关键知识点

### GenICam 寄存器地址空间

- 地址范围: `0x10000` - `0x1EFFF`（USB3Vision）
- 对齐方式: 4 字节对齐
- 分配策略: 按功能区域划分

### GenICam 专家系统知识库（5 个 JSON 文件）

- `standard_categories` - 标准类别定义
- `standard_properties` - 标准属性定义
- `naming_patterns` - 命名模式规范
- `data_type_rules` - 数据类型规则
- `address_ranges` - 地址范围分配

### SC 系列传感器驱动关键点

- **支持型号**: SC130、SC136、SC160、SC235、SC535、SC950
- **通信接口**: GigE（SPI，deviceid 0xFE/0xFF）、USB3（SPI，addr<<1|WR_FLAG）
- **INI 文件解析**: ParaList 格式（非标准 Python configparser），包含 VTS/HTS/PCLK/SCLK 时序参数
- **关键计算**: lineTime、帧率、曝光设置（GigE 用 resetTime，USB 用 PIXEL_RESET+VTS_QUANTITY）
- **增益映射**: 0x3e08/0x3e09 寄存器，PGGAIN 模式
- **知识库**: sensor_info.yaml（型号参数）、bug_database.yaml（bug 经验库）
- **Python 兼容性**: 脚本必须兼容 Python 2.7

### ISP 算法流水线

支持完整 ISP 流水线模块注册：
`GIC → IIF → Gain → BLC → FPN → SPC → Sharpness → Tone → Denoise → Statistics → AWB → AE → CCM → RGB2YUV → FFC → ColorAdjust → DFC → PGI`

现有 18 个模块参考实现，支持多种运行模式（Thread/Gain/Exposure/ROI/FPGA Sequence）。

### sensor-driver-adapter 4 处联动注册

新增传感器驱动时必须完成的 4 处修改：
1. `compile.mk` - 添加编译入口
2. `drv_sensor.c` - 添加 OP 声明和映射表注册
3. `commonDefine.h` - 添加传感器枚举编号
4. `configure` - 添加条件编译宏

支持接口: GigE 1G/10G/Line、USB、USB 1U/2U、CXP

### 产品编码规则

格式: `AH{R}{S}{I}{O}{XXX}`
- R = 分辨率、S = 传感器类型、I = 接口、O = 选项、XXX = 版本号

## 附加项目：U3V 日志分析 Skill

基于 CamSkills 的 Skill 设计方法论，还设计了一套 U3V（USB3Vision）日志分析 Skill，采用三级递进式架构：

1. **u3v-log-basic** - 单文件基础版，关键词→诊断映射表，约 200 行
2. **u3v-log-branching** - 4 分支判断版（USB 连接/端点传输/DMA-GPIF/初始化系统），含二级决策树和严重程度分级，约 300-350 行
3. **u3v-log-knowledge** - 知识库驱动版，集成 4 个 YAML 知识文件（error-patterns / sdk-error-codes / troubleshooting / hw-variants），输出完整诊断报告

## 构建与运行

```bash
# GenICam 专家系统测试
cd genicam-expert && python test_expert.py

# INI 解析器（Python 2.7 兼容）
python sc-sensor-driver/scripts/ini_parser.py --search-all SC136HGS --resources-dir ../resources

# Bug 知识库搜索
python sc-sensor-driver/scripts/bug_search.py <keyword>

# 从 git 历史同步 bug 记录
python sc-sensor-driver/scripts/sync_git_bugs.py --repo <MVDF-Drv-path>

# Python 依赖安装（使用大华镜像源）
pip install -i https://pypi.dahuatech.com/simple/ PyYAML xlrd
```

### Skill 部署

Skills 部署到 `.codebuddy/skills/{skill-name}/SKILL.md`，使用 `skill-deployer` 技能。部署时 `references/`、`scripts/`、`knowledge/` 目录需随 SKILL.md 一起复制。

## 关键设计洞察

1. **Skill-as-Prompt 范式**: Skill 是指导 AI 的结构化文档而非可执行代码，核心价值在于将领域专家知识固化为 AI 可理解的工作流
2. **四要素模型**: Skill = 领域知识 + 工作流程 + 代码模板 + 验证规则
3. **三级演进路径**: Level 1 流程型（5 步内无分支）→ Level 2 条件分支型（5-10 步）→ Level 3 多阶段架构型（10+ 步，跨架构层）
4. **从错误到规则**: AI 犯过的错就是该写进 Skill 的规则（如 sensor-driver-adapter 的 4 处联动规则即从 AI 遗漏中提炼）
5. **知识库双模式**: "标准参考型"（如 genicam-expert 的 5 个 JSON）和"经验积累型"（如 sc-sensor-driver 的 bug_database.yaml）

## 相关概念链接

- [[GenICam]] - 通用工业相机接口标准
- [[USB3Vision]] - USB3 接口的工业相机传输协议
- [[ISP-Image-Signal-Processing]] - 图像信号处理流水线
- [[SmartSens-SC-Series]] - 思特威 CMOS 传感器系列
- [[MVDF-Drv-Firmware]] - 目标固件工程项目
- [[Cline-AI-Assistant]] - AI 编程助手平台
- [[FPGA-Sequencer]] - FPGA 序列曝光控制
```

## 相关笔记

- [[camera-diag-skills]] — Camera 诊断技能
- [[skill]] — GenICam 属性管理与专家系统
- [[work]] — SC-Sensor-Driver 专家系统
- [[rv1126b]] — RV1126B 运动相机项目
- [[ok1126b-sdk]] — OK1126B SDK 与项目知识库
