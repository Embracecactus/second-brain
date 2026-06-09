---
tags:
  - douyin
  - crawler
  - whisper
  - f2
  - ffmpeg
  - style-analysis
  - nlp
  - cuda
  - asr
  - 口播
  - 风格分析
category: content/douyin
created: 2026-06-09
source: ~/.claude/skills/douyin-crawler/
---

# 抖音视频爬取与口播风格分析管线 (douyin-crawler Skill)

## 项目概述

`douyin-crawler` 是一个 Claude Code Skill，用于对抖音账号执行完整的 **视频批量下载 -> 音频提取 -> 语音转录 -> 风格量化分析** 管线，并可基于分析结果自动生成专属的文案写作 Skill。

核心价值：给定一个抖音博主主页 URL，自动化完成从视频采集到写作风格逆向工程的全流程。

**触发词**：抖音爬取、下载抖音视频、分析口播风格、douyin download、crawl douyin、风格分析、style analysis、爬抖音、爬取视频、视频转录

**输入要求**：
- 抖音主页 URL（必填）：格式 `https://www.douyin.com/user/MS4wLjABAAAA...`
- 账号昵称（必填）：用于命名输出目录和 Skill，如 `zhuzhuxu`
- 工作目录（可选）：默认 `{项目根目录}/data/{昵称}/`
- 已有 SRT 字幕目录（可选）
- 是否生成写作 Skill（可选）

## 关键知识点

### 管线架构 (8 个 Phase)

| Phase | 名称 | 工具 | 说明 |
|-------|------|------|------|
| 0 | 初始化 | mkdir | 创建 videos/audio/transcripts/srt_originals/analysis 目录结构 |
| 1 | 环境检查 | python3/f2/ffmpeg/nvidia-smi | 检查并安装依赖，探测 CUDA 库路径 |
| 2 | Cookie 管理 | f2 + YAML | 读取/验证/刷新抖音登录 Cookie |
| 3 | 批量下载视频 | F2 (`f2 dy -m post`) | 下载账号全部视频 MP4 + 描述文案 |
| 4 | 批量提取音频 | FFmpeg | 从 MP4 提取 16kHz 单声道 WAV |
| 5 | 批量语音转录 | faster-whisper | GPU/CPU 自动降级的 ASR 转录 |
| 6 | SRT 字幕转换 | convert_srt.py | 将现成 SRT 字幕转纯文本（可选） |
| 7 | 风格量化分析 | analyze_style.py | 统计分析词频、句式、内容类型等 |
| 8 | 生成写作 Skill | Claude + skill-generator-guide | 基于分析结果生成专属文案写作 Skill |

### 核心工具链

- **F2**：抖音视频下载工具，需登录 Cookie，支持断点续传，自动跳过已下载文件
- **FFmpeg**：音视频处理，提取 16kHz/mono PCM WAV 音频用于 ASR
- **faster-whisper**：OpenAI Whisper 的高性能实现，支持 CUDA 加速
- **Claude Code Skill 系统**：通过 `~/.claude/skills/` 目录注册自定义技能

### 分析维度

风格分析脚本 (`analyze_style.py`) 对转录文本执行以下量化分析：

1. **篇幅分析**：平均字数、中位数、最短/最长、平均句长
2. **句式特征**：疑问句数量、感叹句数量、总句数
3. **高频短语**：30+ 预定义短语的出现频率（步骤标记、开场白、结尾语、互动词、强调词）
4. **内容类型分类**：技术教程/工具分享/学习日记/项目展示/踩坑记录/日常分享
5. **高频双字词** (Bigram)：Top 30 中文双字组合
6. **技术话题频率**：嵌入式/Linux/STM32/Zephyr/Docker 等关键词出现次数
7. **开场白样本**：前 25 篇文案的前 40 字
8. **结尾样本**：前 25 篇文案的后 50 字

### 写作 Skill 生成规范

基于分析结果生成的写作 Skill 包含：

