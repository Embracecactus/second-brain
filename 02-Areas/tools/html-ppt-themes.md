---
tags: [ppt, presentation, themes]
category: tools
created: 2026-06-09
---

# html-ppt — HTML PPT Studio

## 项目概述

html-ppt 是一款专业的 **AgentSkill**，让 AI 生成真正可交付的 HTML 演示文稿。纯静态 HTML/CSS/JS，零构建步骤，仅 CDN 加载 webfonts。

**核心数据：**
- 36 套主题 (Themes)
- 15 套完整 Deck 模板 (Full-deck templates)
- 31 种单页布局 (Single-page layouts)
- 47 个动效 (27 CSS Animations + 20 Canvas FX)
- 内置演讲者模式 (Presenter Mode) — 像素级完美预览 + 逐字稿提词器 + 计时器

**安装：**
```bash
npx skills add https://github.com/lewislulu/html-ppt-skill
```

**作者：** lewis <sudolewis@gmail.com>
**协议：** MIT
**GitHub：** https://github.com/lewislulu/html-ppt-skill

---

## 关键知识点

### 1. Token 驱动的设计系统

所有颜色、圆角、阴影、字体决策都集中在 `assets/base.css` 中定义的 CSS Custom Properties (变量)，每个主题文件只覆盖这些 token。修改一个变量，整份 deck 优雅重排。

核心 token 包括：`--bg`, `--bg-soft`, `--surface`, `--surface-2`, `--border`, `--text-1/2/3`, `--accent`, `--accent-2/3`, `--good`, `--warn`, `--bad`, `--grad`, `--grad-soft`, `--radius*`, `--shadow*`, `--font-sans`, `--font-display`

**铁律：** 使用 `var(--text-1)` 等 token，绝不写 `color: #111` 这样的字面量。

### 2. Iframe 隔离预览

主题 / 布局 / 完整 deck 的 showcase 都用 `<iframe>` 每页独立渲染，确保预览是真实、独立的渲染结果，不会样式污染。

### 3. 幻灯片系统结构

每个逻辑页面是一个 `<section class="slide">` 元素。`runtime.js` 通过 `.is-active` class 控制可见性，其余 slide 隐藏。

```html
<div class="deck">
  <section class="slide" data-title="Cover">
    <!-- 内容 -->
    <aside class="notes">逐字稿</aside>
  </section>
</div>
```

### 4. 演讲者模式 (Presenter Mode)

按 `S` 键弹出独立演讲者窗口，包含 4 个可拖拽、可调整大小的**磁吸卡片 (Magnetic Cards)**：

| 卡片 | 颜色 | 功能 |
|---|---|---|
| CURRENT | 蓝色 | 当前页像素级完美预览 (iframe) |
| NEXT | 紫色 | 下一页预览 (iframe) |
| SPEAKER SCRIPT | 橙色 | 逐字稿大字号显示 |
| TIMER | 绿色 | 计时器 + 页码 + 前后翻页按钮 |

**像素级完美原理：** 每个预览是一个 `<iframe>`，加载同一份 deck HTML 文件，URL 多了 `?preview=N` 参数。`runtime.js` 检测后只渲染第 N 页并隐藏所有 chrome。两个窗口通过 `BroadcastChannel` 双向同步翻页，iframe 内通过 `postMessage({type:'preview-goto', idx:N})` 切换 `.is-active`，不重新加载、不闪烁。

卡片位置和尺寸自动保存到 `localStorage`，支持"重置布局"按钮。

### 5. 逐字稿写作三铁律

1. **不是讲稿，是"提示信号"** — 关键词用 `<strong>` 加粗，过渡句独立成段
2. **每页 150–300 字** — 约 2–3 分钟/页的节奏
3. **用口语，不用书面语** — "所以"不是"因此"，"这个"不是"该"

逐字稿放在 `<aside class="notes">` 或 `<div class="notes">` 中，`.notes` 类默认 `display:none`，只在演讲者视图可见。

### 6. 中英双语支持

- 预导入 Noto Sans SC / Noto Serif SC
- 使用 `lang="zh-CN"` 属性
- 双语标题写法：`<h1 class="h1">主标题<br><span class="dim">English subtitle</span></h1>`

---

## 技术细节

### 主题系统 (36 Themes)

