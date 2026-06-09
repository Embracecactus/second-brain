---
title: kitware-archive.sh 脚本知识提取
date: 2026-06-09
tags:
  - shell
  - apt
  - kitware
  - cmake
  - ubuntu
  - 软件源配置
category: DevOps/脚本
source: /home/lijian/kitware-archive.sh
---

# kitware-archive.sh 脚本知识提取

## 概述

这是一个用于在 Ubuntu 系统上配置 Kitware APT 软件源的 Shell 脚本。Kitware 是 CMake 的官方维护者，该脚本帮助用户添加 Kitware 的官方 APT 源以获取最新版本的 CMake 及其相关工具。

## 核心功能

- 自动检测 Ubuntu 版本并配置对应的软件源
- 安装 Kitware 的 GPG keyring 以验证软件包签名
- 支持配置 Release Candidate (RC) 版本的软件源

## 关键参数

| 参数 | 说明 |
|------|------|
| `--release <ubuntu-release>` | 指定 Ubuntu 版本代号（如 noble、jammy、focal） |
| `--rc` | 是否添加 Release Candidate 版本的软件源 |
| `--help` | 显示帮助信息 |

## 支持的 Ubuntu 版本

- **Noble** (24.04)
- **Jammy** (22.04)
- **Focal** (20.04)

## 详细流程

### 1. 参数解析
脚本使用循环解析命令行参数，支持：
- `--release` 指定 Ubuntu 版本
- `--rc` 启用 RC 版本源
- `--help` 显示帮助

### 2. 版本检测
如果未指定 `--release` 参数，脚本会：
- 读取 `/etc/os-release` 文件
- 提取 `UBUNTU_CODENAME` 变量
- 验证是否为 Ubuntu 系统

### 3. 版本验证
只支持三个 Ubuntu 版本，其他版本会报错退出：
```bash
case "${release}" in
noble|jammy|focal)
  # 支持的版本
  ;;
*)
  echo "Only Ubuntu Noble (24.04), Jammy (22.04), and Focal (20.04) are supported. Aborting."
  exit 1
  ;;
esac
```

### 4. Keyring 安装
检查 `kitware-archive-keyring` 包是否已安装：
- 如果未安装，添加必要的依赖包：`ca-certificates`、`gpg`、`wget`
- 下载并安装 Kitware 的 GPG key

### 5. 配置软件源
创建 `/etc/apt/sources.list.d/kitware.list` 文件：
- 主源：`https://apt.kitware.com/ubuntu/ ${release} main`
- 如果启用 `--rc`，添加 RC 源：`https://apt.kitware.com/ubuntu/ ${release}-rc main`

### 6. 最终安装
- 更新 APT 缓存
- 安装 `kitware-archive-keyring` 包
- 清理临时 GPG key 文件

## 技术要点

- 使用 `set -eu` 确保脚本在错误时立即退出
- 使用 `set -x` 启用调试模式显示执行命令
- GPG key 存储在 `/usr/share/keyrings/kitware-archive-keyring.gpg`
- 软件源配置使用 `signed-by` 选项确保安全性

## 使用示例

```bash
# 自动检测 Ubuntu 版本并配置
sudo ./kitware-archive.sh

# 指定 Ubuntu 版本
sudo ./kitware-archive.sh --release jammy

# 同时添加 RC 版本源
sudo ./kitware-archive.sh --release noble --rc
```

## 注意事项

1. 需要 root 权限运行（使用 sudo）
2. 仅支持 Ubuntu 系统
3. 仅支持 20.04、22.04、24.04 三个版本
4. 脚本会修改系统 APT 源配置
5. 执行后需要运行 `apt-get update` 更新缓存
