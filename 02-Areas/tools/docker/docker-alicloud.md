---
tags:
  - docker
  - alibaba-cloud
  - acr
  - container-registry
  - zephyr
  - ci
category: tools/docker
created: 2026-06-09
aliases:
  - 阿里云镜像推送
  - Docker Push to ACR
---

# Docker 镜像推送到阿里云 ACR

## 项目概述

本笔记记录了将本地构建的 Docker 镜像推送到**阿里云容器镜像服务 ACR (Alibaba Cloud Container Registry)** 个人版的完整流程。以 `zephyr-ci` 镜像（包含 Zephyr SDK 0.17.4 的 CI 环境）为例，涵盖从 ACR 控制台配置到命令行推送/拉取的全链路操作。

> [!info] 运行环境
> WSL2 (Windows Subsystem for Linux 2)，宿主机 `DESKTOP-0B2000K`，用户 `liijian`。

---

## 关键知识点

### 阿里云容器服务全景

阿里云提供了完整的容器产品矩阵：

| 服务 | 缩写 | 说明 |
|---|---|---|
| 容器服务 Kubernetes 版 | ACK | 一站式管理容器应用的 K8s 服务 |
| 容器计算服务 | ACS | 以 K8s 为界面供给容器算力资源 |
| **容器镜像服务** | **ACR** | **容器镜像的安全托管及高效分发平台** |
| 服务网格 | ASM | 兼容 Istio 的托管式微服务流量管理平台 |
| 弹性容器实例 | ECI | Serverless 容器化弹性计算服务 |
| 分布式云容器平台 | ACK One | 混合云、多集群、分布式容器平台 |

### ACR 个人版核心限制

- **命名空间**：每个账号最多 **3 个**命名空间（类似项目名称）
- **镜像仓库**：每个实例最多 **300 个**镜像仓库
- **命名空间规则**：长度 2-30 位，仅限小写英文字母、数字，分隔符 `-` 和 `_`（不能在首位或末位），**设置后不可修改**
- **仓库名称规则**：长度 2-64 位，小写英文字母、数字，分隔符 `_`、`-`、`.`（不能在首位或末位）
- **仓库类型**：公开 / 私有
- **摘要**：最长 100 个字符
- **描述**：支持 Markdown 格式

### 代码源选项

创建镜像仓库时可选择的代码源：
- **Codeup** - 阿里云代码托管
- **GitHub**
- **Bitbucket**
- **私有 GitLab**
- **本地仓库** - 通过命令行推送镜像

---

## 技术细节

### 完整操作流程（五步法）

#### 第一步：进入 ACR 控制台

1. 阿里云首页 -> 产品 -> 容器 -> **容器镜像服务 ACR**
2. 实例列表中选择 **个人版**

#### 第二步：创建命名空间

- 进入左侧菜单「仓库管理」->「命名空间」
- 点击「创建命名空间」
- 本例中命名空间为 `lijian_docker`
- 可设置自动创建仓库的默认配置（公开/私有）

#### 第三步：创建镜像仓库

- 进入左侧菜单「仓库管理」->「镜像仓库」
- 点击「创建镜像仓库」
- 填写信息：
  - **地域**：华东1（杭州）
  - **命名空间**：lijian_docker
  - **仓库名称**：zephyr-ci
  - **仓库类型**：私有
  - **摘要**：zephyr-ci的镜像包含sdkzephyr-sdk-0.17.4_
- 代码源选择「本地仓库」
- 创建完成后可在仓库详情页查看操作指南

#### 第四步：推送镜像

在 WSL2 终端中依次执行：

```bash
# 1. 登录阿里云 ACR（需要输入密码）
docker login --username=wjhgvpqm crpi-njzo90d6qanuirs8.cn-hangzhou.personal.cr.aliyuncs.com

# 2. 查看本地镜像，找到目标 IMAGE ID
docker images

# 3. 给镜像打标签（tag）
docker tag f4f942002d29 crpi-njzo90d6qanuirs8.cn-hangzhou.personal.cr.aliyuncs.com/lijian_docker/zephyr-ci:v1.0

# 4. 推送镜像到 ACR
docker push crpi-njzo90d6qanuirs8.cn-hangzhou.personal.cr.aliyuncs.com/lijian_docker/zephyr-ci:v1.0
```