```
~/.claude/skills/{昵称}-writer/
  SKILL.md              # 主 Skill 定义（9 个必含章节）
  references/
    vocabulary.md       # 个人词表（8 个章节）
    style-examples.md   # 代表性真实文案 8-10 篇
    templates.md        # 内容类型填空模板
```

SKILL.md 9 个必含章节：人设与语气、内容类型与结构模板、开场白库、结尾库、类比技法、节奏控制、标签策略、备选标题策略、质量检查清单。

## 技术细节

### 音频提取规格

```
FFmpeg 参数: -vn -acodec pcm_s16le -ar 16000 -ac 1
```
- 格式：16-bit PCM WAV
- 采样率：16kHz（Whisper 标准输入）
- 声道：单声道（mono）
- 支持跳过已存在的文件

### faster-whisper 模型降级策略

转录脚本实现了三级自动降级：

| 优先级 | 模型 | 设备 | Compute Type | VRAM 需求 |
|--------|------|------|-------------|----------|
| 1 | large-v3 | CUDA | float16 | ~3GB |
| 2 | medium | CUDA | float16 | ~1.5GB |
| 3 | medium | CPU | int8 | 无 GPU |

转录参数：`beam_size=5`, `vad_filter=True`

**预估耗时**：
- GPU (large-v3)：10-20 秒/视频，100 个视频约 20-30 分钟
- CPU (medium)：2-5 分钟/视频，100 个视频约 3-5 小时

### CUDA 库路径探测

faster-whisper 依赖 NVIDIA CUDA 库，WSL2 环境下需手动设置：

```bash
# 探测 libcublas 路径
find ~/.local/lib -name "libcublas.so*" 2>/dev/null

# 设置环境变量（典型路径）
export LD_LIBRARY_PATH=/home/{user}/.local/lib/python3.10/site-packages/nvidia/cu13/lib/:$LD_LIBRARY_PATH
```

### Cookie 管理机制

F2 需要抖音登录 Cookie 才能下载视频。配置存储在 `~/.f2/app.yaml`：

```yaml
douyin:
  headers:
    User-Agent: "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/148.0.0.0 Safari/537.36"
    Referer: "https://www.douyin.com/"
  cookie: "{用户粘贴的Cookie}"
  proxys: null
```

Cookie 验证命令：
```bash
f2 dy -m post -u "{URL}" -o 1 -p /tmp/cookie_test --max-tasks 1
```

Cookie 刷新流程：Windows 浏览器登录 douyin.com -> F12 -> Network -> 复制 Cookie header。

### 风格分析默认追踪短语

```python
DEFAULT_PHRASES = [
    "第一步", "第二步", "第三步", "第四步", "第五步",       # 步骤标记
    "哈喽大家好", "大家好", "最近", "今天", "分享",         # 开场白
    "评论区留言", "评论区", "点赞关注", "下期见", "试试",   # 结尾/CTA
    "是不是", "说实话", "其实", "然后", "所以说",           # 互动/修辞
    "特别", "就像", "好比", "简单说", "说白了",             # 强调/类比
]
```

### 内容类型分类关键词

```python
DEFAULT_CONTENT_TYPES = {
    "技术教程": ["教程", "步骤", "安装", "配置", "编译", "命令", "代码", "终端"],
    "工具分享": ["分享", "推荐", "工具", "插件", "好用", "效率", "必备"],
    "学习日记": ["学习", "今天学", "学了", "入门", "新手", "开始学"],
    "项目展示": ["项目", "做了", "制作", "开发板", "成品", "功能"],
    "踩坑记录": ["踩坑", "报错", "问题", "解决", "折腾", "失败", "搞了"],
    "日常分享": ["出差", "加班", "上班", "工作", "生活", "日常", "吃"],
}
```

## 代码/配置片段

### F2 视频下载命令

```bash
f2 dy -m post \
  -u "{用户URL}" \
  -i all \
  -p "{工作目录}/videos" \
  --max-tasks 5
```

