---
tags:
  - boardroot
  - methodology
  - planning
category: embedded-linux/rockchip
created: 2026-06-09
---

# BoardRoot 嵌入式 Linux 厂商适配框架知识库

## 概述

BoardRoot 是一个**厂商适配框架（vendor adaptation framework）**，构建于 Ubuntu/Debian 的 debootstrap 基础之上。其核心设计目标是利用 Ubuntu 的 APT 生态系统，为嵌入式 Linux 板卡（如 Rockchip RV1126B、Allwinner H616）提供可复现的 rootfs 构建能力。

与 Buildroot 的"每个包单独编写"模式不同，BoardRoot 的核心价值在于：
1. 使用 `debootstrap` 创建标准 Ubuntu/Debian rootfs，依赖 APT 包管理
2. 通过三种厂商适配机制注入芯片厂商的特定修改
3. 声明式 manifest 驱动系统配置（hostname、users、fstab、network）

项目路径：`/home/lijian/project/rv1126b/BoardRoot`
SDK 路径：`/home/lijian/project/rv1126b/rv1126bsportCam/sdk/atk_dlrv1126b_linux6.1_sdk_release_v1.2.1_20260327/`

---

## 关键知识点

### 1. 三种厂商适配机制

BoardRoot 的核心差异化在于三种厂商适配机制，前两种产出 `.deb` 包通过 APT 管理：

| 机制 | source.type | 描述 | 适用场景 |
|------|-------------|------|----------|
| **二进制覆盖** | `sdk-path` / `buildroot-output` | 直接复制 .so/.ko/firmware 到 rootfs | 固件、内核模块、闭源库 |
| **源码补丁** | `apt-source-patch` | 下载 Ubuntu 源码包 → 应用厂商 patch → dpkg-buildpackage 编译为 .deb | GStreamer 插件、MPP、RGA |
| **Buildroot 桥接** | `buildroot-deb` / `buildroot-deb-package` | 将 Buildroot 交叉编译的 .deb 导入为本地 APT 仓库 | 厂商专有库、开发头文件 |

### 2. 七层架构模型

BoardRoot 的配置采用七层抽象模型：

- **Distro** — 发行版选择（Ubuntu Jammy / Debian Bookworm）
- **Platform** — 芯片厂商抽象（Rockchip / Allwinner / Generic）
- **Board** — 硬件板级支持（sportcam / ipcam / orangepi-zero3）
- **Product** — 产品类型（minimal-system / sportcam / ipcamera）
- **Profile** — 构建配置（debug / release）
- **Packages** — APT 包管理（base / network / debug）
- **Components** — 可复用组件（firmware / nginx / docker-ce 等）

### 3. 三阶段构建流水线

```
debootstrap → configure-system → install-components
```

1. **debootstrap** — 创建基础 Ubuntu/Debian rootfs
2. **configure-system** — 配置 hostname、users、fstab、network、APT
3. **install-components** — 按依赖顺序安装所有启用的组件

### 4. 组件生命周期

每个组件遵循 `prepare → install → validate` 生命周期：

- **component.yaml** — v2 元数据声明（name、version、source、install、validate）
- **install.sh** — 安装脚本（调用 `boardroot_parse_component_copy_args` + `boardroot_copy_component_files`）
- **validate.sh** — 验证脚本（检查二进制存在、配置文件完整、服务链接正确）

### 5. 组件依赖解析

使用 Kahn 算法（topological sort）进行拓扑排序，确保依赖组件先于被依赖组件安装：

```yaml
# component.yaml 中声明依赖
runtime:
  requires:
    components:
      - firmware
      - kernel-modules
```

- 依赖图从所有已启用组件的 `runtime.requires.components` 构建
- 检测循环依赖并报错
- `rockchip-mpp` 依赖 `firmware` → firmware 先安装

### 6. Kconfig 配置系统

支持 `menuconfig` TUI 和 `defconfig` 预设：

```
configs/kconfig/Config.in      # Kconfig 定义
configs/defconfigs/             # 预设配置文件
configs/defconfigs/fragments/   # 配置片段（overlay 机制）
```

关键命令：
- `./build.sh menuconfig` — 启动 TUI 配置界面
- `./build.sh saveconfig <name>` — 保存当前配置
- `./build.sh mergeconfig base fragment1 fragment2` — 合并配置片段
- `./build.sh listdefconfig` — 列出所有预设

---

## 技术细节

### Manifest 格式

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

### component.yaml v2 格式

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

### 安全加固要点

| 安全问题 | 修复方案 | 涉及文件 |
|----------|----------|----------|
| Shell injection（用户创建） | 使用 env 变量传递 + heredoc 脚本替代 `bash -c` 单引号插值 | `core/configure-rootfs.sh` |
| Hostname 注入 | RFC 1123 正则验证 + awk 替代 sed | `core/configure-rootfs.sh` |
| 网络配置注入 | CIDR/IP/接口名格式验证 | `core/configure-network.sh` |
| fstab 注入 | fstype 白名单 + mountpoint 绝对路径 + dump/passno 整数验证 | `core/configure-fstab.sh` |
| APT mirror 注入 | http/https URL 格式验证 + 换行符检测 | `core/configure-apt.sh` |
| sed 注入 | `escape_sed()` 函数转义特殊字符 | `core/kconfig-merge.sh` |

