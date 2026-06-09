---
tags:
  - boardroot
  - methodology
  - planning
  - embedded-linux
  - rootfs
  - debootstrap
  - rockchip
  - composable
  - vendor-bsp
  - apt
  - chroot
category: embedded-linux/rockchip
created: 2026-06-09
updated: 2026-06-09
status: active
---

# BoardRoot — 嵌入式 Linux 厂商适配框架方法论

## 项目/工具概述

BoardRoot 是一个**可组合的 Ubuntu/Debian 根文件系统构建框架（composable rootfs builder）**，专为嵌入式 Linux 板卡设计。它构建于 `debootstrap` 基础之上，利用 Ubuntu/Debian 的 APT 生态系统为嵌入式设备提供可复现的 rootfs 构建能力。与 Buildroot 的"每个包单独编写"模式不同，BoardRoot 的核心价值在于通过分层配置模型将发行版、芯片厂商、板级硬件、产品定义、构建配置分离，同时提供三种厂商适配机制将闭源 BSP 二进制文件无缝集成到标准 Ubuntu rootfs 中。

项目路径：`/home/lijian/project/rv1126b/BoardRoot`
首个参考目标：Rockchip RV1126B SportCam
SDK 路径：`/home/lijian/project/rv1126b/rv1126bsportCam/sdk/atk_dlrv1126b_linux6.1_sdk_release_v1.2.1_20260327/`

## 技术栈 / 关键特性

| 层面 | 技术选型 |
|------|----------|
| 基础 rootfs | debootstrap (Ubuntu Jammy / Debian Bookworm, ARM64) |
| 跨架构模拟 | qemu-aarch64-static (second-stage chroot) |
| 包管理 | APT (标准 Ubuntu 仓库 + 本地 .deb 仓库) |
| 配置格式 | YAML manifest + Kconfig (menuconfig TUI) |
| YAML 解析 | awk 纯文本解析（无外部 YAML 库依赖） |
| 系统配置 | systemd-networkd, /etc/fstab, /etc/hostname |
| 脚本规范 | Bash + `set -euo pipefail`，所有脚本使用 `#!/usr/bin/env bash` |
| 组件依赖 | Kahn 算法（拓扑排序） |
| 验证体系 | 离线验证 (28+ check 脚本) + 目标板验证 + 单元测试 (290+) |
| CI/CD | GitHub Actions (push/PR 触发 validate + dry-run) |
| 许可证 | Apache-2.0 |

## 架构与设计

### 核心组合模型

BoardRoot 镜像由以下层次组合而成：

```
rootfs = distro + platform + board + product + profile + components
```

### 七层抽象模型

| 层次 | 职责 | 示例 |
|------|------|------|
| **Distro** | 发行版选择、架构、APT mirror | Ubuntu Jammy ARM64 |
| **Core** | 通用构建引擎（vendor-neutral） | debootstrap, chroot, overlay |
| **Platform** | 芯片厂商 BSP 集成 | rockchip-rv1126b, generic-arm64 |
| **Board** | 板级硬件差异 | rv1126b-reference, qemu-arm64 |
| **Product** | 产品使用场景 | minimal-system, sportcam, ipcamera |
| **Profile** | 构建变体 | debug, release, factory, recovery |
| **Components** | 可复用安装特性 | firmware, rockchip-mpp, nginx, docker-ce |

设计原则：每个文件、包、服务、脚本必须属于一个明确的层次。厂商 BSP 逻辑不得泄漏到 `core/` 构建引擎中。

### 三阶段构建流水线

```
debootstrap → configure-system → install-components
```

1. **debootstrap** (`core/debootstrap-rootfs.sh`) — `debootstrap --foreign` + second-stage chroot (qemu-aarch64-static)
2. **configure-system** (`core/configure-system.sh`) — 编排 hostname, apt, fstab, network, users 配置
3. **install-components** (`core/install-components.sh`) — 按依赖顺序遍历启用的组件，执行 install.sh + validate.sh

可选的第四步：`core/make-ext4.sh` 生成 ext4 镜像文件。

### 三种厂商适配机制

