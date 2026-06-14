---
title: Sub2API 部署指南
tags: [AI, Claude, 中转, 部署, 服务器]
created: 2026-06-14
---

# Sub2API 部署指南

## 项目简介

**Sub2API** 是一个开源 AI API 网关平台，用于将 Claude、OpenAI、Gemini、Antigravity 等订阅统一接入，支持账号共享、拼车分摊成本。

- **GitHub**: https://github.com/Wei-Shaw/sub2api
- **作者**: Wei-Shaw（与 CRS 同一作者）
- **技术栈**: Go + Vue 3 + PostgreSQL + Redis
- **优势**: 功能最全面、Header 指纹归一化防封更好

## 与其他方案对比

| 方案 | 语言 | 数据库 | 内存占用 | 功能 | 防封能力 |
|------|------|--------|---------|------|---------|
| **Sub2API** | Go | PostgreSQL + Redis | ~150-200MB | 最全面 | ⭐⭐⭐⭐⭐ |
| CRS | Node.js | Redis | ~500-700MB | 全面 | ⭐⭐⭐ |
| CC-Relay | Go | 无 | ~30-50MB | 轻量 | ⭐⭐ |

## 硬件要求

| 配置项 | 最低要求 | 推荐 |
|--------|---------|------|
| CPU | 1 核 | 2 核 |
| 内存 | 1GB | 2GB |
| 硬盘 | 20GB | 30GB |
| 系统 | Ubuntu 20.04+ / Debian | Ubuntu 22.04+ |
| 网络 | 需能访问 Anthropic API | 海外服务器或有代理 |

> [!note] 本文部署环境
> 1 核 1GB 内存服务器，已占用约 300MB，部署方式为非 Docker（直接安装）。

## 部署步骤

### 1. 安装 Docker（可选）

如果选择 Docker 部署方式：

```bash
curl -fsSL https://get.docker.com | sh
sudo systemctl start docker
sudo systemctl enable docker
```

### 2. 添加 Swap（内存不足时推荐）

```bash
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### 3. 安装 PostgreSQL

```bash
sudo apt update
sudo apt install -y postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

设置密码：

```bash
sudo -u postgres psql -c "ALTER USER postgres WITH PASSWORD '你的密码';"
```

### 4. 安装 Redis

```bash
sudo apt install -y redis-server
sudo systemctl start redis-server
sudo systemctl enable redis-server
```

验证：

```bash
redis-cli ping
# 应返回 PONG
```

### 5. 开放防火墙端口

```bash
sudo ufw allow 8000/tcp
sudo ufw reload
```

> [!warning] 安全提示
> - 只需开放 Sub2API 端口（如 8000）和 SSH（22）
> - **不要开放** PostgreSQL（5432）和 Redis（6379）端口
> - 云服务器还需在安全组中放行对应端口

### 6. 一键安装 Sub2API

```bash
curl -sSL https://raw.githubusercontent.com/Wei-Shaw/sub2api/main/deploy/install.sh | sudo bash
```

安装脚本交互配置：

| 配置项 | 建议值 |
|--------|--------|
| 监听地址 | `0.0.0.0`（回车默认） |
| 端口 | `8000`（或自定义） |
| 数据库主机 | `localhost` |
| 数据库端口 | `5432` |
| 数据库用户 | `postgres` |
| 数据库密码 | 你刚才设置的密码 |
| 数据库名 | `sub2api` |
| Redis 地址 | `localhost:6379` |
| Redis 密码 | 留空（无密码） |

### 7. Web 安装向导

访问 `http://服务器IP:8000`，按向导完成配置：

1. **数据库配置** — 填入 PostgreSQL 信息，点「测试连接」
2. **Redis 配置** — 保持默认，**关闭 TLS**，点「测试连接」
3. **管理员账户** — 设置管理员邮箱和密码
4. **准备安装** — 确认并完成安装

> [!tip] Redis 连接失败？
> 本地 Redis 不需要 TLS，确保「启用 TLS」选项是**关闭**的。

### 8. 启动服务

```bash
sudo systemctl start sub2api
sudo systemctl enable sub2api
```

查看状态：

```bash
sudo systemctl status sub2api
```

## 使用方法

### 添加 Claude 账号

1. 登录管理后台
2. 点击「账户管理」→「添加账户」
3. 选择账户类型（OAuth 或 API Key）
4. 按提示完成授权

### 生成 API Key

1. 在「用户管理」中分配额度
2. 生成 API Key（`sk-xxx` 格式）

### Claude Code 配置

```bash
export ANTHROPIC_BASE_URL="http://服务器IP:8000"
export ANTHROPIC_AUTH_TOKEN="sk-你生成的key"
```

然后正常使用 `claude` 命令即可。

## 常用命令

```bash
# 查看状态
sudo systemctl status sub2api

# 查看日志
sudo journalctl -u sub2api -f

# 重启服务
sudo systemctl restart sub2api

# 升级（一键脚本安装方式）
# 在 Web 管理后台左上角点「检查更新」
```

