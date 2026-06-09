---
tags: [sensor-driver, sc-series, gige, usb]
category: camera-isp/sensor-driver
created: 2026-06-09
---
I now have all the information needed. Let me generate the Obsidian note.

---
tags:
  - camera-isp
  - sensor-driver
  - smartsens
  - industrial-camera
  - embedded
  - knowledge-driven
category: camera-isp/sensor-driver
created: 2026-06-09
status: active
project: work
---

# SC-Sensor-Driver -- SmartSens SC系列 Sensor 驱动开发专家系统

## 项目概述

SC-Sensor-Driver 是一个基于 AI 的工业相机图像传感器驱动开发专家系统，针对 SmartSens SC 系列 sensor（SC130/SC136/SC160/SC235/SC535/SC950 等）进行驱动适配。项目通过结构化知识库、自动化工具脚本和标准化流程，将传统 3-5 天的驱动适配周期缩短至 0.5-1 天，实现效率提升 3-6 倍。

## 技术栈

| 技术领域 | 具体技术 |
|---------|---------|
| 驱动开发框架 | MVDF-Drv（工业相机驱动框架） |
| 驱动编程语言 | C 语言 |
| 工具脚本语言 | Python |
| 知识表示 | YAML 结构化数据、Markdown 文档 |
| 版本控制与知识管理 | Git |
| 演示工具 | PptxGenJS (Node.js) |
| 支持接口 | GigE（以太网）、USB |
| 目标 Sensor | SmartSens SC 系列 |

## 架构与关键设计决策

### 总体架构

```
sc-sensor-driver/
├── SKILL.md                          # 系统核心配置（"大脑"）
├── knowledge/                        # 结构化知识库
│   ├── bug_database.yaml             # Bug 经验库（23条记录，5大分类）
│   └── sensor_info.yaml              # Sensor 参数信息库（19个型号）
├── references/                       # 代码模板与配置示例
│   ├── sc_driver_template.md         # 驱动代码模板（GigE+USB 双版本）
│   └── sample_init.ini               # .ini 初始化文件示例
├── checklists/                       # 质量保证检查清单
│   ├── new_driver_checklist.md       # 9大类50+项交付检查
│   └── code_review_checklist.md      # 7大类代码审查检查
├── cases/                            # 驱动适配案例
├── resources/                        # 数据手册、.ini 文件（19个子目录）
├── scripts/                          # 自动化工具
│   ├── ini_parser.py                 # .ini 文件解析器
│   ├── bug_search.py                 # Bug 知识库搜索工具
│   └── sync_git_bugs.py              # Git 历史 Bug 同步工具
└── assets/                           # 图表辅助资源
```

### 关键设计决策

**1. 知识驱动 + 工具自动化 + 流程标准化**
项目的核心理念不是用 AI 替代工程师，而是将专家知识结构化沉淀，通过工具自动化执行标准化流程，让 AI 系统（SKILL.md 定义的 Skill）在每次调用时严格遵循 6 步驱动适配工作流程。

**2. 6 步驱动适配工作流程**
```
步骤1: 收集必要信息（交互式）→ 步骤2: 选择参考驱动 → 步骤3: 解析 .ini 和数据手册
→ 步骤4: 生成驱动代码 → 步骤5: 检查常见问题（Bug 库对照）→ 步骤6: 集成到构建系统
```

**3. GigE vs USB 接口差异化知识体系**
系统化整理了 9 个关键差异点，避免开发人员混淆两种接口的实现逻辑：

| 差异项 | GigE | USB |
|--------|------|-----|
| SPI 通信 | deviceid=0xFE/0xFF, addrLen=24 | 多数同 GigE；SC130USB: `addr<<1\|WR_FLAG`, addrLen=17 |
| lineTime | 12.3~19.79 us | 5.96~6.8 us |
| 输出格式 | 固定 12bit | 可 setLineDummy 切换 10bit/12bit/8bit |
| 增益控制 | 含数字增益(0x3e06/0x3e07) | 仅模拟增益(0x3e08/0x3e09) |
| setExp 补偿 | resetTime 行数 | PIXEL_RESET 行时间 + VTS_QUANTITY 行数 |
| 格式切换 | 无 | setLineDummy 支持运行时切换 |

**4. 数据驱动决策优先级**
```
FAE 已知问题/workaround > FAE 特定场景配置 > .ini 文件（厂商验证序列）> 数据手册推荐值 > 推算值
```

**5. Bug 经验库分类体系**

| 分类缩写 | 名称 | 说明 |
|---------|------|------|
| `fmt` | format_switch | 格式切换相关（6+条） |
| `lvd` | LVDS | LVDS/图像质量相关 |
| `gai` | gain_exposure | 增益/曝光相关 |
| `roi` | ROI_seq | ROI/Sequencer 相关 |
| `ini` | init_default | 初始化/默认值相关 |