每个主题是一个纯 CSS 文件 (`assets/themes/*.css`)，覆盖 `base.css` 中定义的 token。通过 `<link id="theme-link">` 切换，或按 `T` 键循环。

**主题分类：**

| 类别 | 主题 |
|---|---|
| Light & calm | `minimal-white`, `editorial-serif`, `soft-pastel`, `xiaohongshu-white`, `solarized-light`, `catppuccin-latte` |
| Bold & statement | `sharp-mono`, `neo-brutalism`, `bauhaus`, `swiss-grid`, `memphis-pop` |
| Cool & dark | `catppuccin-mocha`, `dracula`, `tokyo-night`, `nord`, `gruvbox-dark`, `rose-pine`, `arctic-cool` |
| Warm & vibrant | `sunset-warm` |
| Effect-heavy | `glassmorphism`, `aurora`, `rainbow-gradient`, `blueprint`, `terminal-green` |
| Professional light | `corporate-clean`, `pitch-deck-vc`, `academic-paper`, `japanese-minimal`, `engineering-whiteprint` |
| Bold editorial | `magazine-bold`, `news-broadcast`, `midcentury`, `retro-tv` |
| Dramatic | `cyberpunk-neon`, `vaporwave`, `y2k-chrome` |

**T 键循环配置：**
```html
<body data-themes="minimal-white,aurora,catppuccin-mocha" data-theme-base="../assets/themes/">
```

### 布局系统 (31 Layouts)

每个布局是 `templates/single-page/<name>.html`，包含真实示例数据，可独立在浏览器打开。

**分类：**

| 类型 | 布局 |
|---|---|
| 开场/过渡 | `cover`, `toc`, `section-divider` |
| 文本中心 | `bullets`, `two-column`, `three-column`, `big-quote` |
| 数据/数字 | `stat-highlight`, `kpi-grid`, `table`, `chart-bar`, `chart-line`, `chart-pie`, `chart-radar` |
| 代码/终端 | `code`, `diff`, `terminal` |
| 图表/流程 | `flow-diagram`, `arch-diagram`, `process-steps`, `mindmap` |
| 计划/对比 | `timeline`, `roadmap`, `gantt`, `comparison`, `pros-cons`, `todo-checklist` |
| 视觉 | `image-hero`, `image-grid` |
| 结尾 | `cta`, `thanks` |

### CSS 动画 (27 Animations)

通过 `data-anim="name"` 或 `class="anim-name"` 应用。`runtime.js` 在 slide 激活时自动重新触发 `data-anim` 元素。

**关键动画：**
- **方向性淡入：** `fade-up`, `fade-down`, `fade-left`, `fade-right`
- **戏剧性入场：** `rise-in`, `drop-in`, `zoom-pop`, `blur-in`, `glitch-in`
- **文字效果：** `typewriter`, `neon-glow`, `shimmer-sweep`, `gradient-flow`
- **列表/数字：** `stagger-list` (子元素依次入场), `counter-up` (数字滚动)
- **SVG/几何：** `path-draw`, `morph-shape`
- **3D/透视：** `parallax-tilt`, `card-flip-3d`, `cube-rotate-3d`, `page-turn-3d`, `perspective-zoom`
- **环境/持续：** `marquee-scroll`, `kenburns`, `confetti-burst`, `spotlight`, `ripple-reveal`

所有动画在 `prefers-reduced-motion: reduce` 下自动禁用。

### Canvas FX 动画 (20 Effects)

通过 `<div data-fx="name">` 使用，需要引入 `fx-runtime.js`。进入 slide 时自动初始化，离开时停止并释放 canvas。

```html
<script src="../assets/animations/fx-runtime.js"></script>
<div data-fx="particle-burst" style="width:100%;height:360px;"></div>
```

**Canvas FX 列表：**

