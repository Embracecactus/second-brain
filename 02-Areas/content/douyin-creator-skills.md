---
title: douyin-creator-skills - 抖音创作者 Claude Code 技能包
tags:
  - douyin
  - claude-code
  - skill
  - content-creation
  - nlp
  - whisper
  - manim
  - ffmpeg
  - f2
  - style-analysis
category: content/douyin
created: 2026-06-09
status: active
project_path: /home/lijian/project/douyin-creator-skills
---

# douyin-creator-skills - 抖音创作者 Claude Code 技能包

## 项目概述

一套面向抖音内容创作者的 Claude Code Skill 集合，包含三个核心技能：口播文案自动生成器、视频爬取与风格分析管线、技术动画生成器。项目通过量化分析真实文案数据（273 篇口播文案），提炼出博主的个人风格特征（词汇、句式、节奏、类比技法），并将其编码为可复用的 Skill，实现"给定主题即可生成完全符合个人风格的口播文案"的目标。

## 技术栈

- **Claude Code Skill**: SKILL.md + references/ 目录结构，基于 frontmatter 触发词机制
- **F2**: 抖音视频批量下载工具（Python），需 Douyin Cookie 认证
- **FFmpeg**: 视频音频提取（PCM 16kHz 单声道 WAV）
- **faster-whisper**: OpenAI Whisper 的高性能 Python 实现，支持 GPU (CUDA float16) 和 CPU (int8) 降级
- **Python 3.10+**: 风格分析脚本（正则、Counter、JSON 输出）
- **Manim**: 数学动画引擎，用于生成技术概念可视化动画
- **FFmpeg concat**: 动画拼接与视频后处理

## 三个 Skill 架构

### 1. `/zhuzhuxu-writer` -- 口播文案生成器

基于 273 篇真实口播文案的风格分析，生成符合博主"猪猪猪序员"个人风格的完整口播文案。

**核心设计**:
- 人设内核: 嵌入式程序员 + 猪猪侠粉丝，谦逊好学、真诚自嘲
- 5 种内容类型模板: 技术教程 (60%)、工具分享 (15%)、学习日记 (10%)、日常分享 (15%)、项目展示
- 输出结构: 口播文案 + 5 个备选标题（6 种策略）+ 配图 prompt + 标签策略
- 质量检查: 10 项自检清单（字数 300-500、口语化、类比、反问互动等）

**关键风格规则**:
- 短句为主（10-25 字/句），口语化连接词（"其实""然后""说白了"）
- 技术概念必须映射到生活化类比（"deb 包就像 Windows 的 .exe 文件"）
- 绝对禁止学术腔、夸大宣传、说教语气、AI 味套话

### 2. `/douyin-crawler` -- 视频爬取与风格分析管线

给定抖音账号主页 URL，执行完整的 7 阶段管线:

```
Phase 0: 初始化（创建工作目录）
Phase 1: 环境检查（Python/F2/FFmpeg/faster-whisper/CUDA）
Phase 2: Cookie 管理（验证/刷新 Douyin 登录 Cookie）
Phase 3: F2 批量下载全部视频
Phase 4: FFmpeg 批量提取音频（16kHz mono WAV）
Phase 5: faster-whisper GPU 批量语音转录（large-v3 → medium → CPU 降级链）
Phase 6: SRT 字幕转换（可选）
Phase 7: 风格量化分析（篇幅/短语频率/句式/内容类型/话题聚类）
Phase 8: 自动生成专属写作 Skill
```

**脚本模板机制**: references/ 目录下存放 Python/Bash 脚本模板，包含 `{video_dir}` 等占位符，Skill 在运行时替换为实际路径后生成到工作目录执行。

### 3. `/video-enhancer` -- 技术动画生成器

根据口播文案内容，自动生成 Manim 技术动画视频，用户导入剪映叠加到口播视频上。

**6 种动画类型模板**:
- 概念对比（最常用）: 单 vs 多、旧 vs 新
- 流程步骤: 三步法、操作流程
- 架构图: 系统设计、模块关系
- 数据结构: 链表、树、图
- 图表对比: 性能对比、BarChart
- 代码高亮: 关键代码片段

**色彩方案**: Neon Tech（默认，深黑背景 + 青色/紫色/绿色）、Warm Tech、Dark Pro

## 关键架构设计决策

### 1. Skill 文件结构

采用 `SKILL.md` + `references/` 分离设计:
- `SKILL.md` 包含完整的 Skill 定义（frontmatter 触发词 + 9 个章节的写作规范）
- `references/` 存放详细的参考数据（词表、风格范例、模板、脚本模板）
- Claude 在首次生成前按需读取 references，避免一次性加载过多上下文

### 2. 风格量化分析管线

`analyze_style.py` 实现了从原始文案到量化指标的完整分析:
- 篇幅分布（平均/中位/最短/最长字数）
- 短语频率（30+ 追踪短语）
- 句式特征（疑问句/感叹句/平均句长）
- 内容类型分类（关键词匹配）
- 高频双字词（bigram Top 30）
- 话题关键词频率
- 开场/结尾样本提取

### 3. ASR 模型降级策略

transcribe.py 实现三级降级链，确保在不同硬件环境下都能完成转录:
1. `large-v3` on CUDA + float16（约 3GB VRAM，最快）
2. `medium` on CUDA + float16（约 1.5GB VRAM）
3. `medium` on CPU + int8（最慢但保底）