## 关键代码模式

### 时序参数推算公式

```c
lineTime(us) = HTS / PCLK * 1000
frameRate = PCLK * 1000 / (VTS * HTS)
```

参数来源优先级：.ini 注释行显式值 > ParaList 段 0x320c/0x320d/0x320e/0x320f 寄存器推算 > 数据手册 > FAE 修正值。

### GigE 与 USB 曝光控制差异

GigE 版本：
```c
frameRdTime = (imgHeight + 26) * lineTime;
tFramDly = expValue + frameRdTime - (1000000.0/fps) - exp;
FPGA_SET_REG_VAL(FPGA_RID_SENCTRL_FRM_DLY, (uint32_t)(tFramDly + 520));
FPGA_SET_REG_VAL(FPGA_RID_SENCTRL_EXP_LINE_NUM, exp + resetTime);
```

USB 版本：
```c
frameReadTime = (imgHeight + 36 + 16 + VTS_QUANTITY) * lineTime;
tFrameDly = expValue + frameReadTime - (1000000.0/fps) - exp;
FPGA_SET_REG_VAL(FPGA_RID_SENCTRL_FRM_DLY, (uint32_t)(tFrameDly + 520));
FPGA_SET_REG_VAL(FPGA_RID_SENCTRL_EXP_LINE_NUM, (exp + PIXEL_RESET * lineTime - 4));
```

### 增益控制关键点

```c
// 0x3e08: 增益模式寄存器
// 0x00=1x, 0x01=2x, 0x80=PG*3.1, 0x81=PG*6.2, 0x83=PG*12.4, 0x87=PG*24.8
// 0x3e09: 模拟增益值 = 32 * anaGain / modeBase
// 写入顺序：先 0x3e08（模式），再 0x3e09（值），不可颠倒！
```

### USB 格式切换（setLineDummy）流程

1. 先 reset FPGA LVDS RX 和 sensor
2. 根据 pixformat 选择对应 init 序列和 lineTime
3. 修改 FPGA_DUMY_H_LENGTH（帧消隐行长）
4. 修改 DESER_TYPE（1=10bit, 2=12bit）
5. 重新对齐 LVDS 时序

### Bug 经验库数据结构（bug_database.yaml）

```yaml
- id: BUG-{分类缩写}-{3位序号}   # 如 BUG-fmt-007
  title: 简短描述 bug 现象
  category: format_switch        # fmt/lvd/gai/roi/ini
  models: [SC535USB]
  interface: USB                 # GigE/USB/both
  root_cause: 根因分析
  fix: 修复方式
  git_keyword: git 搜索关键词
  source: git_history            # git_history/user_report
  related_code: setLineDummy
```

### Sensor 参数信息结构（sensor_info.yaml）

```yaml
SC535:
  full_name: SC535GS
  resolution: "2448x2048"
  pixel_size: "2.5um"
  max_framerate_10bit: 65
  max_framerate_12bit: 55
  output_formats: [RG10, RG12, RGB8, YUV422]
  interfaces:
    gige:
      spi_deviceid_write: 0xFE
      spi_deviceid_read: 0xFF
      spi_addrlen: 24
      lineTime_typical_us: 19.79
      pggain: 3.1
      resetTime: 26
    usb:
      spi_deviceid_write: 0xFE
      spi_deviceid_read: 0xFF
      spi_addrlen: 24
      lineTime_typical_us: 6.8
      pggain: 3.1
      pixel_reset: 4
      vts_quantity: 16
      lvds_channels: 4
      has_setLineDummy: true
```

## 关键学习与洞察

### 1. Git 历史是宝贵的 Bug 知识源
通过 `sync_git_bugs.py` 自动扫描 MVDF-Drv 仓库的 git 提交历史，提取 Bug 修复信息并结构化存储。当前已积累 23 条结构化 Bug 记录。知识不依赖个人经验，形成组织级资产。

### 2. 接口差异是驱动适配的核心风险
GigE 和 USB 接口在 SPI 通信方式、时序参数、增益控制、格式切换等 9 个维度存在差异。系统化整理这些差异并在代码模板中提供双版本实现，是避免混淆的关键。

### 3. 多源交叉验证确保参数准确性
.ini 文件是厂商验证过的配置序列，是最可靠的数据源。但需与数据手册和 FAE 文档交叉验证，特别是 lineTime 等时序参数的推算。

### 4. 预防优于修复
在代码生成阶段自动对照历史 Bug 库（步骤 5），比事后修复效率高得多。9 大类 50+ 项交付检查清单确保零遗漏。