| 名称 | 效果 | 场景 |
|---|---|---|
| `particle-burst` | 粒子从中心爆发，重力+淡出，每 2.5s 重新爆发 | 数据揭示、统计页 |
| `confetti-cannon` | 从两底角射出彩色旋转矩形 | 感谢页、成功页 |
| `firework` | 从底部发射火箭，爆炸成彩色火花 | 庆祝、发布页 |
| `starfield` | 3D 透视星空向外飞行 | 科幻/深空背景 |
| `matrix-rain` | 绿色片假名+十六进制列下落 | 赛博/安全/数据主题 |
| `knowledge-graph` | 力导向图，28 标签节点，~50 边，实时物理 | 知识/RAG/图谱幻灯片 |
| `neural-net` | 4-6-6-3 前馈网络，脉冲沿边传播 | ML/模型架构 |
| `constellation` | 漂浮点，150px 内连线，按距离调透明度 | 环境英雄背景 |
| `orbit-ring` | 5 个同心环，不同速度的点，径向光晕 | 系统/行星/层次概念 |
| `galaxy-swirl` | ~800 粒子的对数螺旋，慢速旋转 | 封面页、介绍 |
| `word-cascade` | 单词从顶部下落，底部堆积 | 词汇/概念云 |
| `letter-explode` | 标题字母从随机方向飞入，~4.5s 循环 | 大标题、英雄文本 |
| `chain-react` | 8 个圆，多米诺脉冲波传播 | 管道/顺序流 |
| `magnetic-field` | 粒子沿贝塞尔/正弦曲线运动留轨迹 | 能量/流动/抽象 |
| `data-stream` | 滚动的十六进制/二进制文本行 | 数据、API、安全 |
| `gradient-blob` | 4 个漂浮模糊径向渐变 (叠加) | 柔和英雄背景 |
| `sparkle-trail` | 指针驱动的闪光发射器 | 交互揭示 |
| `shockwave` | 从中心扩展的环 | 冲击、启动、警报 |
| `typewriter-multi` | 3 行同时打字，闪烁块光标 | 终端、Agent 启动日志 |
| `counter-explosion` | 数字从 0 计数到目标，爆发粒子，4s 后重置 | KPI 揭示 |

颜色自动读取主题的 `--accent`, `--accent-2`, `--ok`, `--warn`, `--danger`。

---

## 代码/配置片段

### 基础 HTML 结构

```html
<!DOCTYPE html>
<html lang="zh-CN" data-themes="tokyo-night,dracula,corporate-clean">
<head>
  <meta charset="utf-8">
  <title>演示文稿标题</title>
  <link rel="stylesheet" href="../assets/fonts.css">
  <link rel="stylesheet" href="../assets/base.css">
  <link rel="stylesheet" id="theme-link" href="../assets/themes/tokyo-night.css">
  <link rel="stylesheet" href="../assets/animations/animations.css">
  <link rel="stylesheet" href="style.css">
</head>
<body>
<div class="deck">
  <section class="slide" data-title="Cover">
    <h1>你的标题</h1>
    <p>副标题</p>
    <aside class="notes">
      <p>讲稿段落 1（加<strong>加粗关键词</strong>）。</p>
      <p>讲稿段落 2（过渡句独立成段）。</p>
    </aside>
  </section>
</div>
<script src="../assets/runtime.js"></script>
</body>
</html>
```

### 脚手架命令

```bash
# 新建 deck
./scripts/new-deck.sh my-talk
open examples/my-talk/index.html

# Headless Chrome 导出 PNG
./scripts/render.sh templates/theme-showcase.html        # 单页
./scripts/render.sh examples/my-talk/index.html 12       # 12 页
./scripts/render.sh examples/my-talk/index.html all      # 自动检测页数
```

`render.sh` 使用 macOS Chrome 的 `--headless=new` 模式，默认 1920x1080。小红书图文需改为 1242x1660。

### Counter-up 数字滚动

```html
<span class="counter" data-to="1248">0</span>
```

### 3D / Canvas FX 使用

```html
<!-- 引入 FX runtime -->
<script src="../assets/animations/fx-runtime.js"></script>

<!-- 在 slide 中使用 -->
<section class="slide">
  <div data-fx="knowledge-graph" style="width:100%;height:360px;"></div>
</section>
```

### 自定义主题

复制现有主题，重命名，只覆盖需要的变量。每个主题控制在 ~200 行以内。

### 键盘快捷键

```
← → Space PgUp PgDn Home End   翻页
F                               全屏
S                               打开演讲者窗口 (磁吸卡片)
N                               底部 notes 抽屉
R                               重置计时器 (演讲者窗口)
O                               slide 总览网格
T                               切换主题
A                               循环演示动画
#/N (URL)                       深链到第 N 页
?preview=N (URL)                预览模式 (单页无 chrome)
```

---

## 文件结构