| 机制 | source.type | 描述 | 适用场景 |
|------|-------------|------|----------|
| **二进制覆盖** | `sdk-path` / `buildroot-output` | 直接复制 .so/.ko/firmware 到 rootfs | 固件、内核模块、闭源库 |
| **源码补丁** | `apt-source-patch` | 下载 Ubuntu 源码包 → 应用厂商 patch → dpkg-buildpackage 编译为 .deb | GStreamer 插件、MPP、RGA |
| **Buildroot 桥接** | `buildroot-deb` / `buildroot-deb-package` | 将 Buildroot 交叉编译的 .deb 导入为本地 APT 仓库 | 厂商专有库、开发头文件 |

前两种机制产出 `.deb` 包，通过 APT 统一管理。

### 目录结构

```
configs/            层配置片段和构建预设
  distros/          发行版配置 (.mk)
  platforms/        芯片平台配置 (.mk)
  boards/           板级配置 (.mk)
  products/         产品配置 (.mk)
  profiles/         构建变体配置 (.mk)
  presets/          完整构建预设 (.yaml)
  kconfig/          Kconfig 定义文件
  defconfigs/       预设配置和片段 (fragments)
core/               通用 rootfs 构建引擎脚本
components/         可复用安装/验证单元
packages/           按用途分组的包列表 (.list)
examples/           示例构建 manifest
schemas/            JSON schema 配置验证
validation/         离线和目标板验证脚本
  offline/          28+ 主机端检查脚本
  target/           目标板运行时验证
docs/               架构、移植、贡献文档
vendors/            厂商 SDK 检测/导入脚本
output/             生成的镜像、日志、manifest、报告
  images/           rootfs 和固件镜像
  logs/             构建日志
  manifests/        解析后的构建 manifest
  reports/          大小、overlay、验证报告
```

## 核心知识点

### 1. Manifest 格式与 system 命名空间

BoardRoot 使用 YAML manifest 驱动全部构建配置。`system:` 命名空间接受运行时系统配置：

```yaml
system:
  hostname: sportcam
  apt:
    mirror: http://ports.ubuntu.com/ubuntu-ports/
  users:
    root:
      password: root
    ubuntu:
      password: ubuntu
      groups: sudo
      shell: /bin/bash
  fstab:
    - source: /dev/mmcblk0p2
      mountpoint: /
      type: ext4
      options: defaults
      dump: 0
      pass: 1
  network:
    renderer: networkd
    interfaces:
      - name: eth0
        dhcp4: true
```

兼容性规则：
- `system.hostname` 覆盖旧版 `board.hostname`
- `system.users` 覆盖旧版顶层 `users`
- `system.apt.mirror` 覆盖 `distro.mirror`（用于 configure-apt.sh）
- `distro.mirror` 仍由 debootstrap-rootfs.sh 使用

### 2. 组件生命周期与 v2 元数据

每个组件遵循 `prepare → install → validate` 生命周期，目录结构：

```
components/<name>/
  component.yaml    # 元数据：name, mode, vendor, source, install, runtime, validate
  install.sh        # 安装脚本（可执行）
  validate.sh       # 验证脚本（可执行，可选）
```

component.yaml v2 格式示例：

```yaml
name: rockchip-mpp
version: 1.0.0
description: Rockchip Media Process Platform library
mode: copy
vendor: rockchip
source:
  type: buildroot-output
  vendor: rockchip
  path: buildroot/output/rv1126b_sportcam/target
runtime:
  requires:
    components:
      - firmware
install:
  mode: copy
  files:
    - src: usr/lib/librockchip_mpp.so*
      dest: usr/lib/
      mode: "0644"
  packages:
    - librockchip-mpp-dev
validate:
  offline:
    - component-metadata-coherent
  target:
    - path-exists: usr/lib/librockchip_mpp.so
    - executable: usr/bin/mpi_dec_test
```

v2 元数据一致性规则：`source`, `install`, `runtime`, `validate` 四个 section 必须全部存在或全部不存在（all-or-nothing coherence）。

### 3. 组件依赖解析

