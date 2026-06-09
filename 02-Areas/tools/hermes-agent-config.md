---
title: Hermes Agent Configuration
date: 2026-06-09
tags:
  - hermes
  - ai-agent
  - configuration
  - llm
aliases:
  - Hermes配置
  - Hermes Agent 设置
---

## 概述

Hermes 是一个多平台 AI Agent 框架，支持 CLI、Telegram、Discord、WhatsApp、Slack、Signal、Home Assistant 等多种接入方式。配置文件位于 `/home/lijian/.hermes/config.yaml`，采用 YAML 格式，当前版本为 `_config_version: 14`。

## 核心配置要点

### 模型与提供商 (Model & Providers)

| 配置项 | 当前值 | 说明 |
|--------|--------|------|
| `model.default` | `anthropic/claude-opus-4.6` | 默认主模型 |
| `model.provider` | `auto` | 自动选择 provider |
| `model.base_url` | `https://openrouter.ai/api/v1` | 通过 OpenRouter 路由 |
| `fallback_providers` | `[]` | 未配置 fallback provider |
| `smart_model_routing` | `enabled: false` | 未启用智能路由 |

支持的 fallback provider 包括：`openrouter`、`openai-codex`、`nous`、`zai`、`kimi-coding`、`minimax`、`minimax-cn`。

### Agent 行为

| 配置项 | 当前值 | 说明 |
|--------|--------|------|
| `agent.max_turns` | `90` | 单次会话最大轮次 |
| `agent.gateway_timeout` | `1800` (30min) | 网关超时时间 |
| `agent.gateway_timeout_warning` | `900` (15min) | 超时警告阈值 |
| `agent.reasoning_effort` | `medium` | 推理强度 |
| `agent.tool_use_enforcement` | `auto` | 工具使用策略 |
| `agent.verbose` | `false` | 非详细模式 |

### 个性化 (Personalities)

预设了 13 种人格模式，当前激活 `kawaii`：

- `helpful` -- 友好助手
- `concise` -- 简洁模式
- `technical` -- 技术专家
- `creative` -- 创意助手
- `teacher` -- 耐心教学
- `kawaii` -- 可爱风格 (当前激活)
- `catgirl` -- 猫娘 Neko-chan
- `pirate` -- 海盗船长
- `shakespeare` -- 莎士比亚风格
- `surfer` -- 冲浪者
- `noir` -- 黑色电影风格
- `uwu` -- UwU 卖萌
- `philosopher` -- 哲学家
- `hype` -- 热血激情

### 终端与容器 (Terminal & Containers)

| 配置项 | 当前值 | 说明 |
|--------|--------|------|
| `terminal.backend` | `local` | 使用本地终端 |
| `terminal.modal_mode` | `auto` | 自动模态模式 |
| `terminal.timeout` | `180` | 命令超时 3 分钟 |
| `terminal.persistent_shell` | `true` | 持久化 shell |
| `terminal.docker_image` | `nikolaik/python-nodejs:python3.11-nodejs20` | 默认 Docker 镜像 |
| `terminal.container_cpu` | `1` | 容器 CPU 核心 |
| `terminal.container_memory` | `5120` (5GB) | 容器内存 |
| `terminal.container_disk` | `51200` (50GB) | 容器磁盘 |

### 辅助服务 (Auxiliary Services)

各子服务均配置为 `provider: auto`，独立超时设置：

| 服务 | 超时 | 用途 |
|------|------|------|
| `vision` | 30s | 图像识别 |
| `web_extract` | 360s | 网页内容提取 |
| `compression` | 120s | 上下文压缩 |
| `session_search` | 30s | 会话搜索 |
| `skills_hub` | 30s | 技能中心 |
| `approval` | 30s | 审批流程 |
| `mcp` | 30s | MCP 工具 |
| `flush_memories` | 30s | 记忆刷新 |

### 上下文压缩 (Compression)

