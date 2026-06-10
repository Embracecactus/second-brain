---
tags:
  - douyin
  - crawler
  - whisper
  - ffmpeg
  - nlp
  - style-analysis
  - claude-code-skill
category: content/douyin
created: 2026-06-09
updated: 2026-06-09
status: active
---

# 抖音视频爬虫与文案风格分析

## 项目概述

抖音创作者的 Claude Code 技能包，包含两个核心 Skill：口播文案生成器 `/zhuzhuxu-writer` 和视频爬取分析管线 `/douyin-crawler`。通过爬取抖音视频、语音转录、风格量化分析，自动生成符合博主个人风格的口播文案。

## 技术栈

- **视频下载**: F2 (抖音视频下载器)
- **音频提取**: FFmpeg (16kHz mono WAV)
- **语音转录**: faster-whisper (large-v3, GPU优先, CPU降级)
- **字幕处理**: SRT 格式转换
- **风格分析**: Python NLP (词频、二元组、内容分类、话题聚类)
- **文案生成**: Claude Code Skill (基于273篇真实文案分析)
- **依赖**: Python 3.10+

## 两个 Skill

### `/zhuzhuxu-writer` — 口播文案生成器

基于 273 篇真实口播文案（136个SRT原始字幕 + 170个ASR转录）的风格分析。

**功能：**
- 给定主题，自动生成完整口播文案（300-500字）
- 5个备选标题（好奇心缺口、反常识、痛点共鸣等6种策略）
- 配图指导（封面图 + 正文配图 prompt）
- 标签策略（核心标签 + 按内容类型标签）

**支持内容类型：** 技术教程、工具分享、学习日记、日常分享、项目展示

### `/douyin-crawler` — 视频爬取与风格分析管线

给定抖音账号主页 URL，执行完整的 7 阶段管线：

| 阶段 | 功能 | 工具 |
|------|------|------|
| Phase 1 | Cookie 管理（自动检查、验证、引导刷新） | F2 |
| Phase 2 | F2 批量下载全部视频 | F2 |
| Phase 3 | FFmpeg 批量提取音频 | FFmpeg |
| Phase 4 | faster-whisper GPU 批量语音转录 | faster-whisper |
| Phase 5 | SRT 字幕转换 | Python |
| Phase 6 | 风格量化分析 | Python NLP |
| Phase 7 | 自动生成专属写作 Skill | Claude Code |

## 风格分析维度

- **篇幅统计**: 平均字数、句数、段落数
- **高频短语**: 词频分析、二元组(bigram)提取
- **句式模式**: 疑问句、感叹句、陈述句比例
- **内容分类**: 技术教程/工具分享/学习日记/日常分享/项目展示
- **话题聚类**: 关键词提取和主题归类

## 目录结构

```
douyin-creator-skills/
├── README.md
└── skills/
    ├── zhuzhuxu-writer/
    │   ├── SKILL.md           # 主 Skill 文件
    │   └── references/
    │       ├── vocabulary.md  # 个人词表（高频词、过渡词、禁用词）
    │       ├── style-examples.md  # 10篇代表性真实文案
    │       └── templates.md   # 5种内容类型填空模板
    └── douyin-crawler/
        ├── SKILL.md           # 主 Skill 文件
        └── references/
            ├── extract_audio.sh    # FFmpeg批量音频提取
            ├── transcribe.py       # faster-whisper GPU转录
            ├── convert_srt.py      # SRT字幕转文本
            ├── analyze_style.py    # 风格量化分析
            └── skill-generator-guide.md  # 写作Skill生成指南
```

## 安装使用

```bash
# 复制到 Claude Code skills 目录
cp -r skills/zhuzhuxu-writer ~/.claude/skills/
cp -r skills/douyin-crawler ~/.claude/skills/

# 使用
/zhuzhuxu-writer Docker容器化开发环境介绍-工具分享
/douyin-crawler
```

## 关键学习收获

1. **风格量化**: 通过词频和句式分析可精确描述写作风格
2. **Cookie管理**: 抖音爬虫需要定期刷新session cookie
3. **SRT vs ASR**: SRT字幕质量优于ASR转录，但不是所有视频都有
4. **GPU降级**: faster-whisper在无GPU时自动降级到CPU，速度慢10倍
5. **反AI味词汇**: 需维护禁用词列表避免AI生成内容过于机械

## 相关笔记

- [[selfMedia]] — 猪猪猪序员自媒体内容制作
- [[douyin-creator-skills]] — 抖音创作者AI技能集合
- [[zhuzhuxu-style]] — 猪猪猪序员口播文案风格
- [[ffmpeg]] — FFmpeg 多媒体处理框架