## Docker 部署方式（备选）

```bash
mkdir -p ~/sub2api-deploy && cd ~/sub2api-deploy
curl -sSL https://raw.githubusercontent.com/Wei-Shaw/sub2api/main/deploy/docker-deploy.sh | bash
docker compose up -d
docker compose logs -f sub2api
```

Docker 常用命令：

```bash
docker compose ps          # 查看状态
docker compose logs -f     # 查看日志
docker compose restart     # 重启
docker compose down        # 停止
docker compose pull && docker compose up -d  # 升级
```

## 内存占用参考（1 核 1GB 服务器）

```
Sub2API:       ~100-150MB
PostgreSQL:    ~30-50MB
Redis:         ~20-30MB
系统+已有:     ~300MB
Swap:          1GB (安全网)
─────────────────────────
总计:          ~450-530MB / 1024MB ✅
```

## 故障排除

### 安装阶段问题

#### PostgreSQL 服务不存在

```
Unit postgresql.service could not be found.
```

**原因**：PostgreSQL 未安装。

```bash
sudo apt update
sudo apt install -y postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

#### PostgreSQL 密码认证失败

```
Connection failed: ping failed: pq: password authentication failed for user "postgres"
```

**原因**：密码不匹配或认证方式不对。

```bash
# 重设密码
sudo -u postgres psql -c "ALTER USER postgres WITH PASSWORD '123456';"

# 如果还不行，修改认证方式
sudo nano /etc/postgresql/*/main/pg_hba.conf
# 找到: local all postgres peer
# 改为: local all postgres md5
sudo systemctl restart postgresql
```

#### Redis 连接被拒绝

```
Connection failed: ping failed: dial tcp 127.0.0.1:6379: connect: connection refused
```

**原因**：Redis 未安装。

```bash
sudo apt install -y redis-server
sudo systemctl start redis-server
sudo systemctl enable redis-server
```

#### Redis 连接超时

```
Connection failed: ping failed: context deadline exceeded
```

**原因**：Web 向导中启用了 TLS，但本地 Redis 不支持 TLS。

**解决**：在 Web 向导的 Redis 配置页面，**关闭「启用 TLS」选项**。

#### 端口无法访问

```
浏览器访问 http://服务器IP:8000 超时
```

**检查清单**：
1. 系统防火墙：`sudo ufw allow 8000/tcp && sudo ufw reload`
2. 云安全组：在云控制台放行 8000 端口（入站）
3. 确认服务运行：`sudo systemctl status sub2api`

> [!warning] 端口安全
> - 只需开放 Sub2API（8000）和 SSH（22）
> - **不要开放** PostgreSQL（5432）和 Redis（6379），开了有安全风险

### 使用阶段问题

#### API 403 余额不足

```
API Error: 403 Insufficient account balance
```

**原因**：Sub2API 用户没有分配额度。

**解决**：
1. 登录管理后台 →「用户管理」
2. 编辑用户 → 设置余额（如 `10000`）
3. 或开启 Simple 模式跳过计费：
   ```bash
   # 编辑 sub2api 配置或环境变量
   export RUN_MODE=simple
   export SIMPLE_MODE_CONFIRM=true
   sudo systemctl restart sub2api
   ```

#### 503 没有可用账号

```
HTTP 503: No available accounts: no available accounts
```

**原因**：没有添加上游 Claude 账号，或账号未分配到用户所在的分组。

**解决**：
1. 「账户管理」→ 添加 Claude 账号（OAuth 或 API Key）
2. 确保账号的**分组**和用户的**分组一致**
3. 账号状态为「启用」

```
账号（Claude）→ 分组 A
用户（你）    → 分组 A  ✅ 必须匹配
```

#### 忘记管理员密码

```bash
sudo journalctl -u sub2api | grep "admin password"
```

### 常见问题速查表

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| PostgreSQL 服务不存在 | 未安装 | `apt install postgresql` |
| PostgreSQL 认证失败 | 密码错误或认证方式 | 重设密码 + 改 pg_hba.conf 为 md5 |
| Redis 连接被拒绝 | 未安装 | `apt install redis-server` |
| Redis 连接超时 | TLS 未关闭 | Web 向导中关闭 TLS |
| 端口无法访问 | 防火墙/安全组 | `ufw allow 8000/tcp` + 云安全组放行 |
| 403 余额不足 | 用户未分配额度 | 后台充值或开启 Simple 模式 |
| 503 没有可用账号 | 未添加账号或分组不匹配 | 添加账号 + 分组一致 |
| 忘记管理员密码 | - | 查看日志获取 |

## 参考链接

- [GitHub 仓库](https://github.com/Wei-Shaw/sub2api)
- [知乎部署教程](https://zhuanlan.zhihu.com/p/2032101946493027471)
- [CRS vs Sub2API 区别](https://github.com/Wei-Shaw/claude-relay-service/issues/1076)
