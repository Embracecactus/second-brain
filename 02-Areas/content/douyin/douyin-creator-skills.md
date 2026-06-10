---
tags:
  - 抖音
  - AI技能
  - Claude-Code
  - 口播文案
  - 视频处理
  - 风格分析
  - Manim
  - 自动化工作流
  - 创作者工具
category: 技术项目
created: 2026-06-10
updated: 2026-06-10
status: active
---

# 抖音创作者AI技能集合

## 项目概述

`douyin-creator-skills` 是一套面向抖音内容创作者的 Claude Code Skill 技能包，旨在通过 AI 自动化解决短视频创作中的核心痛点：口播文案撰写、竞品风格分析和技术动画制作。该项目包含三个核心 Skill，覆盖了从内容研究（爬取竞品视频并量化分析口播风格）、文案生产（基于 273 篇真实口播文案提炼的风格模型自动生成符合博主个人风格的脚本）、到视觉增强（利用 Manim 自动生成技术概念动画并叠加到口播视频）的完整创作链路。整套工具以"AI 辅助创作、人类把控质量"为理念，将传统需要数小时的文案撰写和动画制作工作压缩到分钟级别，同时通过严格的质量检查清单确保输出风格的一致性和专业性。项目采用 MIT 开源协议，依赖 F2、FFmpeg、faster-whisper、Manim 等开源工具，可在配备 NVIDIA GPU 的 Linux/WSL2 环境中高效运行。

## 技术栈

| 层级 | 技术/工具 | 用途 |
|------|-----------|------|
| AI 运行时 | Claude Code | Skill 执行引擎，承载所有 Skill 的调度与生成逻辑 |
| 视频下载 | F2 (Python) | 抖音视频批量下载，支持 Cookie 认证、断点续传 |
| 音频处理 | FFmpeg | 批量从 MP4 提取 WAV 音频 |
| 语音转录 | faster-whisper (large-v3) | GPU 加速的语音识别，支持中文 ASR |
| 字幕处理 | Python (自定义脚本) | SRT 字幕格式转纯文本 |
| 风格分析 | Python (量化分析脚本) | 篇幅统计、短语频率、句式分析、话题聚类 |
| 动画生成 | Manim (Python) | 技术概念可视化动画，输出 MP4 |
| 视频编辑 | 剪映 | 最终视频合成，叠加动画层 |
| GPU 加速 | NVIDIA CUDA + cuBLAS | faster-whisper 模型推理加速 |
| 脚本语言 | Python 3.10+、Bash | 数据处理管线脚本 |

## 架构与设计

### 整体架构

项目采用 **Skill 即插件** 的架构模式，每个 Skill 是一个独立目录，包含 `SKILL.md`（主定义文件）和 `references/`（参考资源）。所有 Skill 安装到 `~/.claude/skills/` 目录后，由 Claude Code 统一调度。

```
创作流程管线：
竞品研究 → 风格分析 → 文案生成 → 动画制作 → 视频合成
(douyin-crawler)  ↓      (zhuzhuxu-writer)  (video-enhancer)    (剪映)
              style_analysis.json ──→ 风格模型 ──→ 动画匹配
```

### Skill 协作关系

三个 Skill 形成完整的创作流水线：`/douyin-crawler` 负责上游的数据采集和风格建模，其输出的 `style_analysis.json` 可被 `/zhuzhuxu-writer` 消费以生成更精准的文案；`/video-enhancer` 则读取文案内容，识别需要可视化的技术概念并生成对应动画；最终用户在剪映中完成合成。这种松耦合设计允许单独使用任一 Skill，也可以组合成完整管线。

### 数据流设计

`douyin-crawler` 采用 Phase 分阶段执行模型，每个阶段独立可验证：

```
Phase 0: 初始化（创建目录结构）
Phase 1: 环境检查（Python/F2/FFmpeg/faster-whisper/CUDA）
Phase 2: Cookie 管理（检查→验证→刷新）
Phase 3: F2 批量下载视频 → videos/
Phase 4: FFmpeg 提取音频 → audio/
Phase 5: faster-whisper 转录 → transcripts/
Phase 6: SRT 字幕转换 → srt_originals/
Phase 7: 风格量化分析 → analysis/
Phase 8: 生成专属写作 Skill
```

## 核心知识点

### 1. 口播文案风格建模

`zhuzhuxu-writer` 的核心是一套从 273 篇真实口播文案（136 个 SRT 原始字幕 + 170 个 ASR 转录）中提炼的量化风格模型。关键参数：

- **目标字数**：300-500 字（均值 385 字）
- **语速**：200-250 字/分钟
- **句长**：每句 10-25 字，短句为主
- **步骤数**：教程类 3-4 步（80% 案例使用）
- **内容类型分布**：技术教程 ~60%、工具分享 ~15%、学习日记 ~10%、日常分享 ~15%

