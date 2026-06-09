---
tags: [douyin, crawler, whisper]
category: content/douyin
created: 2026-06-09
---
Obsidian note generated at `/home/lijian/project/Obsidian/content/douyin/douyin-crawler-skill.md`.

Extracted knowledge from 6 files:
- `SKILL.md` -- main skill definition with full 8-phase pipeline
- `references/extract_audio.sh` -- FFmpeg batch audio extraction (16kHz mono WAV)
- `references/transcribe.py` -- faster-whisper batch ASR with 3-tier GPU/CPU fallback
- `references/convert_srt.py` -- SRT subtitle to plain text converter
- `references/analyze_style.py` -- quantitative style analysis (phrase frequency, bigrams, content type classification, topic extraction)
- `references/skill-generator-guide.md` -- writing skill generation spec (9 chapters, vocabulary, templates, examples)

## 相关笔记

- [[selfMedia]] — 猪猪猪序员自媒体内容制作项目
- [[douyin-creator-skills]] — 抖音创作者技能集合
- [[zhuzhuxu-style]] — 猪猪猪序员口播文案生成器
- [[ffmpeg]] — FFmpeg 多媒体处理框架