- `-m post`：下载模式为用户帖子
- `-i all`：下载全部视频
- `--max-tasks 5`：最大并发数 5

输出目录结构：`{工作目录}/videos/douyin/post/{昵称}/`，每个视频生成 `*.mp4` + `*_desc.txt`。

### FFmpeg 音频提取关键命令

```bash
ffmpeg -i "$f" -vn -acodec pcm_s16le -ar 16000 -ac 1 "$OUT" -y 2>/dev/null
```

### faster-whisper 转录核心代码

```python
segments, info = model.transcribe(
    str(audio_file),
    language="zh",
    beam_size=5,
    vad_filter=True,
)
transcript = "".join(seg.text for seg in segments)
```

### SRT 转文本解析逻辑

```python
def srt_to_text(srt_path: str) -> str:
    # 跳过纯数字行（序号）
    # 跳过时间戳行（HH:MM:SS,mmm --> HH:MM:SS,mmm）
    # 移除 HTML 标签
    # 拼接所有文本行
```

### 中文字符计数

```python
def count_chinese_chars(text: str) -> int:
    return len(re.findall(r"[一-鿿]", text))
```

### Bigram 分析

```python
chinese_only = re.sub(r"[^一-鿿]", "", all_text)
bigrams = [chinese_only[i:i + 2] for i in range(len(chinese_only) - 1)]
bigram_freq = Counter(bigrams).most_common(50)
```

## 输出目录结构

```
{工作目录}/
  videos/              # F2 下载的原始视频
  audio/               # 提取的 WAV 音频 (16kHz mono)
  transcripts/         # ASR 转录文本 (.txt)
  srt_originals/       # SRT 字幕转文本（可选）
  analysis/
    style_analysis.json         # 完整分析数据（JSON）
    style_analysis_report.md    # 可读分析报告（Markdown）
```

## 故障排查

| 错误 | 原因 | 修复 |
|------|------|------|
| `f2: command not found` | F2 未安装 | `pip3 install -U f2` |
| F2 返回 HTTP 403 | Cookie 过期 | Phase 2: 刷新 Cookie |
| F2 下载 0 个视频 | Cookie 无登录态 / URL 错误 | 确认 Cookie 含 `sessionid`；检查 URL 格式 |
| `libcublas.so.* not found` | CUDA 库路径未设置 | `export LD_LIBRARY_PATH=...` |
| CUDA out of memory | large-v3 需要 ~3GB VRAM | 脚本自动降级到 medium |
| `nvidia-smi` 失败 | WSL2 无 GPU 驱动 | 脚本自动降级到 CPU |
| 转录结果为空 | 音频损坏或静音 | `ffprobe {文件}.wav` 检查 |
| `httpx.ReadTimeout` | 网络慢/文件大 | 重跑即可，F2 自动跳过已下载 |
| 音频提取不完整 | pipe 子 shell 变量丢失 | 使用脚本文件而非内联命令 |

## Reference 文件清单

| 文件路径 | 用途 |
|---------|------|
| `~/.claude/skills/douyin-crawler/SKILL.md` | 主 Skill 定义，完整管线流程 |
| `references/extract_audio.sh` | FFmpeg 批量音频提取脚本模板 |
| `references/transcribe.py` | faster-whisper 批量转录脚本模板 |
| `references/convert_srt.py` | SRT 字幕转纯文本脚本模板 |
| `references/analyze_style.py` | 风格量化分析脚本模板 |
| `references/skill-generator-guide.md` | 写作 Skill 生成规范指南 |

所有 Reference 文件均使用占位符（`{xxx}`）机制，由 Skill 在运行时替换为实际路径后生成到工作目录执行。

## 相关链接

- Skill 文件位置：`/home/lijian/.claude/skills/douyin-crawler/`
- 相关知识库：[[douyin-creator-skills]]（抖音创作者 Skills 汇总）
- F2 工具：`pip3 install -U f2`
- faster-whisper：`pip3 install faster-whisper`
- Whisper 模型：`large-v3`（首选）、`medium`（降级）
