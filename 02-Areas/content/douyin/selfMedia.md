---
tags: [douyin, self-media, video, whisper]
category: content/douyin
created: 2026-06-09
---
# selfMedia - 猪猪猪序员自媒体内容制作项目

## 项目概述

抖音博主"猪猪猪序员"（嵌入式程序猿 + 猪猪侠粉丝）的自媒体内容制作全流程管理系统，涵盖口播文案创作、风格分析、B-Roll 自动生成、视频后期编辑等环节。项目基于 Python + FFmpeg 技术栈，集成了 AI 辅助文案生成、语音转录、关键词提取、图片生成与视频合成等自动化管线。

## 目录结构

```
selfMedia/
├── CLAUDE.md                    # Claude Code 项目指引
├── README.md                    # 项目说明
├── 口播文案/                     # 按日期组织的视频项目
│   ├── 2026-06-03/              # deb包制作教程-技术教程
│   ├── 2026-06-04/              # ppt-master使用分享-工具分享
│   ├── 2026-06-05/              # superpower介绍与使用-工具分享
│   └── 2026-06-08/              # workflows多agent编排, skills-cleaner插件介绍
├── data/
│   ├── analyze_style.py         # 口播风格量化分析脚本
│   ├── style_analysis.json      # 风格分析原始数据
│   ├── style_analysis_report.md # 风格分析报告（273篇）
│   ├── transcribe.py            # faster-whisper 批量语音转录
│   ├── convert_srt.py           # SRT 字幕格式转换
│   └── extract_audio.sh         # 音频提取脚本
├── scripts/
│   ├── broll_pipeline.py        # B-Roll 完整自动化管线（支持 AI 图片生成）
│   └── broll_generator.py       # B-Roll 生成器（简化版）
├── video-use/                   # 视频编辑 Skill（browser-use/video-use）
│   ├── SKILL.md                 # Skill 定义与使用指南
│   ├── install.md               # 安装说明
│   ├── pyproject.toml           # Python 依赖配置
│   └── helpers/                 # 转录、渲染、调色等辅助脚本
├── logs/                        # F2 工具运行日志
└── douyin_users.db              # 抖音用户数据库
```

## 技术栈

- **语言**：Python 3.10+, Bash
- **视频处理**：FFmpeg, FFprobe
- **语音识别**：faster-whisper (large-v3, GPU/CPU), ElevenLabs Scribe (video-use)
- **NLP**：jieba 分词与关键词提取
- **图片生成**：OpenAI DALL-E (gpt-image-2), Stable Diffusion (本地), FFmpeg 占位图
- **视频编辑**：video-use (browser-use/video-use), 剪映
- **动画生成**：Manim, HyperFrames, Remotion, PIL + PNG 序列
- **依赖管理**：pyproject.toml, uv / pip
- **字幕处理**：pysrt, SRT 格式

## 架构与关键设计决策

### 1. 内容创作工作流

```
确定选题 → 写设计文档(spec) → 生成文案(zhuzhuxu-writer) → 制作视觉资源 → 录制视频 → 剪映粗剪 → video-use 精剪(动画/调色/字幕)
```

每个视频项目固定目录结构：`文案.md` + `images/` + `videos/` + `examples/`（可选）。

### 2. 口播风格量化分析

基于 273 篇真实文案（133 篇 SRT 原始字幕 + 166 篇 ASR 转录），通过正则匹配 + jieba 分词实现风格量化：

- 平均 385 字/篇，中位数 356 字
- 高频口头禅：今天(65.6%), 然后(56.0%), 分享(35.5%), 其实(34.1%)
- 结构化标志：第一步(23.8%) → 第二步(19.8%) → 第三步(16.1%)
- 内容类型：技术教程(168), 工具分享(146), 项目展示(138), 学习日记(128)

### 3. B-Roll 自动化管线

四级管线设计：SRT 解析 → 关键词提取 → AI 图片生成 → 视频合成

```python
# 核心流程 (broll_pipeline.py)
process_video(video, srt, output):
    1. SRTAnalyzer: 解析字幕 + 合并短段落(>=3s)
    2. extract_keywords: jieba TF-IDF 提取 Top 3 关键词
    3. BrollGenerator.generate: 三选一生成策略
       - PlaceholderGenerator: FFmpeg 绘制占位图
       - OpenAIGenerator: DALL-E API (gpt-image-2)
       - LocalSDGenerator: 本地 Stable Diffusion
    4. VideoCompositor: 分批(每批4张) FFmpeg overlay 合成
```

关键设计：分批处理避免 FFmpeg 参数过多，每批最多 4 张图片级联合成。

### 4. video-use 视频编辑 Skill