使用 Kahn 算法（拓扑排序）进行依赖解析：
- 依赖图从所有已启用组件的 `runtime.requires.components` 构建
- 检测循环依赖并报错
- 例：`rockchip-mpp` 依赖 `firmware` → firmware 先安装

### 4. YAML 解析器（纯 awk 实现）

`core/manifest.sh` 使用 awk 实现全部 YAML 解析，零外部依赖。关键函数：

| 函数 | 用途 |
|------|------|
| `manifest_get_scalar` | 获取顶层标量值 |
| `manifest_get_section_scalar` | 获取 section 下的标量值 |
| `manifest_get_nested_scalar` | 获取嵌套 section 的标量值 |
| `manifest_get_hostname` | 获取 hostname（system 优先，board 兜底） |
| `manifest_get_apt_mirror` | 获取 APT mirror（system.apt 优先，distro 兜底） |
| `manifest_get_users` | 获取用户列表 |
| `manifest_get_user_value` | 获取用户特定字段 |
| `manifest_get_components` | 获取组件列表 |
| `manifest_get_component_value` | 获取组件特定字段 |
| `manifest_get_fstab_entries` | 解析 fstab 条目 |
| `manifest_get_network_interfaces` | 解析网络接口配置 |
| `component_v2_metadata_is_coherent` | 验证 v2 元数据一致性 |

### 5. component-copy.sh 共享库

`core/component-copy.sh` 提供 copy-mode 组件安装器的共享逻辑：

```bash
# 组件 install.sh 模板
#!/usr/bin/env bash
set -euo pipefail
root_dir="$(cd "$(dirname "${BASH_SOURCE[0]}")/../.." && pwd)"
. "$root_dir/core/common.sh"
. "$root_dir/core/component-copy.sh"

boardroot_parse_component_copy_args "components/<name>/component.yaml" "$@"
boardroot_copy_component_files
```

核心函数：
- `boardroot_parse_component_copy_args` — 解析 --rootfs, --vendor-sdk, --config 参数
- `boardroot_copy_component_files` — 按 component.yaml 的 files 列表逐个复制文件
- `boardroot_copy_one_file` — 单文件复制，自动创建目标目录并设置权限

### 6. 安全加固要点

| 安全问题 | 修复方案 | 涉及文件 |
|----------|----------|----------|
| Shell injection（用户创建） | 使用 env 变量传递 + heredoc 脚本替代 `bash -c` 单引号插值 | `core/configure-rootfs.sh` |
| Hostname 注入 | RFC 1123 正则验证 + awk 替代 sed | `core/configure-rootfs.sh` |
| 网络配置注入 | CIDR/IP/接口名格式验证 | `core/configure-network.sh` |
| fstab 注入 | fstype 白名单 + mountpoint 绝对路径 + dump/passno 整数验证 | `core/configure-fstab.sh` |
| APT mirror 注入 | http/https URL 格式验证 + 换行符检测 | `core/configure-apt.sh` |
| sed 注入 | `escape_sed()` 函数转义特殊字符 | `core/kconfig-merge.sh` |

### 7. 可靠性机制

- **原子下载** — 使用 `.part` 临时文件 + `flock` 文件锁，下载完成后 mv 到缓存路径
- **并发构建保护** — `flock -n` 独占锁，第二个构建立即失败
- **信号处理** — `trap cleanup_on_exit EXIT`，SIGINT/SIGTERM 清理挂载点和临时文件
- **构建检查点** — `save_checkpoint` / `load_checkpoint` 实现断点续传
- **双重挂载防护** — `mountpoint -q` 检查后再 mount

### 8. Vendor SDK 桥接

BoardRoot 不要求将闭源 SDK 文件提交到仓库。厂商资产从本地 SDK 路径导入并记录在构建 manifest 中：

```yaml
# boardroot.local.yaml (gitignored)
vendors:
  rockchip:
    sdk_path: /opt/sdk/rockchip-rv1126b
```

生成解析后的可审计 manifest：

```bash
./build.sh use rockchip-rv1126b-debug
./build.sh resolve
./build.sh resolved
```