### 可靠性机制

- **原子下载** — 使用 `.part` 临时文件 + `flock` 文件锁，下载完成后 mv 到缓存路径
- **并发构建保护** — `flock -n` 独占锁，第二个构建立即失败
- **信号处理** — `trap cleanup_on_exit EXIT`，SIGINT/SIGTERM 清理挂载点和临时文件
- **构建检查点** — `save_checkpoint` / `load_checkpoint` 实现断点续传
- **双重挂载防护** — `mountpoint -q` 检查后再 mount

---

## 代码/配置片段

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

### 组件安装器模板

```bash
#!/usr/bin/env bash
set -euo pipefail
root_dir="$(cd "$(dirname "${BASH_SOURCE[0]}")/../.." && pwd)"
. "$root_dir/core/common.sh"
. "$root_dir/core/component-copy.sh"

boardroot_parse_component_copy_args "components/<name>/component.yaml" "$@"
boardroot_copy_component_files

# 额外安装逻辑（如有）
```

### 组件验证器模板

```bash
#!/usr/bin/env bash
set -euo pipefail
root_dir="$(cd "$(dirname "${BASH_SOURCE[0]}")/../.." && pwd)"
. "$root_dir/core/common.sh"
if [ "$#" -ne 1 ]; then boardroot_die "usage: $0 <rootfs>"; fi
rootfs="$1"

[ -x "$rootfs/usr/sbin/<binary>" ] || boardroot_die "<binary> not found"
[ -f "$rootfs/etc/<config>" ] || boardroot_die "<config> not found"
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

---

## 相关链接

### 项目阶段规划文件

| 文件 | 描述 |
|------|------|
| `/home/lijian/.claude/plans/boardroot-goal.md` | Phase 2 生产就绪目标 |
| `/home/lijian/.claude/plans/boardroot-phase2.md` | Phase 2 详细计划（依赖解析、Fragment、CI/CD、Allwinner） |
| `/home/lijian/.claude/plans/boardroot-phase3.md` | Phase 3 全球开源就绪（下载系统、20+ 通用组件、测试框架） |
| `/home/lijian/.claude/plans/boardroot-phase3-deep.md` | Phase 3 深度工程实现（生产级配置、systemd 服务、focused validation） |
| `/home/lijian/.claude/plans/boardroot-phase4-hardening.md` | Phase 4 工程加固（安全、可靠性、解析器、测试深度、错误恢复） |
| `/home/lijian/.claude/plans/boardroot-12h-v2.md` | 12 小时计划 v2（简化组件、源码补丁、Buildroot 桥接） |
| `/home/lijian/.claude/plans/boardroot-12h-v3.md` | 12 小时计划 v3（补全多媒体组件、Overlay 实际配置、增量构建） |
| `/home/lijian/.claude/plans/boardroot-12h-v4-deb-integration.md` | 12 小时计划 v4（apt-source-patch 真正可用、厂商组件 .deb 化） |
| `/home/lijian/.claude/plans/boardroot-12h-autonomous.md` | 自主执行计划（通用组件实现、多厂商示例、集成测试） |
| `/home/lijian/.claude/plans/agile-hopping-haven.md` | apt-source-patch 端到端验证（deb-src 修复、GStreamer 插件、dev 包） |

### 组件分类

**Rockchip 厂商组件（binary overlay）：**
firmware, kernel-modules, rockchip-mpp, rockchip-rga, rockchip-isp-server, rockchip-npu-server, rockchip-audio, rockchip-iva, rockchip-update-tools, udev-rules, disk-helpers, gstreamer-rockchip

**通用组件（package-list / apt）：**
nginx, ssh-server, python-runtime, ntp-client, dev-tools, docker-ce, wifi-manager, bluetooth-stack, audio-manager, gpio-tools, camera-tools

**源码补丁组件（apt-source-patch）：**
gstreamer-rockchip-src, rockchip-mpp-src, rockchip-rga-src, gstreamer-plugins-base-rga

**Buildroot 桥接组件（buildroot-deb）：**
rockchip-debs, rockchip-rga-dev-deb, rockchip-mpp-dev-deb

**Allwinner 厂商组件：**
allwinner-cedarx

### 关键技术参考

- Rockchip SDK: `atk_dlrv1126b_linux6.1_sdk_release_v1.2.1_20260327`
- Buildroot 输出路径: `output/buildroot/target/usr/lib/`
- GStreamer Rockchip 插件: `libgstrockchipmpp.so`
- ISP 3A 服务: `rkaiq_3A_server`
- NPU 服务: `rknn_server`
- 音频库: `librkaudio.so`, `librockit.so`, `librkdemuxer.so`
