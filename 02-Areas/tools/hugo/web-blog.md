---
tags: [hugo, blog, azure, nginx]
category: tools/hugo
created: 2026-06-09
---
I now have all the information needed. Here is the comprehensive Obsidian markdown note:

```markdown
---
title: "Hugo 个人技术博客 (web-blog)"
tags:
  - hugo
  - blog
  - static-site
  - webhook
  - nginx
  - azure
  - automation
  - devops
  - fixit-theme
category: tools/hugo
created: 2026-06-09
status: production
project_name: web-blog
project_path: /home/lijian/project/web/boke
---

# Hugo 个人技术博客 (web-blog)

## 项目概述

基于 Hugo 静态站点生成器 + FixIt 主题构建的个人技术博客，专注于嵌入式硬件和软件技术分享。通过阿里云 Codeup 托管代码，部署在 Azure 服务器上，利用 Python Flask Webhook 实现 Git Push 后自动构建部署的完整 CI/CD 流程。

## 技术栈

| 组件 | 技术选型 | 说明 |
|------|---------|------|
| 静态生成器 | Hugo 0.160.1 (Extended) | 秒级构建，单二进制文件 |
| 博客主题 | FixIt | 功能丰富的 Hugo 主题，支持中英文 |
| 代码托管 | 阿里云 Codeup | 国内访问快，支持 Webhook |
| 服务器 | Microsoft Azure | Ubuntu 系统，IP: 20.64.150.63 |
| Web 服务 | Nginx | 静态文件服务 + 反向代理 |
| 自动化部署 | Python Flask | Webhook 接收端，触发构建 |
| 服务管理 | systemd | Webhook 服务守护进程 |
| 开发环境 | WSL2 | 本地开发与测试 |

## 架构设计

### 系统架构

```
开发环境 (WSL2)                    生产环境 (Azure)
┌─────────────┐                   ┌──────────────────────────────┐
│  Hugo 写作  │                   │  Nginx (:80)                 │
│      ↓      │                   │    ├── /     → 静态文件      │
│  Git Commit │                   │    ├── /webhook → Flask(:8080)│
│      ↓      │                   │    └── /health → Flask(:8080) │
│  Push Codeup│ ── Webhook ──→   │                              │
│             │                   │  Flask Webhook 服务           │
│             │                   │    git pull → hugo build      │
│             │                   │    → chown → 返回状态         │
└─────────────┘                   └──────────────────────────────┘
```

### 部署流水线

```
本地写作 → hugo new posts/xxx.md → git add/commit → git push origin main
    → Codeup 触发 Webhook → Azure Flask 接收 → git pull → hugo -d /var/www/html
    → Nginx 自动服务新内容
```

### 端口规划

| 端口 | 服务 | 用途 | 对外开放 |
|------|------|------|----------|
| 22 | SSH | 服务器管理 | 是 |
| 80 | Nginx | HTTP 博客访问 | 是 |
| 8080 | Flask | Webhook 内部服务 | 否 (仅 127.0.0.1) |

### 关键目录结构

```
项目本地: /home/lijian/project/web/boke
├── hugo.toml              # Hugo 主配置文件
├── content/posts/         # 文章目录
├── themes/FixIt/          # FixIt 主题
├── archetypes/default.md  # 文章模板
├── static/                # 静态资源
├── public/                # 本地构建输出
└── layouts/               # 自定义布局覆盖

服务器: /var/www/
├── blog/                  # 博客源代码 (git clone)
│   ├── webhook_server.py  # Webhook 服务脚本
│   └── deploy.log         # 部署日志
└── html/                  # Hugo 构建输出 (Nginx root)
```

## 关键配置

### Hugo 核心配置 (`hugo.toml`)

```toml
title = ""
baseURL = "http://localhost:1313"
theme = ["FixIt"]
defaultContentLanguage = "zh-cn"
languageCode = "zh-CN"
enableRobotsTXT = true
enableGitInfo = true
enableEmoji = true

[markup.highlight]
codeFences = true
noClasses = false
lineNos = true