#### 第五步：拉取镜像

在任何可联网的机器上执行：

```bash
docker pull crpi-njzo90d6qanuirs8.cn-hangzhou.personal.cr.aliyuncs.com/lijian_docker/zephyr-ci:[镜像版本号]
```

### ACR Registry URL 解析

```
crpi-njzo90d6qanuirs8.cn-hangzhou.personal.cr.aliyuncs.com
├── crpi-njzo90d6qanuirs8  → 实例 ID
├── cn-hangzhou             → 地域（华东1杭州）
├── personal                → 实例类型（个人版）
└── cr.aliyuncs.com         → ACR 域名
```

### 本地镜像信息

| Repository | TAG | IMAGE ID | Size |
|---|---|---|---|
| zephyr-ci | v1.0 | f4f942002d29 | 20.1GB |
| zephyr-ci-base | v1.0 | d6f99b80b0d0 | 8.05GB |
| docker.1ms.run/zephyrprojectrtos/zephyr-build | v0.28.7 | ae780e2bf991 | 20.3GB |

> [!note] 关于镜像来源
> `zephyr-ci` 是基于 `zephyrprojectrtos/zephyr-build:v0.28.7` 构建的自定义 CI 镜像，包含 Zephyr SDK 0.17.4。

### 推送过程中的 Layer 状态

推送大镜像时会看到多个 layer 并行上传：

| 状态 | 含义 |
|---|---|
| `Pushed` | 该 layer 已上传完成 |
| `Pushing [===>]` | 正在上传，显示进度条 |
| `Waiting` | 排队等待上传 |

> [!warning] 大镜像推送注意
> `zephyr-ci` 镜像约 20.1GB，推送耗时较长。网络不稳定时可能出现中断，需要重新执行 `docker push` 命令（已完成的 layer 会跳过）。

---

## 代码/配置片段

### ACR 登录命令模板

```bash
docker login --username=<阿里云用户名> <实例ID>.<地域>.personal.cr.aliyuncs.com
```

### 镜像打标签命令模板

```bash
docker tag <本地IMAGE_ID> <ACR仓库地址>/<命名空间>/<仓库名>:<版本号>
```

### 镜像推送命令模板

```bash
docker push <ACR仓库地址>/<命名空间>/<仓库名>:<版本号>
```

### 镜像拉取命令模板

```bash
docker pull <ACR仓库地址>/<命名空间>/<仓库名>:<版本号>
```

### 完整示例

```bash
# 登录
docker login --username=wjhgvpqm crpi-njzo90d6qanuirs8.cn-hangzhou.personal.cr.aliyuncs.com

# 打标签
docker tag f4f942002d29 crpi-njzo90d6qanuirs8.cn-hangzhou.personal.cr.aliyuncs.com/lijian_docker/zephyr-ci:v1.0

# 推送
docker push crpi-njzo90d6qanuirs8.cn-hangzhou.personal.cr.aliyuncs.com/lijian_docker/zephyr-ci:v1.0

# 拉取
docker pull crpi-njzo90d6qanuirs8.cn-hangzhou.personal.cr.aliyuncs.com/lijian_docker/zephyr-ci:v1.0
```

---

## 相关链接

- [阿里云容器镜像服务 ACR 控制台](https://cr.console.aliyun.com/)
- [阿里云 ACR 个人版文档](https://help.aliyun.com/product/149654.html)
- [Docker 官方文档 - docker push](https://docs.docker.com/engine/reference/commandline/push/)
- [Docker 官方文档 - docker tag](https://docs.docker.com/engine/reference/commandline/tag/)
- [Zephyr Project](https://www.zephyrproject.org/)
- [Zephyr SDK Docker Images](https://github.com/zephyrproject-rtos/docker-image)
