---
tags: [cmake, apt, ubuntu, kitware, shell, 软件源配置, DevOps]
category: tools
created: 2026-06-09
updated: 2026-06-09
status: active
source: /home/lijian/kitware-archive.sh
---

# CMake APT 安装配置 — Kitware 官方源脚本解析

## 项目/工具概述

`kitware-archive.sh` 是 Kitware 官方提供的 Shell 脚本，用于在 Ubuntu 系统上一键配置 Kitware APT 软件源。Kitware 是 CMake 的创建者和官方维护者，该脚本允许用户绕过 Ubuntu 官方仓库中较旧的 CMake 版本，直接从 Kitware 官方源安装最新稳定版或 Release Candidate 版本的 CMake。脚本设计精简（100 行纯 POSIX Shell），覆盖了从 GPG 密钥管理、仓库列表配置到 keyring 包安装的完整流程，是 C++ / 嵌入式 / 跨平台项目开发环境中常见的基础设施配置步骤。

## 技术栈 / 关键特性

| 类目 | 说明 |
|------|------|
| **脚本语言** | POSIX Shell (`#!/bin/sh`)，无 Bash 特有语法，兼容性极强 |
| **包管理** | APT (`apt-get`) |
| **GPG 密钥管理** | `gpg --dearmor` + `/usr/share/keyrings/` 存储架构 |
| **软件源格式** | DEB822 `/etc/apt/sources.list.d/kitware.list` |
| **安全机制** | `signed-by=` 指定密钥文件，避免全局信任 |
| **错误处理** | `set -eu`（未定义变量报错 + 管道错误退出） |
| **调试支持** | `set -x`（脚本进入实际操作阶段后开启） |

### 关键特性

- **自动版本检测** — 读取 `/etc/os-release` 中的 `UBUNTU_CODENAME`，无需手动指定
- **RC 版本支持** — `--rc` 标志可同时添加 Release Candidate 仓库通道
- **幂等安全** — 检查 `kitware-archive-keyring` 版权文件判断是否已安装，避免重复操作
- **临时密钥清理** — 下载的 GPG key 在安装 keyring 包后立即删除

## 架构与设计

脚本采用经典的三段式结构：

```
参数解析 → 环境检测与验证 → 执行安装
```

```
┌─────────────────────────────────────────────────────┐
│  Phase 1: 参数解析 (L9-34)                          │
│  --release <codename>  --rc  --help                 │
│  使用状态机模式逐个处理 $@                           │
└────────────────────┬────────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────────┐
│  Phase 2: 环境检测与验证 (L36-73)                    │
│  若未指定 release → 读 /etc/os-release              │
│  验证 codename ∈ {noble, jammy, focal}              │
│  检查 kitware-archive-keyring 是否已安装             │
└────────────────────┬────────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────────┐
│  Phase 3: 执行安装 (L75-100)                        │
│  1. apt-get install 依赖包                          │
│  2. 下载 GPG key → gpg --dearmor                    │
│  3. 写入 kitware.list（含 signed-by）                │
│  4. apt-get update → 安装 keyring 包 → 清理临时 key  │
└─────────────────────────────────────────────────────┘
```

## 核心知识点

### 1. POSIX Shell 状态机参数解析

脚本没有使用 `getopt` 或 `getopts`，而是用一个 `doing` 变量实现状态机：

```sh
for opt in "$@"
do
  case "${doing}" in
  release)
    release="${opt}"   # 上一轮遇到 --release，本轮取值
    doing=
    ;;
  "")
    case "${opt}" in
    --rc)      rc=1 ;;
    --release) doing=release ;;
    --help)    help=1 ;;
    esac
    ;;
  esac
done
```

这种方式的优点是零外部依赖、纯 POSIX 兼容；缺点是没有对未知参数报错。适合简单脚本，复杂场景建议使用 `getopt`。

### 2. GPG 密钥管理与 `signed-by` 安全架构

现代 Debian/Ubuntu 推荐的 APT 密钥管理方式：

```sh
# 下载 → 解码 → 存入独立 keyring 文件
wget -O - https://apt.kitware.com/keys/kitware-archive-latest.asc \
  2>/dev/null | gpg --dearmor - > /usr/share/keyrings/kitware-archive-keyring.gpg

# sources.list 中通过 signed-by 指定信任范围
echo "deb [signed-by=/usr/share/keyrings/kitware-archive-keyring.gpg] \
  https://apt.kitware.com/ubuntu/ ${release} main" > /etc/apt/sources.list.d/kitware.list
```

> [!important] 为什么用 `signed-by` 而非 `apt-key add`
> `apt-key add` 将密钥加入全局信任环，任何仓库都能用该密钥签名软件包，存在供应链攻击风险。`signed-by` 将密钥绑定到特定仓库，遵循最小权限原则。`apt-key` 已在 Debian 11 / Ubuntu 22.04 起被弃用。