**口语高频词表**（按频率排序）：

| 词汇 | 使用频率 | 场景 |
|------|---------|------|
| 今天 | 65% | 开场引入 |
| 然后 | 56% | 句间过渡 |
| 分享 | 35% | 引出内容 |
| 其实 | 34% | 解释前铺垫 |
| 最近 | 25% | 开场时间锚 |
| 特别 | 20% | 强调感受 |

**类比技法**是该风格的核心特征——遇到任何技术概念都必须映射到日常生活经验：

```
模板："XX就像Windows里的XX文件"
实例："deb包就像是Windows的.exe文件"
实例："静态库想象成一份预制菜"
实例："control文件就像是deb包的身份证"
```

### 2. 标题策略体系

从真实数据中提炼的 6 种标题策略：

| 策略 | 示例 | 适用场景 |
|------|------|---------|
| 好奇心缺口 | "用了XX，效率直接翻倍" | 工具分享 |
| 反常识 | "XX还能这样用？" | 技术教程 |
| 痛点共鸣 | "AI写代码总差点意思？问题出在这" | 学习日记 |
| 数据冲击 | "一条命令搞定XX" | 技术教程 |
| 社交货币 | "程序员必备：XX" | 工具分享 |
| 真人故事 | "嵌入式程序员的XX之路" | 日常分享 |

### 3. Manim 技术动画模板体系

`video-enhancer` 定义了 6 种标准化动画类型，覆盖技术内容可视化的主要场景：

**概念对比动画**——最常用的模板，用于"单 vs 多""旧 vs 新"等对比场景：

```python
from manim import *
BG = "#0A0A0A"
PRIMARY = "#00F5FF"
SECONDARY = "#FF00FF"
ACCENT = "#39FF14"
MONO = "WenQuanYi Zen Hei"

class ConceptComparison(Scene):
    def construct(self):
        self.camera.background_color = BG
        title = Text("对比标题", font_size=36, color=WHITE, font=MONO)
        title.to_edge(UP, buff=0.5)
        self.play(Write(title))
        # 左侧方案A、右侧方案B，使用 RoundedRectangle + Text 组合
        left_box = RoundedRectangle(width=5, height=3, color=RED, fill_opacity=0.2)
        left_box.shift(LEFT * 3.5)
        # ... 动画序列
```

**关键设计规则**——文字必须跟随框一起排列：

```python
# 正确做法：每个元素是框+文字的 VGroup 组合
tasks = VGroup()
for i in range(5):
    rect = RoundedRectangle(...)
    label = Text(...)
    label.move_to(rect)          # 文字先居中到框内
    task_group = VGroup(rect, label)  # 组合成一个单元
    tasks.add(task_group)
tasks.arrange(RIGHT, buff=0.3)   # 整体排列，文字跟随框移动
```

原理：Manim 的 `arrange()` 只移动 VGroup 的直接子元素。如果框和文字是两个独立的 VGroup，`arrange()` 会分别移动它们导致位置错乱。

### 4. 色彩方案系统

三套预定义色彩方案适配不同场景：

```python
# Neon Tech（默认，技术感）
BG, PRIMARY, SECONDARY, ACCENT = "#0A0A0A", "#00F5FF", "#FF00FF", "#39FF14"

# Warm Tech（温暖，入门教程）
BG, PRIMARY, SECONDARY, ACCENT = "#1C1C1C", "#58C4DD", "#83C167", "#FFFF00"

# Dark Pro（专业，高级感）
BG, PRIMARY, SECONDARY, ACCENT = "#1A1A2E", "#E94560", "#0F3460", "#16213E"
```

### 5. faster-whisper GPU 转录管线

转录脚本实现了三级自动降级策略：

1. `large-v3` on CUDA + float16（需 ~3GB VRAM，最快）
2. `medium` on CUDA + float16（需 ~1.5GB VRAM）
3. `medium` on CPU + int8（最慢但保底）

关键环境配置——CUDA 库路径探测：

```bash
# 探测 libcublas.so 位置
find ~/.local/lib -name "libcublas.so*" 2>/dev/null
# 设置环境变量
export LD_LIBRARY_PATH={探测到的路径}:$LD_LIBRARY_PATH
```

### 6. F2 Cookie 认证机制

F2 下载抖音视频需要有效的登录 Cookie。验证流程：

```bash
# 验证 Cookie 有效性（尝试下载 1 个视频）
f2 dy -m post -u "{用户URL}" -o 1 -p /tmp/cookie_test --max-tasks 1
```

Cookie 配置存储在 `~/.f2/app.yaml`，需包含 `sessionid` 字段。过期时通过浏览器 F12 → Network → 复制 Cookie header 值进行刷新。

### 7. 内容分析与动画映射规则

`video-enhancer` 定义了文案信号到动画类型的映射关系：