### 4. Manim 文字跟随排列规则

动画生成中一个关键陷阱: 框和文字必须组合成单个 VGroup 后再 arrange，否则 arrange() 会分别移动它们导致位置错乱。

## 关键经验与洞察

1. **风格迁移的核心是量化**: 不是让 AI "模仿风格"，而是从真实数据中提取可量化的指标（词频、句长、类比模式），再编码为规则
2. **Cookie 管理是爬虫的命门**: F2 需要有效的 Douyin 登录 Cookie，过期后需手动从浏览器 DevTools 刷新，流程自动化程度有限
3. **SRT 字幕 vs ASR 转录**: SRT 字幕（从剪辑软件导出）比 ASR 转录更准确，但不是所有视频都有；项目设计为两者可混合使用，以 first-50-chars 去重
4. **占位符模板模式**: references/ 中的脚本使用 `{xxx}` 占位符，Skill 运行时替换后生成到工作目录，这是一种轻量级的"代码生成"模式
5. **内容类型自动判断**: 通过关键词匹配而非机器学习来分类内容类型，简单但在 273 篇样本上效果足够好
6. **口语化 vs AI 味的对抗**: vocabulary.md 专门维护了"应该避免的表达"列表（正式用语替换表），这是防止 LLM 输出书面化的关键手段

## 代码片段: 风格分析核心逻辑

```python
# analyze_style.py - 高频双字词提取
chinese_only = re.sub(r"[^一-鿿]", "", all_text)
bigrams = [chinese_only[i:i + 2] for i in range(len(chinese_only) - 1)]
bigram_freq = Counter(bigrams).most_common(50)

# 内容类型分类（关键词匹配）
type_counts = {}
for ctype, keywords in content_types.items():
    count = sum(1 for t in texts.values() if any(kw in t for kw in keywords))
    type_counts[ctype] = count
```

## 代码片段: ASR 降级链

```python
# transcribe.py - GPU/CPU 自动降级
if device == "cuda":
    try:
        model = WhisperModel(model_name, device="cuda", compute_type="float16")
    except Exception:
        try:
            model = WhisperModel("medium", device="cuda", compute_type="float16")
        except Exception:
            model = WhisperModel("medium", device="cpu", compute_type="int8")
```

## 代码片段: FFmpeg 音频提取

```bash
# extract_audio.sh - 提取 16kHz 单声道 WAV（适合 Whisper 输入）
ffmpeg -i "$f" -vn -acodec pcm_s16le -ar 16000 -ac 1 "$OUT" -y
```

## 安装与使用

```bash
# 安装 Skill（软链接方式，方便 git pull 更新）
ln -s /home/lijian/project/douyin-creator-skills/skills/zhuzhuxu-writer ~/.claude/skills/zhuzhuxu-writer
ln -s /home/lijian/project/douyin-creator-skills/skills/douyin-crawler ~/.claude/skills/douyin-crawler
ln -s /home/lijian/project/douyin-creator-skills/skills/video-enhancer ~/.claude/skills/video-enhancer

# 生成口播文案
/zhuzhuxu-writer Docker容器化开发环境介绍-工具分享

# 爬取抖音账号并分析风格
/douyin-crawler

# 根据文案生成技术动画
/video-enhancer
```

### douyin-crawler 额外依赖

```bash
pip3 install f2 faster-whisper
sudo apt install ffmpeg
# 需要 NVIDIA GPU + CUDA（推荐），CPU 也可用但转录速度慢 10-20 倍
```

## 目录结构

```
douyin-creator-skills/
├── README.md
└── skills/
    ├── zhuzhuxu-writer/
    │   ├── SKILL.md                    # 口播文案生成器（9 章节完整规范）
    │   └── references/
    │       ├── vocabulary.md           # 个人词表（高频词 Top30、过渡词、禁用词）
    │       ├── style-examples.md       # 10 篇代表性真实文案
    │       └── templates.md            # 5 种内容类型填空模板
    ├── douyin-crawler/
    │   ├── SKILL.md                    # 视频爬取管线（7 阶段完整工作流）
    │   └── references/
    │       ├── extract_audio.sh        # FFmpeg 批量音频提取脚本模板
    │       ├── transcribe.py           # faster-whisper GPU 转录脚本模板
    │       ├── convert_srt.py          # SRT 字幕转文本脚本模板
    │       ├── analyze_style.py        # 风格量化分析脚本模板
    │       └── skill-generator-guide.md # 写作 Skill 生成指南
    └── video-enhancer/
        ├── SKILL.md                    # 技术动画生成器（6 种动画类型模板）
        └── references/
            ├── manim-templates.md      # Manim 动画模板库
            └── ffmpeg-recipes.md       # FFmpeg 命令参考
```

## 相关概念

- [[Claude Code]] - Anthropic 的 CLI 编程助手，支持 Skill 扩展机制
- [[抖音]] - 短视频平台，本项目的内容发布目标
- [[faster-whisper]] - OpenAI Whisper 的 CTranslate2 高性能实现
- [[Manim]] - 3Blue1Brown 开发的数学动画引擎
- [[FFmpeg]] - 音视频处理瑞士军刀
- [[F2]] - 抖音/TikTok 视频下载工具
- [[口播文案]] - 短视频中博主对着镜头讲述的文稿
- [[风格迁移]] - 将一种表达风格应用到新内容上的技术