### 9. 组件分类

**Rockchip 厂商组件（binary overlay）：**
firmware, kernel-modules, rockchip-mpp, rockchip-rga, rockchip-isp-server, rockchip-npu-server, rockchip-audio, rockchip-iva, rockchip-update-tools, udev-rules, disk-helpers, gstreamer-rockchip

**通用组件（package-list / apt）：**
nginx, ssh-server, python-runtime, ntp-client, dev-tools, docker-ce, wifi-manager, bluetooth-stack, audio-manager, gpio-tools, camera-tools

**源码补丁组件（apt-source-patch）：**
gstreamer-rockchip-src, rockchip-mpp-src, rockchip-rga-src, gstreamer-plugins-base-rga

**Buildroot 桥接组件（buildroot-deb）：**
rockchip-debs, rockchip-rga-dev-deb, rockchip-mpp-dev-deb

## 关键代码/配置片段

### 完整 Manifest 示例

```yaml
distro:
  family: ubuntu
  release: jammy
  arch: arm64
platform:
  vendor: rockchip
  soc: rv1126b
board:
  name: sportcam
product:
  type: sportcam
profile:
  variant: release
system:
  hostname: sportcam
  apt:
    mirror: http://ports.ubuntu.com/ubuntu-ports/
  users:
    root:
      password: root
    ubuntu:
      password: ubuntu
      groups: sudo
      shell: /bin/bash
  fstab:
    - source: /dev/mmcblk0p2
      mountpoint: /
      type: ext4
      options: defaults
      dump: 0
      pass: 1
  network:
    renderer: networkd
    interfaces:
      - name: eth0
        dhcp4: true
components:
  firmware:
    enabled: true
  kernel-modules:
    enabled: true
  rockchip-mpp:
    enabled: true
  nginx:
    enabled: true
image:
  size_mb: 4096
```

### build.sh 核心命令

```bash
./build.sh list                  # 列出可用预设
./build.sh use <preset>          # 选择预设 → output/manifests/current.yaml
./build.sh plan                  # 打印构建计划
./build.sh dry-run               # 打印构建计划（无需 sudo/网络）
./build.sh resolve               # 合并 boardroot.local.yaml SDK 路径
./build.sh rootfs                # 完整构建：debootstrap + configure + install-components
./build.sh image                 # 从 rootfs 目录创建 ext4 镜像
./build.sh validate              # 委托给 make validate
./build.sh clean                 # 清理日志和报告
./build.sh distclean             # 清理全部生成输出（含镜像和 work 目录）
```

### 验证命令

```bash
cd /home/lijian/project/rv1126b/BoardRoot
make validate           # 44+ 离线验证目标
make test-unit          # 290+ 单元测试
make test-integration   # 集成测试
./build.sh dry-run --config examples/rockchip-rv1126b-sportcam-release.yaml
./build.sh plan --config examples/rockchip-rv1126b-sportcam-release.yaml
```

### 测试框架核心函数

```bash
# tests/lib/test-helpers.sh
run_test_suite()     # 运行测试套件，统计通过/失败
run_test()           # 运行单个测试，捕获输出和退出码
setup_test_env()     # 创建临时目录，设置环境变量
create_mock_rootfs() # 创建模拟 rootfs 目录结构

# tests/lib/assertions.sh
assert_equals(expected, actual)
assert_file_exists(path)
assert_file_contains(path, pattern)
assert_file_mode(path, mode)
assert_command_succeeds(cmd)
assert_component_valid(component_dir)
```

### import-buildroot.sh 核心逻辑

```bash
# 1. 从 Buildroot 输出目录找到 .deb 文件
# 2. 创建本地 APT 仓库: /usr/local/share/boardroot-repo/
# 3. 生成 Packages.gz
# 4. 添加 sources.list 条目
# 5. apt-get update && apt-get install
```

### patch-package.sh 核心逻辑

```bash
# 1. 添加 deb-src 行到 sources.list（临时）
# 2. chroot 中 apt-get update
# 3. chroot 中 apt-get source <package>
# 4. 应用 vendor patches（quilt import + push -a）
# 5. dpkg-buildpackage -b -uc
# 6. dpkg -i ../<package>.deb
```

