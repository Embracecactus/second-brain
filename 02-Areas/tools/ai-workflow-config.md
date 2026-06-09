---
tags: [ai, hermes, workflow, ai-agent, llm, multi-platform]
category: tools
created: 2026-06-09
updated: 2026-06-09
status: active
---

# Hermes Agent -- AI Workflow 配置与架构详解

## 项目/工具概述

Hermes Agent 是由 [Nous Research](https://nousresearch.com) 开发的**自学习型多平台 AI Agent 框架**，核心特色是内置闭环学习系统：Agent 可从经验中创建 Skill、在使用过程中自我改进、定期主动持久化记忆、跨会话检索历史对话，并逐步建立对用户的深度理解模型。框架支持 CLI、Telegram、Discord、Slack、WhatsApp、Signal、Home Assistant 等多种接入方式，可部署在 $5 VPS、GPU 集群或 Serverless 基础设施上，通过单一 Gateway 进程统一管理所有平台的对话与工具调用。当前实例使用 `anthropic/claude-opus-4.6` 作为默认模型，经 OpenRouter 路由，配置版本为 `_config_version: 14`。

## 技术栈 / 关键特性

- **Language**: Python 3.11+, 使用 `uv` 包管理器
- **LLM 接口**: OpenAI-compatible API (支持 OpenRouter / Nous Portal / z.ai / Kimi / MiniMax / Ollama 等)
- **CLI 框架**: Rich (渲染) + prompt_toolkit (输入/补全) + curses (菜单)
- **Web 工具**: firecrawl-py, parallel-web (搜索/提取)
- **TTS/STT**: Edge TTS (免费), ElevenLabs, OpenAI Whisper, Mistral Voxtral, NeuTTS
- **消息平台**: python-telegram-bot, discord.py, aiohttp (Slack/WhatsApp/Signal 适配器)
- **容器后端**: Docker, SSH, Singularity, Modal, Daytona (6 种 terminal backend)
- **浏览器自动化**: Browserbase / Browser Use / Firecrawl cloud
- **MCP 集成**: 原生 MCP 客户端, 支持 OAuth 2.1 PKCE 认证
- **IDE 集成**: ACP (Agent Client Protocol) -- VS Code / Zed / JetBrains
- **存储**: SQLite + FTS5 全文搜索 (会话), YAML (配置), 文件系统 (记忆/Skill)
- **测试**: pytest (~3000 测试用例)
- **RL 训练**: Atropos 环境集成 (可选)

## 架构与设计

### 核心模块依赖链

```
tools/registry.py          -- 工具注册中心 (无依赖)
       ^
tools/*.py                 -- 各工具注册 handler (调用 registry.register)
       ^
model_tools.py             -- 工具发现 _discover_tools() + 调度 handle_function_call()
       ^
run_agent.py / cli.py / batch_runner.py / environments/
```

### AIAgent 核心循环 (run_agent.py)

Agent 的核心是一个同步对话循环：

```python
class AIAgent:
    def __init__(self,
        model: str = "anthropic/claude-opus-4.6",
        max_iterations: int = 90,
        enabled_toolsets: list = None,
        platform: str = None,    # "cli", "telegram", etc.
        session_id: str = None,
        ...
    ): ...

    # 核心循环
    while api_call_count < self.max_iterations:
        response = client.chat.completions.create(
            model=model, messages=messages, tools=tool_schemas
        )
        if response.tool_calls:
            for tool_call in response.tool_calls:
                result = handle_function_call(tool_call.name, tool_call.args)
                messages.append(tool_result_message(result))
            api_call_count += 1
        else:
            return response.content
```

消息遵循 OpenAI 格式: `{"role": "system/user/assistant/tool", ...}`。Reasoning content 存储在 `assistant_msg["reasoning"]` 中。

### CLI 架构 (cli.py)

- **Rich** 渲染 banner/panels，**prompt_toolkit** 处理多行输入与自动补全
- **KawaiiSpinner** (`agent/display.py`) -- API 调用期间的动画表情，`|` 活动流显示工具输出
- **Skin Engine** (`hermes_cli/skin_engine.py`) -- 数据驱动的主题系统，支持内置 skin (default/ares/mono/slate) 和用户自定义 YAML skin
- **Slash Command Registry** (`hermes_cli/commands.py`) -- 统一命令注册中心，CLI/Gateway/Telegram/Slack/自动补全均从此派生

### Gateway 消息网关

```
gateway/run.py       -- 主循环, 消息分发
gateway/session.py   -- SessionStore (会话持久化)
gateway/platforms/   -- 平台适配器: telegram, discord, slack, whatsapp, homeassistant, signal
```

## 核心知识点

### 1. 记忆系统 (Memory)

记忆功能已启用，包含以下配置：

| 配置项 | 值 | 说明 |
|--------|-----|------|
| `memory.memory_enabled` | `true` | 记忆功能开关 |
| `memory.user_profile_enabled` | `true` | 用户画像 |
| `memory.memory_char_limit` | `2200` | 记忆字符上限 |
| `memory.user_char_limit` | `1375` | 用户信息字符上限 |
| `memory.nudge_interval` | `10` | 每 10 轮提醒 Agent 更新记忆 |
| `memory.flush_min_turns` | `6` | 最少 6 轮后可刷新记忆 |

支持多种记忆后端: 内置文件系统、Honcho (辩证用户建模)、Supermemory、mem0、Hindsight、RetainDB、OpenViking、ByteRover。

### 2. 上下文压缩 (Compression)

当上下文超过阈值时自动压缩，使用廉价模型生成摘要：

```yaml
compression:
  enabled: true
  threshold: 0.5          # 上下文使用率达 50% 时触发
  target_ratio: 0.2       # 压缩到 20%
  protect_last_n: 20      # 保护最近 20 条消息不被压缩
  summary_model: google/gemini-3-flash-preview
```

### 3. 终端后端 (Terminal Backends)

支持 6 种终端后端，可通过 `terminal.backend` 切换：

| 后端 | 说明 |
|------|------|
| `local` | 本地终端 (当前使用) |
| `docker` | Docker 容器隔离 |
| `ssh` | 远程 SSH |
| `modal` | Serverless (按需唤醒) |
| `daytona` | Serverless 持久化 |
| `singularity` | HPC 容器 |

容器默认镜像: `nikolaik/python-nodejs:python3.11-nodejs20`，资源配置: 1 CPU / 5GB RAM / 50GB Disk。

### 4. Skill 系统

当前安装了 26 个 Skill 模块，覆盖以下领域：

| 分类 | Skill 目录 |
|------|-----------|
| 开发 | `software-development`, `github`, `devops`, `mcp` |
| 研究 | `research`, `data-science`, `mlops`, `autonomous-ai-agents` |
| 生产力 | `productivity`, `note-taking`, `email`, `feeds` |
| 创意 | `creative`, `diagramming`, `media`, `gifs`, `html-ppt` |
| 平台 | `apple`, `smart-home`, `social-media`, `gaming` |
| 安全 | `red-teaming`, `domain` |
| 其他 | `inference-sh`, `leisure`, `dogfood` |

Skill 以文件形式存储在 `~/.hermes/skills/<name>/`，每个 Skill 包含 `SKILL.md` 定义和可选的脚本文件。Agent 在对话中将 Skill 内容作为 user message 注入（而非 system prompt），以保持 prompt caching 有效性。

### 5. 个性化 (Personalities)

预设 14 种人格模式，当前激活 `kawaii`。可通过 `/personality <name>` 切换，也可在 `SOUL.md` 中自定义人格描述（文件加载后即时生效，无需重启）。

### 6. 安全机制

- **命令审批**: `approvals.mode: manual` -- 危险命令需人工确认
- **密钥脱敏**: `security.redact_secrets: true`
- **Tirith 安全策略**: 已启用 (fail-open 模式)
- **MCP 安全**: OAuth 2.1 PKCE 认证 + OSV 恶意软件扫描
- **容器隔离**: Docker/Singularity 提供沙箱执行环境

### 7. 定时任务 (Cron)

内置 cron 调度器，支持自然语言描述的定时任务，可将结果投递到任意消息平台。支持 pre-run 脚本注入用于数据收集和变更检测。

## 关键配置片段

### 当前 config.yaml 核心配置

```yaml
model:
  default: anthropic/claude-opus-4.6
  provider: auto
  base_url: https://openrouter.ai/api/v1

agent:
  max_turns: 90
  gateway_timeout: 1800
  reasoning_effort: medium
  tool_use_enforcement: auto

terminal:
  backend: local
  persistent_shell: true
  docker_image: nikolaik/python-nodejs:python3.11-nodejs20

compression:
  enabled: true
  threshold: 0.5
  target_ratio: 0.2
  summary_model: google/gemini-3-flash-preview

memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200
  nudge_interval: 10

display:
  personality: kawaii
  streaming: true
  inline_diffs: true

security:
  redact_secrets: true
  tirith_enabled: true

platform_toolsets:
  cli: [hermes-cli]
  telegram: [hermes-telegram]
  discord: [hermes-discord]
  whatsapp: [hermes-whatsapp]
  slack: [hermes-slack]
  signal: [hermes-signal]
  homeassistant: [hermes-homeassistant]
```

### Fallback Model 配置 (未启用)

```yaml
# fallback_model:
#   provider: openrouter
#   model: anthropic/claude-sonnet-4

# smart_model_routing:
#   enabled: true
#   max_simple_chars: 160
#   cheap_model:
#     provider: openrouter
#     model: google/gemini-2.5-flash
```

## 使用方法 / 构建步骤

### 安装

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
source ~/.bashrc
hermes              # 启动对话
```

### 常用命令

```bash
hermes              # 交互式 CLI
hermes model        # 选择 LLM provider 和模型
hermes tools        # 配置启用的工具
hermes config set   # 设置单个配置值
hermes gateway      # 启动消息网关 (Telegram/Discord 等)
hermes setup        # 完整设置向导
hermes update       # 更新到最新版
hermes doctor       # 诊断问题
hermes skills       # 管理技能
```

### 开发环境搭建

```bash
git clone --recurse-submodules https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
uv venv venv --python 3.11
source venv/bin/activate
uv pip install -e ".[all,dev]"
python -m pytest tests/ -q    # 运行 ~3000 测试
```

### Slash 命令速查

| 命令 | 说明 |
|------|------|
| `/new`, `/reset` | 新建/重置会话 |
| `/model [provider:model]` | 切换模型 |
| `/personality [name]` | 切换人格 |
| `/retry`, `/undo` | 重试/撤销 |
| `/compress`, `/usage` | 压缩上下文/查看用量 |
| `/skills` | 浏览技能 |
| `/stop` | 中断当前任务 |

## 目录结构

```
~/.hermes/
├── config.yaml              # 主配置文件 (_config_version: 14)
├── .env                     # API Keys (权限 600)
├── SOUL.md                  # Agent 人格定义 (当前为空，使用 config personality)
├── hermes-agent/            # Agent 源码 (git clone)
├── skills/                  # 26 个技能模块
├── memories/                # 持久化记忆
├── sessions/                # 会话历史 (SQLite + FTS5)
├── hooks/                   # 生命周期钩子
├── cron/                    # 定时任务
├── logs/                    # 运行日志 (agent.log + errors.log)
├── audio_cache/             # 音频缓存
├── image_cache/             # 图片缓存
├── pairing/                 # 配对设备
└── whatsapp/                # WhatsApp 集成
```

## 备注

- `streaming.enabled: false` 与 `display.streaming: true` 存在不一致，前者控制 API streaming，后者控制 CLI 显示
- `.env` 文件权限 `600`，存储 OpenRouter / ZAI / Kimi 等 API Keys
- `SOUL.md` 为空文件，Agent 使用 config.yaml 中的 `personality` 设置 (当前 `kawaii`)
- 当前版本为 v0.8.0 (v2026.4.8)，209 个 merged PR，82 个 resolved issues
- Smart Model Routing 可开启以将简单请求路由到廉价模型 (如 Gemini 2.5 Flash)

## 相关笔记

- [[hermes-agent-config]] -- Hermes Agent 详细配置参考
- [[agent-skills-kb]] -- Agent Skills 知识库 (iOS 相机诊断/计算机视觉)
- [[AI-EDA]] -- AI-EDA 智能 EDA 工具
- [[myskills]] -- CamSkills 工业相机固件开发技能包