| 文案信号 | 动画类型 | 示例 |
|---------|---------|------|
| "A就像B" | 类比可视化 | "deb包就像Windows的exe" |
| "第一步...第二步..." | 流程步骤 | 安装步骤、操作流程 |
| "A和B的区别" | 对比动画 | 单agent vs 多agent |
| "底层原理" | 架构图 | 系统架构、模块关系 |
| "从X变成Y" | 变换动画 | 状态变化、流程转换 |

不需要动画的情况：纯观点输出、情感表达、简单事实陈述、已有实操录屏的内容。

### 8. 渲染与拼接

```bash
# 预览（854x480，快速）
manim -ql script.py SceneName
# 正式（1920x1080）
manim -qh script.py SceneName
# 4K（3840x2160）
manim -qk script.py SceneName

# 多场景拼接
ffmpeg -y -f concat -safe 0 -i concat.txt -c copy all_scenes.mp4
```

叠加到口播视频时，使用深色背景（`BG = "#0A0A0A"`），在剪映中用「滤色」混合模式去除黑底。

## 构建/使用步骤

### 安装 Skill

```bash
# 方式一：直接复制
cp -r skills/zhuzhuxu-writer ~/.claude/skills/
cp -r skills/video-enhancer ~/.claude/skills/
cp -r skills/douyin-crawler ~/.claude/skills/

# 方式二：软链接（方便后续 git pull 更新）
ln -s /path/to/douyin-creator-skills/skills/zhuzhuxu-writer ~/.claude/skills/zhuzhuxu-writer
ln -s /path/to/douyin-creator-skills/skills/video-enhancer ~/.claude/skills/video-enhancer
ln -s /path/to/douyin-creator-skills/skills/douyin-crawler ~/.claude/skills/douyin-crawler
```

### 安装 douyin-crawler 依赖

```bash
pip3 install -U f2 faster-whisper
sudo apt install ffmpeg
# 确认 NVIDIA GPU 驱动
nvidia-smi
```

### 使用示例

```bash
# 生成口播文案
/zhuzhuxu-writer Docker容器化开发环境介绍-工具分享

# 生成技术动画（基于文案）
/video-enhancer

# 爬取抖音账号并分析风格
/douyin-crawler
```

### 目录结构

```
skills/
  zhuzhuxu-writer/
    SKILL.md                    # 主 Skill 文件
    references/
      vocabulary.md             # 个人词表（高频词、过渡词、禁用词）
      style-examples.md         # 10 篇代表性真实文案
      templates.md              # 5 种内容类型填空模板
  video-enhancer/
    SKILL.md                    # 主 Skill 文件（Manim 动画生成指南）
    references/
      manim-templates.md        # Manim 动画模板
      ffmpeg-recipes.md         # FFmpeg 命令参考
  douyin-crawler/
    SKILL.md                    # 主 Skill 文件（完整工作流指南）
    references/
      extract_audio.sh          # FFmpeg 批量音频提取脚本模板
      transcribe.py             # faster-whisper GPU 转录脚本模板
      convert_srt.py            # SRT 字幕转文本脚本模板
      analyze_style.py          # 风格量化分析脚本模板
      skill-generator-guide.md  # 写作 Skill 生成指南
```

## 关键学习收获

1. **风格建模方法论**：通过量化分析大量真实文案（字数分布、高频词、句式模式、开场/结尾模式），可以构建可复用的个人写作风格模型。273 篇样本足以提炼出稳定的风格参数。

2. **Claude Code Skill 架构模式**：Skill = `SKILL.md`（指令定义） + `references/`（参考资源），安装到 `~/.claude/skills/` 即可被 Claude Code 识别和触发。这是一种轻量级的 AI 能力扩展机制。

3. **Manim 动画的 VGroup 排列陷阱**：`arrange()` 只移动 VGroup 的直接子元素，文字和形状必须先组合成一个 VGroup 单元再整体排列，否则会导致位置错乱。这是 Manim 开发中的高频坑点。

4. **GPU 降级策略**：AI 推理管线应设计多级降级（large→medium→GPU→CPU），确保在不同硬件环境下都能完成任务，只是速度差异。CUDA 库路径的动态探测是 WSL2 环境下的关键步骤。

5. **数据驱动的内容创作**：从"凭感觉写文案"到"基于数据模型生成文案"的范式转变。高频词、类比模板、开场模式等都从真实数据中提炼，而非主观臆断。

6. **管线化思维**：将复杂的创作流程拆分为独立可验证的 Phase（下载→提取→转录→分析→生成），每个阶段有明确的输入输出和验证方法，支持断点续传和故障排查。

## 相关笔记

- [[Claude Code Skill 开发指南]]
- [[Manim 动画制作入门与进阶]]
- [[短视频口播文案写作技巧]]
- [[faster-whisper 语音识别实践]]
- [[FFmpeg 音视频处理常用命令]]
- [[抖音创作者工具链生态]]