### 3. `/etc/os-release` 版本检测

```sh
unset UBUNTU_CODENAME
. /etc/os-release          # source 该文件，获取系统变量

if [ -z "${UBUNTU_CODENAME+x}" ]   # +x 判断变量是否 set（即使是空串）
then
  echo "This is not an Ubuntu system. Aborting." >&2
  exit 1
fi
release="${UBUNTU_CODENAME}"
```

> [!tip] `${var+x}` vs `${var:-default}`
> `${UBUNTU_CODENAME+x}` 在变量未定义时展开为空串（不触发 `set -u`），在变量已定义时展开为 `x`。这是 POSIX Shell 中检测变量是否 set 的标准惯用法。

### 4. `set -eu` 的管道陷阱

脚本使用 `set -eu`，但第 89 行的管道命令需要注意：

```sh
wget -O - https://apt.kitware.com/keys/kitware-archive-latest.asc 2>/dev/null \
  | gpg --dearmor - > /usr/share/keyrings/kitware-archive-keyring.gpg
```

在 `set -e` 下，管道中只有**最后一个命令**的退出状态会被检查。如果 `wget` 失败但 `gpg` 成功（退出 0），脚本不会报错。使用 `set -o pipefail`（Bash 扩展）可以修复此问题，但此脚本选择 POSIX 兼容，因此依赖 `wget` 的错误输出重定向到 `/dev/null` 来隐藏警告。

### 5. Keyring 包的安装策略

脚本先用临时 GPG key 完成首次 `apt-get update`，然后安装 `kitware-archive-keyring` 系统包，最后删除临时 key：

```sh
apt-get update                                            # 用临时 key 刷新
test -n "${get_keyring}" && rm /usr/share/keyrings/kitware-archive-keyring.gpg  # 删除临时
apt-get install -y kitware-archive-keyring                # 安装正式 keyring 包
```

`kitware-archive-keyring` 是 Kitware 维护的 Debian 包，包含其 GPG 公钥，安装后存放在 `/usr/share/doc/kitware-archive-keyring/copyright`。后续更新通过 APT 包管理自动维护，无需手动管理密钥。

## 关键配置片段

### 生成的 APT 源文件

文件路径：`/etc/apt/sources.list.d/kitware.list`

```
# 稳定版源
deb [signed-by=/usr/share/keyrings/kitware-archive-keyring.gpg] https://apt.kitware.com/ubuntu/ noble main

# 若启用 --rc，追加 RC 源
deb [signed-by=/usr/share/keyrings/kitware-archive-keyring.gpg] https://apt.kitware.com/ubuntu/ noble-rc main
```

### 安装后验证

```bash
# 检查源是否生效
apt-cache policy cmake

# 安装最新 CMake
sudo apt-get install cmake

# 验证版本
cmake --version
```

## 使用方法 / 构建步骤

```bash
# 基本用法 — 自动检测 Ubuntu 版本
sudo ./kitware-archive.sh

# 指定 Ubuntu 版本
sudo ./kitware-archive.sh --release jammy

# 同时添加 RC 版本源
sudo ./kitware-archive.sh --release noble --rc

# 安装完成后
sudo apt-get install cmake
cmake --version
```

### 支持的 Ubuntu 版本

| 代号 | 版本号 | 状态 |
|------|--------|------|
| **Noble** | 24.04 LTS | 当前推荐 |
| **Jammy** | 22.04 LTS | 长期支持 |
| **Focal** | 20.04 LTS | 维护中 |

### 注意事项

1. **需要 root 权限** — 脚本修改 `/etc/apt/sources.list.d/` 和 `/usr/share/keyrings/`，必须以 `sudo` 运行
2. **仅限 Ubuntu** — 脚本通过 `UBUNTU_CODENAME` 检测系统，Debian / Linux Mint 等不直接兼容
3. **非 LTS 版本不支持** — 如 23.10 (mantic) 等非 LTS 版本不在支持列表中
4. **幂等执行** — 重复运行安全，已有 keyring 时跳过密钥下载步骤
5. **网络依赖** — 需要访问 `apt.kitware.com`，确保网络可达

## 相关笔记

- [[kitware-archive]] — 脚本详细知识提取笔记
- [[make]] — GNU Make 构建系统学习（CMake 的底层替代方案）
- [[qt-projects]] — LinuxCNC CAM 项目（CMake + Qt6 构建）
- [[rv1126b]] — RV1126B 运动相机项目（CMake 交叉编译实践）
- [[zephyr]] — Zephyr RTOS 项目笔记（CMake + West 构建系统）
- [[TuyaOpen]] — 涂鸦 IoT 框架（CMake 构建系统）