### 5. SC130USB 的特殊 SPI 通信方式
SC130 系列 USB 版本使用特殊的 SPI 地址格式：`addr << 1 | WR_FLAG`，addrLen=17，与标准方式（deviceid=0xFE/0xFF, addrLen=24）完全不同，必须在步骤 1 自动识别。

### 6. 数据手册 PDF 全部加密
所有 SmartSens 数据手册 PDF 密码统一为 `dahua`。PDF 解析能力有限，优先从 .ini 文件提取参数。

## 实际应用效果

| 指标 | 传统方式 | 使用本系统 | 提升幅度 |
|------|---------|-----------|---------|
| 驱动适配周期 | 3-5 天 | 0.5-1 天 | 3-6 倍 |
| 参数收集时间 | 2-4 小时 | 10-30 分钟 | 4-8 倍 |
| Bug 排查时间 | 4-8 小时 | 0.5-2 小时 | 4-8 倍 |
| 代码审查时间 | 2-3 小时 | 0.5-1 小时 | 3-4 倍 |

## 构建/运行指令

### ini_parser.py -- .ini 文件解析与资源搜索

```bash
# 搜索指定型号的所有 .ini 文件和数据手册
python scripts/ini_parser.py --search-all SC132GS --resources-dir resources

# 解析 .ini 文件，提取参数和 init 序列
python scripts/ini_parser.py <.ini文件路径> --resources-dir resources

# 保存用户确认的参数
python scripts/ini_parser.py --save-params SC132GS --resources-dir resources --lineTime 6.8 --pggain 3.1
```

### bug_search.py -- Bug 知识库搜索

```bash
# 按关键词搜索
python scripts/bug_search.py "格式切换"

# 按分类过滤
python scripts/bug_search.py --category format_switch

# 按型号过滤
python scripts/bug_search.py --model SC535USB
```

### sync_git_bugs.py -- Git 历史 Bug 同步

```bash
# 扫描 MVDF-Drv 仓库 git 历史，同步到 bug_database.yaml
python scripts/sync_git_bugs.py
```

### 演示文稿生成

```bash
# 使用 Node.js 生成项目汇报 PPT
npm install
node create_presentation.js
```

## 已覆盖的 Sensor 型号

### 已实现驱动（7个 sensor，9个接口变体）
SC130GS、SC136HGS、SC160HGS、SC235HGS、SC535GS（GigE+USB）、SC950HGS

### 待开发驱动（12个）
SC132GS、SC133HGS、SC135HGS、SC233HGS、SC410GS、SC480DC、SC538HGS、SC635HGS、SC650HGS、SC850SL、SC1235HGS、SC460LA

## 后续开发计划

| 阶段 | 周期 | 重点任务 |
|------|------|---------|
| 阶段1: 工具化完善 | 1-2 个月 | 优化 .ini 搜索算法、增强参数解析准确率、完善 Bug 搜索 |
| 阶段2: 智能化升级 | 3-6 个月 | AI 辅助参数预测、自动代码审查、智能 Bug 定位、PDF 自动提取 |
| 阶段3: 平台扩展 | 6-12 个月 | 支持 CXP 接口、跨框架移植、Web 可视化界面 |
| 阶段4: 生态系统 | 12 个月+ | 插件系统、开发者社区、标准化输出 |

## 相关链接与概念

- [[camera-isp]] -- Camera ISP 技术体系
- [[sensor-driver-development]] -- Sensor 驱动开发流程
- [[industrial-camera]] -- 工业相机系统
- [[MVDF-Drv]] -- 工业相机驱动框架
- [[SmartSens-SC-series]] -- SmartSens SC 系列传感器
- [[GigE-vs-USB-interface]] -- GigE 与 USB 接口差异
- [[LVDS-communication]] -- LVDS 通信协议
- [[SPI-register-programming]] -- SPI 寄存器编程
- [[knowledge-driven-development]] -- 知识驱动开发方法论
- [[YAML-structured-knowledge]] -- YAML 结构化知识表示

## 项目文件路径

- 项目汇报文档: `/mnt/c/Users/lijian/workspace/work/项目汇报文档_sc-sensor-driver.md`
- PPT 生成脚本: `/mnt/c/Users/lijian/workspace/work/create_presentation.js`
- 生成的 PPT: `/mnt/c/Users/lijian/workspace/work/SC-Sensor-Driver项目汇报.pptx`

## 相关笔记

- [[skill]] — GenICam 属性管理与专家系统
- [[myskills]] — CamSkills 工业相机固件开发技能包
- [[camera-diag-skills]] — Camera 诊断技能
- [[rv1126b]] — RV1126B 运动相机项目
- [[ok1126b-sdk]] — OK1126B SDK 与项目知识库
