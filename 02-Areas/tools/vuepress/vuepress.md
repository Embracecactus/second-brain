---
tags: [vuepress, github-pages]
category: tools/vuepress
created: 2026-06-09
---
```markdown
---
title: VuePress 知识库
tags:
  - vuepress
  - static-site-generator
  - documentation
  - vue
  - vite
  - github-pages
category: tools/vuepress
created: 2026-06-09
status: active
---

# VuePress 知识库

## 项目概述

本项目是基于 VuePress 2.x + vuepress-theme-plume 主题构建的文档站点，用于云物通科技有限公司的产品文档展示，包含控制器硬件说明、IoT 设备集成文档等内容，通过 GitHub Pages 自动部署。

## 技术栈

| 分类 | 技术 | 版本 |
|------|------|------|
| 核心框架 | VuePress | 2.0.0-rc.24 |
| 主题 | vuepress-theme-plume | 1.0.0-rc.158 |
| 备用主题 | vuepress-theme-hope | 2.0.0-rc.94 |
| 打包器 | @vuepress/bundler-vite | 2.0.0-rc.24 |
| 前端框架 | Vue | 3.5.17 |
| 样式预处理 | sass-embedded | 1.89.2 |
| 类型系统 | TypeScript | 5.8.3 |
| 包管理器 | pnpm | 10.13.1 |
| Node.js | Node.js (via nvm) | 22 (CI) / 18.19.0 (本地) |
| 部署 | GitHub Pages (gh-pages 分支) | - |

## 架构与关键设计

### 项目目录结构

```
vuepress/
├── README.md                          # 安装配置指南 (Linux/Windows)
├── Embracecactus.github.io/           # 主站点项目 (GitHub Pages 部署)
│   ├── src/
│   │   ├── .vuepress/
│   │   │   ├── config.ts              # VuePress 核心配置
│   │   │   ├── theme.ts               # 主题配置 (plumeTheme)
│   │   │   ├── notes.ts               # 笔记目录配置
│   │   │   ├── navbar/                # 导航栏配置 (zh/en)
│   │   │   └── sidebar/               # 侧边栏配置
│   │   ├── coreDevice/                # 联犀控制器文档
│   │   ├── hardware/                  # 硬件产品文档
│   │   ├── soft/                      # 开源软件文档 (ithings, linuxcnc)
│   │   ├── device/                    # 设备集成文档
│   │   ├── blog/                      # 博客文章
│   │   ├── doc/                       # 用户/开发手册
│   │   ├── about/                     # 关于页面
│   │   └── contact/                   # 联系页面
│   ├── .github/workflows/deploy-docs.yml  # CI/CD 部署配置
│   ├── check-git-update.sh            # 自动检测更新脚本
│   └── package.json
├── my-docs/                           # 开发环境测试项目
└── my-docs-1/                         # 另一个测试项目
```

### 核心配置架构

**VuePress 配置** (`config.ts`)：
- 使用 `defineUserConfig` 定义站点级别配置
- 语言设为 `zh-CN`，站点标题"云物通科技有限公司"
- 集成 Google Analytics (`G-QKNZQ3Z8J4`)
- 启用 `viteBundler()` 作为构建器
- 关闭 `shouldPrefetch` 优化大站点性能

**主题配置** (`theme.ts`)：
- 从 `vuepress-theme-hope` 迁移到 `vuepress-theme-plume`
- 博客功能禁用 (`blog: false`)
- 启用 filesystem 缓存加速编译
- 自动 frontmatter 生成（permalink、createTime、title）
- 启用 bulletin 公告板功能
- 丰富的 markdownPower 插件：abbr、annotation、PDF 嵌入、caniuse、bilibili/youtube 视频嵌入、代码演示容器 (go/rust/kotlin)、图片尺寸自动填充
- 侧边栏自动模式 (`sidebar: 'auto'`)

**笔记配置** (`notes.ts`)：
- 使用 `defineNotesConfig` 定义笔记目录
- `coreDevice` 目录映射到 `/coreDevice` 路由，sidebar 自动生成

### 部署流程

**GitHub Actions CI/CD** (`deploy-docs.yml`)：
1. 触发条件：push 到 `main` 分支
2. 环境：ubuntu-latest, Node.js 22, pnpm 8
3. 构建步骤：`pnpm install --frozen-lockfile` → `pnpm run docs:build`
4. 输出目录：`src/.vuepress/dist`，部署到 `gh-pages` 分支
5. 内存配置：`NODE_OPTIONS: --max_old_space_size=8192`

**自动更新脚本** (`check-git-update.sh`)：
- 比较本地与远程 `main` 分支 commit hash
- 检测到更新时自动 `git pull` + `pnpm docs:build`
- 适用于服务器定时任务 (cron)

## 核心知识点

### 1. VuePress 2.x 项目初始化

```bash
# 使用 pnpm 交互式创建
pnpm create vuepress-theme-hope my-docs