```
html-ppt/
├── SKILL.md                      Agent 入口
├── README.md / README.zh-CN.md   文档
├── LICENSE                       MIT
├── references/                   详细目录文档
│   ├── themes.md                 36 主题 + 使用场景
│   ├── layouts.md                31 布局
│   ├── animations.md             27 CSS + 20 FX 目录
│   ├── full-decks.md             15 完整 deck 模板
│   ├── presenter-mode.md         演讲者模式 + 逐字稿指南
│   └── authoring-guide.md        完整创作工作流
├── assets/
│   ├── base.css                  共享 tokens + 基础组件
│   ├── fonts.css                 8 个 Google Fonts 引入
│   ├── runtime.js                键盘导航 + 演讲者模式 + 总览
│   ├── themes/*.css              36 主题 token 文件
│   └── animations/
│       ├── animations.css        27 个命名 CSS 动画
│       ├── fx-runtime.js         Canvas FX 自动初始化/生命周期
│       └── fx/*.js               20 个 Canvas FX 模块 + _util.js
├── templates/
│   ├── deck.html                 最小起步模板
│   ├── theme-showcase.html       主题展示 (iframe 隔离)
│   ├── layout-showcase.html      布局展示
│   ├── animation-showcase.html   动画展示
│   ├── full-decks-index.html     完整 deck 画廊
│   ├── full-decks/<name>/        15 个 scoped 多页 deck 模板
│   └── single-page/*.html        31 个布局文件 (带示例数据)
├── scripts/
│   ├── new-deck.sh               脚手架
│   └── render.sh                 headless Chrome → PNG
└── examples/demo-deck/           完整可运行示例
```

---

## 15 套完整 Deck 模板

### 提炼款 (8 个，从真实作品提炼)

| 名称 | 视觉特征 | 适用场景 |
|---|---|---|
| `xhs-white-editorial` | 纯白底、顶部彩虹条、80-110px 大标题、渐变文字、马卡龙软卡 | 小红书图文 + 横向 deck |
| `graphify-dark-graph` | 深夜渐变、漂浮模糊球、SVG 力导向图、毛玻璃卡 | 开发工具/CLI/知识图谱发布 |
| `knowledge-arch-blueprint` | 奶油纸底、单铁锈色、48px 蓝图网格、硬边框卡 | 系统架构图、工程白皮书 |
| `hermes-cyber-terminal` | 纯黑、CRT 扫描线+暗角、窗口交通灯、命令行标题 | CLI/Agent 工具评测 |
| `obsidian-claude-gradient` | GitHub 暗底、紫蓝径向光晕、紫色 pill 标签 | 开发工作流/MCP/Agent 教程 |
| `testing-safety-alert` | 红黑警示条、红删除线否定标题、L1/L2/L3 分级卡 | 安全/风险/事后复盘 |
| `xhs-pastel-card` | 奶油底、柔和模糊球、Playfair 斜体大标题、马卡龙卡 | 生活方式/个人成长/慢生活 |
| `dir-key-nav-minimal` | 8 页 8 色、160px 大标题、极大留白 | Keynote 极简演讲 |

### 场景款 (7 个，通用脚手架)

| 名称 | 页数 | 适用场景 |
|---|---|---|
| `pitch-deck` | 10 | 融资路演、投资人会议 |
| `product-launch` | 8 | 产品发布、发布主题演讲 |
| `tech-sharing` | 8 | 技术分享、内部 tech talk |
| `weekly-report` | 7 | 周报、团队状态更新 |
| `xhs-post` | 9 | 小红书图文 (3:4 @ 810x1080) |
| `course-module` | 7 | 教学模块、在线课程 |
| `presenter-mode-reveal` | 6 | 演讲者模式专用，每页带 150-300 字逐字稿 |

---

## 相关链接

- **GitHub 仓库：** https://github.com/lewislulu/html-ppt-skill
- **安装命令：** `npx skills add https://github.com/lewislulu/html-ppt-skill`
- **本地路径：** `/home/lijian/.agents/skills/html-ppt/`
- **字体来源：** Google Fonts (Inter, Noto Sans SC, Noto Serif SC, JetBrains Mono, Playfair Display, Space Grotesk, IBM Plex Mono, Archivo Black)
- **可选 CDN 依赖：** highlight.js (代码高亮), chart.js (交互图表)