采用"文本优先 + 按需视觉"的编辑范式：

```
Transcribe → Pack → LLM 推理 → EDL → Render → Self-Eval
                                               └→ 发现问题则修复 + 重新渲染（最多3轮）
```

12 条硬规则保证生产正确性，其中最关键的：
- 字幕必须在滤镜链最后应用（否则被覆盖遮挡）
- 分段提取 + 无损拼接（避免重复编码）
- 每个切割边界 30ms 音频淡入淡出（消除爆音）
- 永远不在词语中间切割

## 关键代码片段

### 语音转录（GPU 优先降级策略）

```python
# data/transcribe.py
try:
    model = WhisperModel("large-v3", device="cuda", compute_type="float16")
except Exception:
    try:
        model = WhisperModel("medium", device="cuda", compute_type="float16")
    except Exception:
        model = WhisperModel("medium", device="cpu", compute_type="int8")
```

### 关键词提取（中英双语停用词过滤）

```python
# scripts/broll_pipeline.py - SRTAnalyzer.extract_keywords
keywords = jieba.analyse.extract_tags(text, topK=top_k * 2)
filtered = [kw for kw in keywords
            if kw.lower() not in en_stop_words
            and kw not in cn_stop_words
            and len(kw) > 1]
```

### FFmpeg 分批 overlay 合成

```python
# 每批最多4张，避免 FFmpeg filter_complex 参数爆炸
batch_size = 4
batches = [valid_images[i:i+batch_size] for i in range(0, len(valid_images), batch_size)]
```

## 口播文案风格指南

| 维度 | 规范 |
|------|------|
| 开场 | 亲切自然，如"哈喽大家好"、"好久不见" |
| 类比 | 生活化比喻解释技术（如"deb包就像Windows的.exe文件"） |
| 节奏 | 短句为主，适度口语化（"是不是很香？"、"一条命令就能搞定"） |
| 结构 | 分步骤讲解，通常 3-4 步，每步有明确标题 |
| 结尾 | 行动号召 + 互动引导（"赶紧动手试试吧"、"评论区留言"） |
| 时长 | 1-3 分钟，语速约 200-250 字/分钟 |
| 标签 | #嵌入式 #程序员 #单片机 #学习日记 #猪猪猪序员 |

## 构建与运行

```bash
# B-Roll 管线
python3 scripts/broll_pipeline.py data/videos/6月8日/6月8日.mp4
python3 scripts/broll_pipeline.py data/videos/6月8日/6月8日.mp4 --api openai

# 风格分析
python3 data/analyze_style.py

# 语音转录（需 CUDA 或 CPU）
python3 data/transcribe.py

# video-use 安装
cd ~/Developer/video-use && uv sync
ln -sfn ~/Developer/video-use ~/.claude/skills/video-use
```

## 关键学习与洞察

1. **风格量化驱动文案生成**：通过对 273 篇真实文案的量化分析（高频词、句式结构、开场/结尾模式），构建了 `/zhuzhuxu-writer` Skill，实现风格一致性
2. **AI 增强而非替代**：文案创作保持人工审稿，AI 负责风格对齐和初稿生成；B-Roll 图片生成支持三种后端灵活切换
3. **video-use 的"文本优先"范式**：LLM 不直接看视频，而是读转录文本 + 按需查看视觉合成图，将 30000 帧的视觉信息压缩为 12KB 文本 + 少量 PNG
4. **分批处理策略**：FFmpeg overlay 合成采用分批（每批 4 张）避免参数过多导致失败，同时支持断点续处理
5. **内容类型模板化**：技术教程、工具分享、学习日记、项目展示、踩坑记录、开箱评测六种类型各有固定结构模板

## 相关概念

- [[Claude Code Skills]] - Skill 注册与使用机制
- [[FFmpeg]] - 视频处理核心工具
- [[faster-whisper]] - 语音识别引擎
- [[jieba]] - 中文分词库
- [[Manim]] - 数学动画生成
- [[ElevenLabs Scribe]] - 词级时间戳语音转录
- [[DALL-E]] - AI 图片生成
- [[Stable Diffusion]] - 本地图片生成
- [[剪映]] - 视频剪辑工具
- [[抖音内容创作]] - 短视频平台运营

## 相关笔记

- [[douyin-creator-skills]] — 抖音创作者技能集合
- [[douyin-crawler]] — 抖音爬虫与文案分析
- [[zhuzhuxu-style]] — 猪猪猪序员口播文案生成器
- [[redbook]] — 小红书技术内容创作素材库
- [[embedded-blog]] — 嵌入式技术博客
- [[ffmpeg]] — FFmpeg 多媒体处理框架