### GitHub Actions CI

```yaml
name: BoardRoot Validation
on: [push, pull_request]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install dependencies
        run: sudo apt-get install -y debootstrap
      - name: Run validation
        run: make validate
      - name: Test generic dry-run
        run: ./build.sh dry-run --config examples/generic-arm64-debug.yaml
      - name: Test rockchip dry-run
        run: ./build.sh dry-run --config examples/rockchip-rv1126b-ubuntu-jammy-debug.yaml
```

## 使用方法 / 构建步骤

### 快速开始

```bash
cd /home/lijian/project/rv1126b/BoardRoot

# 1. 验证项目结构
make validate

# 2. 查看可用预设
./build.sh list

# 3. 选择预设
./build.sh use generic-arm64-debug

# 4. 查看构建计划
./build.sh plan

# 5. 干跑（无需 sudo 和网络）
./build.sh dry-run
```

### 完整构建流程

```bash
# 1. 配置本地 SDK 路径（gitignored）
cp boardroot.local.yaml.example boardroot.local.yaml
# 编辑 boardroot.local.yaml 设置 sdk_path

# 2. 选择 Rockchip 预设
./build.sh use rockchip-rv1126b-debug

# 3. 解析本地 SDK 路径
./build.sh resolve

# 4. 执行完整构建
./build.sh rootfs --config output/manifests/resolved.yaml

# 5. 创建 ext4 镜像
./build.sh image --config output/manifests/resolved.yaml
```

### 移植到新板卡

参考 `docs/porting-guide.md`，步骤如下：

1. 选择层次组合：`DISTRO + PLATFORM + BOARD + PRODUCT + PROFILE`
2. 添加 distro 配置到 `configs/distros/`
3. 添加 platform 配置到 `configs/platforms/`
4. 添加 board 配置到 `configs/boards/`
5. 添加 product 配置到 `configs/products/`
6. 添加 profile 配置到 `configs/profiles/`
7. 添加组件到 `components/<name>/`
8. 添加离线和目标板验证脚本
9. 从本地 SDK 导入厂商资产（不提交闭源二进制）

### 项目里程碑路线图

| 版本 | 目标 |
|------|------|
| v0.1 | 骨架和 RV1126B 参考布局 |
| v0.2 | 最小构建引擎（debootstrap, chroot, 包安装, overlay） |
| v0.3 | 组件框架（元数据、依赖检查、kernel/firmware/update 组件） |
| v0.4 | Vendor SDK 桥接（导入模型、Rockchip 检测器、copy mode） |
| v0.5 | 可复现性和打包（lock 文件、checksum、deb mode） |
| v1.0 | 社区就绪发布（CI、schema 验证、通用 ARM64 示例、贡献指南） |

## 关键技术参考

- Rockchip SDK: `atk_dlrv1126b_linux6.1_sdk_release_v1.2.1_20260327`
- Buildroot 输出路径: `output/buildroot/target/usr/lib/`
- GStreamer Rockchip 插件: `libgstrockchipmpp.so`
- ISP 3A 服务: `rkaiq_3A_server`
- NPU 服务: `rknn_server`
- 音频库: `librkaudio.so`, `librockit.so`, `librkdemuxer.so`
- Rockchip update 工具: `/usr/bin/updateEngine`, `/usr/bin/update`

## 相关笔记

- [[rv1126b]] — RV1126B 运动相机项目（BoardRoot 首个参考目标）
- [[rk]] — Rockchip Linux SDK
- [[h618-buildroot]] — H618 完整开发笔记（Buildroot 对比参考）
- [[h618]] — H618 TV Box 定制 Linux 系统（debootstrap 对比参考）
- [[ok1126b-sdk]] — OK1126B SDK 开发笔记
- [[rv1126b-xiaoyu]] — 小宇 RV1126B 项目
- [[BoardRoot-嵌入式Linux厂商适配框架知识库]] — BoardRoot 完整知识库（详细版）