| 配置项 | 当前值 | 说明 |
|--------|--------|------|
| `compression.enabled` | `true` | 启用压缩 |
| `compression.threshold` | `0.5` | 触发压缩阈值 |
| `compression.target_ratio` | `0.2` | 压缩目标比率 |
| `compression.protect_last_n` | `20` | 保护最近 20 条消息 |
| `compression.summary_model` | `google/gemini-3-flash-preview` | 使用 Gemini 3 Flash 做摘要 |

### 语音交互 (Voice & TTS/STT)

- **TTS Provider**: Edge TTS (`en-US-AriaNeural`)，支持 ElevenLabs、OpenAI、NeuTTS
- **STT Provider**: Local Whisper (`base` 模型)，支持 OpenAI Whisper、Mistral Voxtral
- **录音快捷键**: `Ctrl+B`，最大录音 120 秒

### 安全设置 (Security)

| 配置项 | 当前值 | 说明 |
|--------|--------|------|
| `privacy.redact_pii` | `false` | 未启用 PII 脱敏 |
| `security.redact_secrets` | `true` | 密钥脱敏已启用 |
| `security.tirith_enabled` | `true` | Tirith 安全策略已启用 |
| `security.tirith_fail_open` | `true` | 安全检查失败时放行 |
| `approvals.mode` | `manual` | 人工审批模式 |

### 记忆系统 (Memory)

| 配置项 | 当前值 | 说明 |
|--------|--------|------|
| `memory.memory_enabled` | `true` | 记忆功能已启用 |
| `memory.user_profile_enabled` | `true` | 用户画像已启用 |
| `memory.memory_char_limit` | `2200` | 记忆字符上限 |
| `memory.user_char_limit` | `1375` | 用户信息字符上限 |
| `memory.nudge_interval` | `10` | 每 10 轮提醒更新记忆 |
| `memory.flush_min_turns` | `6` | 最少 6 轮后刷新 |

### 平台工具集 (Platform Toolsets)

| 平台 | 工具集 |
|------|--------|
| CLI | `hermes-cli` |
| Telegram | `hermes-telegram` |
| Discord | `hermes-discord` |
| WhatsApp | `hermes-whatsapp` |
| Slack | `hermes-slack` |
| Signal | `hermes-signal` |
| Home Assistant | `hermes-homeassistant` |

### 会话管理 (Session)

| 配置项 | 当前值 | 说明 |
|--------|--------|------|
| `session_reset.mode` | `both` | 空闲 + 定时双重重置 |
| `session_reset.idle_minutes` | `1440` (24h) | 空闲 24 小时后重置 |
| `session_reset.at_hour` | `4` | 每天凌晨 4 点重置 |
| `checkpoints.enabled` | `true` | 启用快照 |
| `checkpoints.max_snapshots` | `50` | 最多 50 个快照 |

## 目录结构

```
~/.hermes/
├── config.yaml          # 主配置文件
├── SOUL.md              # Agent 人格定义 (当前为空，使用默认)
├── .env                 # 环境变量 (API Keys)
├── skills/              # 26 个技能模块
│   ├── software-development/
│   ├── research/
│   ├── productivity/
│   ├── mcp/
│   ├── note-taking/
│   └── ...
├── memories/            # 持久化记忆
├── sessions/            # 会话历史
├── hooks/               # 生命周期钩子
├── cron/                # 定时任务
├── logs/                # 运行日志
├── audio_cache/         # 音频缓存
├── image_cache/         # 图片缓存
├── pairing/             # 配对设备
└── whatsapp/            # WhatsApp 集成
```

## 备注

- `.env` 文件存储 API Keys（OpenRouter、ZAI、Kimi 等），权限 `600`
- `SOUL.md` 为空文件，Agent 使用 config.yaml 中的 `personality` 设置
- 26 个 skill 目录表明该实例功能覆盖面广
- `streaming.enabled: false` 但 `display.streaming: true`，两处配置不一致
- Smart Model Routing 可通过 `smart_model_routing` 开启，将简单请求路由到廉价模型（如 Gemini 2.5 Flash）