# 创建过程中的选项：
# - 语言: 简体中文
# - 包管理器: pnpm
# - 打包器: vite
# - 项目类型: blog
# - 多语言: Yes
```

### 2. 主题从 hope 迁移到 plume

项目经历了从 `vuepress-theme-hope` 到 `vuepress-theme-plume` 的迁移：
- `theme.ts` 中导入从 `hopeTheme` 改为 `plumeTheme`
- `navbar` 配置从 `navbar()` 改为 `defineNavbarConfig()`
- 新增 `notes` 配置系统 (`defineNotesConfig`)
- 新增 `markdownPower` 插件体系替代原有 markdown 配置

### 3. pnpm 常用命令

```bash
pnpm install                    # 安装依赖
pnpm docs:dev                   # 启动开发服务器 (localhost:8080)
pnpm docs:build                 # 构建生产版本
pnpm docs:clean-dev             # 清除缓存并启动开发服务器
pnpm dlx vp-update              # 更新 VuePress 相关包
pnpm add -D typescript          # 添加 TypeScript
pnpm add -D tm-themes tm-grammars  # 添加 Shiki 语法高亮主题/语法
```

### 4. 环境准备 (Linux)

```bash
# 安装 nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc

# 安装 Node.js
nvm install --lts
nvm alias default 18.19.0

# 启用 corepack (pnpm 支持)
corepack enable
```

### 5. 导航栏配置模式

使用 `defineNavbarConfig` 定义导航结构，支持：
- 文字链接 (`text` + `link`)
- 图标 (`icon`，使用 Iconify 格式如 `ic:round-home`)
- 下拉菜单 (`items` 嵌套)

## 重要代码片段

### VuePress 核心配置模板

```typescript
// config.ts
import { defineUserConfig } from "vuepress";
import { viteBundler } from '@vuepress/bundler-vite'
import theme from "./theme.js";

export default defineUserConfig({
  base: "/",
  lang: "zh-CN",
  locales: {
    "/": {
      lang: "zh-CN",
      title: "站点标题",
      description: "站点描述",
    },
  },
  head: [
    ['link', { rel: 'icon', type: 'image/png', href: '/logo/logo2.png' }],
  ],
  bundler: viteBundler(),
  shouldPrefetch: false,
  theme,
});
```

### plumeTheme 配置模板

```typescript
// theme.ts
import { plumeTheme } from "vuepress-theme-plume";
import { zhNavbar } from "./navbar/index.js";
import { zhSidebar } from "./sidebar/index.js";
import { zhNotes } from './notes'

export default plumeTheme({
  hostname: "https://your-domain.com",
  contributors: true,
  changelog: true,
  blog: false,
  cache: 'filesystem',
  autoFrontmatter: {
    permalink: true,
    createTime: true,
    title: true,
  },
  plugins: {
    markdownPower: {
      abbr: true,
      annotation: true,
      pdf: true,
      caniuse: true,
      bilibili: true,
      youtube: true,
      imageSize: 'local',
    },
    markdownInclude: true,
  },
  locales: {
    '/': {
      navbar: zhNavbar,
      notes: zhNotes,
      sidebar: 'auto',
      outline: "deep",
    },
  },
});
```

### GitHub Actions 部署配置

```yaml
# .github/workflows/deploy-docs.yml
name: 部署文档
on:
  push:
    branches: [main]
permissions:
  contents: write
jobs:
  deploy-gh-pages:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 8 }
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: corepack enable && pnpm install --frozen-lockfile
      - env:
          NODE_OPTIONS: --max_old_space_size=8192
        run: |
          pnpm run docs:build
          > src/.vuepress/dist/.nojekyll
      - uses: JamesIves/github-pages-deploy-action@v4
        with:
          branch: gh-pages
          folder: src/.vuepress/dist
```

## 构建/运行方法

```bash
# 进入项目目录
cd /mnt/c/Users/lijian/workspace/vuepress/Embracecactus.github.io

# 安装依赖
pnpm install

# 本地开发 (http://localhost:8080)
pnpm docs:dev

# 清除缓存后开发
pnpm docs:clean-dev

# 生产构建
pnpm docs:build

# 更新 VuePress 生态包
pnpm dlx vp-update
```

## 相关笔记链接

- [[Vue 3]]
- [[Vite]]
- [[TypeScript]]
- [[pnpm]]
- [[GitHub Actions]]
- [[GitHub Pages 部署]]
- [[Markdown 语法]]
- [[IoT 物联网平台]]
- [[联犀控制器]]
- [[静态站点生成器对比]]
```
## 相关笔记

- [[web-blog]] — Hugo 个人技术博客（同为静态站点部署）
- [[embedded-blog]] — 嵌入式技术博客
- [[redbook]] — 小红书技术内容创作素材库
- [[feishuxiaoapp]] — 飞书小程序项目