[markup.goldmark.renderer]
unsafe = true
```

### Nginx 配置

```nginx
server {
    listen 80;
    server_name _;

    location /webhook {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /health {
        proxy_pass http://127.0.0.1:8080;
    }

    location / {
        root /var/www/html;
        index index.html;
        try_files $uri $uri/ =404;
        gzip on;
        gzip_vary on;
        gzip_min_length 1024;
        gzip_types text/plain text/css text/xml text/javascript
                   application/json application/javascript;
    }

    location ~ /\. {
        deny all;
    }
}
```

### Webhook 服务核心逻辑

```python
@app.route('/webhook', methods=['POST'])
def webhook():
    # 1. 拉取最新代码
    success, output = run_command('git pull', cwd=BLOG_PATH)
    # 2. 重新克隆主题
    run_command('rm -rf themes/FixIt', cwd=BLOG_PATH)
    run_command('git clone --depth 1 https://github.com/hugo-fixit/FixIt.git themes/FixIt', cwd=BLOG_PATH)
    run_command('rm -rf themes/FixIt/.git', cwd=BLOG_PATH)
    # 3. 构建 Hugo
    success, output = run_command(f'hugo -d {OUTPUT_PATH}', cwd=BLOG_PATH)
    # 4. 设置权限
    run_command(f'chown -R www-data:www-data {OUTPUT_PATH}')
    return jsonify({'status': 'success'}), 200
```

### Systemd 服务配置

```ini
[Unit]
Description=Blog Auto-Deploy Webhook Service
After=network.target

[Service]
Type=simple
User=lijian
Group=lijian
WorkingDirectory=/var/www/blog
ExecStart=/usr/bin/python3 /var/www/blog/webhook_server.py
Restart=always
RestartSec=10
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

## 构建与运行

### 本地开发

```bash
# 创建新文章
hugo new posts/my-new-post.md

# 本地预览 (含草稿)
hugo server -D --bind 0.0.0.0
# 访问 http://localhost:1313

# 构建生产版本
hugo -d /var/www/html
```

### 发布流程

```bash
git add .
git commit -m "Add new post"
git push origin main
# 等待 1-2 分钟，博客自动更新
```

### 服务器管理

```bash
# 查看 Webhook 服务状态
sudo systemctl status blog-webhook

# 重启 Webhook 服务
sudo systemctl restart blog-webhook

# 查看部署日志
tail -f /var/www/blog/deploy.log

# 健康检查
curl http://20.64.150.63/health

# 手动部署 (备用方案)
cd /var/www/blog && git pull && rm -rf themes/FixIt && \
git clone --depth 1 https://github.com/hugo-fixit/FixIt.git themes/FixIt && \
rm -rf themes/FixIt/.git && hugo -d /var/www/html
```

## 关键设计决策

1. **Hugo 而非 Hexo/Jekyll**: Hugo 是 Go 编写的单二进制文件，构建速度极快（秒级），无运行时依赖，中文支持良好
2. **FixIt 主题**: 功能全面的 Hugo 主题，内置代码高亮、数学公式、Mermaid 图表、暗色模式、响应式设计等，且持续维护
3. **Codeup + Azure 混合架构**: 利用国内 Codeup 保证推送速度，Azure 保证全球访问可用性
4. **Flask Webhook 而非 GitHub Actions**: 服务器直接接收 Webhook 并执行构建，减少外部依赖，部署链路更短
5. **systemd 守护进程**: 确保 Webhook 服务持久运行，自动重启，日志可追踪
6. **每次重新克隆主题**: Webhook 部署时删除并重新克隆 FixIt 主题，避免嵌套 Git 仓库问题，保证主题始终最新

## 踩坑记录与经验

### Git 嵌套仓库问题

克隆主题后 `themes/FixIt/` 内含 `.git` 目录，导致主仓库无法正常跟踪。解决方案：每次克隆主题后必须 `rm -rf themes/FixIt/.git`。

### Hugo 配置文件命名变更

Hugo 新版本使用 `hugo.toml` 而非旧版的 `config.toml`。安装 FixIt 主题后从 `exampleSite/config.toml` 复制的配置需要重命名。

### PEP 668 限制

Ubuntu 新版 Python 遵循 PEP 668，禁止 `pip install` 直接安装系统包。解决方案：使用 `apt install python3-flask` 代替 pip。

### Git "dubious ownership" 错误

systemd 服务最初以 `www-data` 用户运行，但仓库属主为 `lijian`，导致 Git 拒绝操作。解决方案：将服务用户改为 `lijian`。

### Git 分支名冲突

本地默认分支为 `master`，远程仓库为 `main`，导致 push/pull 失败。解决方案：`git branch -M main` + `git branch --set-upstream-to=origin/main`。

### resources/_gen 冲突

Hugo 本地构建生成的 `resources/_gen/` 目录在服务器 pull 时与已有文件冲突。解决方案：服务器端 `rm -rf resources/_gen/` 后重新 pull。

## 功能特性清单

- [x] 静态站点生成 (Hugo)
- [x] Markdown 写作与代码高亮
- [x] 响应式设计 (FixIt 主题)
- [x] 中文内容支持 (CJK)
- [x] 自动部署 (Webhook)
- [x] 健康检查端点 (`/health`)
- [x] 部署日志记录
- [x] Gzip 压缩
- [x] SEO 友好 (sitemap, robots.txt)
- [ ] HTTPS / SSL 证书
- [ ] 评论系统 (Waline/Giscus)
- [ ] 站内搜索
- [ ] CDN 加速
- [ ] 自动备份

## 相关资源

- Hugo 官方文档: https://gohugo.io/
- FixIt 主题文档: https://fixit.lruihao.cn/
- FixIt GitHub: https://github.com/hugo-fixit/FixIt
- 阿里云 Codeup: https://codeup.aliyun.com/
- Nginx 文档: https://nginx.org/en/docs/
- Let's Encrypt: https://letsencrypt.org/

## 项目元信息

| 属性 | 值 |
|------|-----|
| 项目名称 | 个人技术博客 |
| 项目路径 | `/home/lijian/project/web/boke` |
| 生产地址 | http://20.64.150.63 |
| 代码仓库 | https://codeup.aliyun.com/687efa92a72fbe627240276f/codeup/boke.git |
| 项目完成时间 | 2026-04-24 |
| 开发周期 | 1 天 |
| 状态 | 生产环境运行中 |
```

## 相关笔记

- [[embedded-blog]] — 嵌入式技术博客（同为 Hugo 博客）
- [[vuepress]] — VuePress 知识库（同为静态站点）
- [[docker-alicloud]] — Docker 镜像推送到阿里云 ACR
- [[lijianResume]] — 李健简历（个人博客链接）
