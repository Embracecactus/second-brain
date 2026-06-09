---
tags: [qemu, arm, docker, embedded-linux, u-boot, ubuntu]
category: embedded-linux
created: 2026-06-09
updated: 2026-06-09
status: active
---

# QEMU ARM U-Boot 模拟启动 — Docker 构建环境

## 项目/工具概述

本项目通过 Docker 容器封装 Ubuntu 20.04 基础环境，结合 QEMU 系统级模拟器实现 ARM 架构的 U-Boot 固件在 x86 主机上的无硬件启动与调试。该方案适用于嵌入式 Linux 开发早期阶段，在没有物理开发板的情况下完成 U-Boot 引导流程验证、内核启动测试以及根文件系统调试。使用 Docker 封装的好处在于环境可复现、依赖隔离，且可以快速在不同主机间迁移开发环境。

## 技术栈 / 关键特性

| 组件 | 版本/说明 |
|------|-----------|
| Docker | 容器化运行环境 |
| Ubuntu 20.04 | 基础镜像 (LTS) |
| QEMU (`qemu-system-arm`) | ARM 架构系统级模拟器 |
| U-Boot | 嵌入式引导加载器 (Bootloader) |
| `virt-6.2` Machine | QEMU 虚拟 ARM 机器类型 |
| `DEBIAN_FRONTEND=noninteractive` | 非交互式包安装模式 |

核心特性：
- 无需物理 ARM 开发板即可验证 U-Boot 启动流程
- Docker 容器环境保证构建可复现性
- `-nographic` 模式通过串口控制台进行交互
- 支持 volume 挂载，方便在宿主机与容器间共享文件

## 架构与设计

整体架构为层级式封装，宿主机通过 Docker 运行 QEMU 模拟器，QEMU 内部运行 ARM 虚拟机并加载 U-Boot 固件：

```
┌─────────────────────────────────────────────┐
│              Host Machine (x86)             │
│  ┌────────────────────────────────────────┐  │
│  │           Docker Container             │  │
│  │  ┌──────────────────────────────────┐  │  │
│  │  │      QEMU (qemu-system-arm)      │  │  │
│  │  │    Machine: virt-6.2 (ARM)       │  │  │
│  │  │  ┌────────────────────────────┐  │  │  │
│  │  │  │    U-Boot (u-boot.img)     │  │  │  │
│  │  │  │  console=ttyO0 (serial)    │  │  │  │
│  │  │  └────────────────────────────┘  │  │  │
│  │  └──────────────────────────────────┘  │  │
│  │         /app ← volume mount            │  │
│  └────────────────────────────────────────┘  │
│       /home/lijian  ← 宿主机共享目录          │
└─────────────────────────────────────────────┘
```

数据流：宿主机目录 `/home/lijian` 通过 Docker volume 挂载到容器 `/app`，QEMU 从容器内路径 `output/images/u-boot.img` 加载 U-Boot 镜像，串口输出通过 `-nographic` 重定向到终端。

## 核心知识点

### 1. Docker 基础镜像选择

使用 `ubuntu:20.04` 作为基础镜像，该版本为 LTS 长期支持版本，软件包生态稳定。设置 `DEBIAN_FRONTEND=noninteractive` 环境变量可避免 `apt-get install` 过程中弹出交互式配置对话框（如时区选择、键盘布局等），这对自动化构建至关重要。

### 2. 镜像层优化

Dockerfile 中使用了标准的镜像层优化策略：
- `apt-get update` 与 `apt-get install` 合并为单条 `RUN` 指令，避免缓存过期的包索引
- `apt-get clean` 清理 APT 缓存
- `rm -rf /var/lib/apt/lists/*` 删除包列表以减小镜像体积
- 安装的工具（`curl`、`wget`、`vim`）均为嵌入式开发常用调试工具

### 3. QEMU ARM 系统模拟

`qemu-system-arm` 是 QEMU 的 ARM 全系统模拟器，关键参数：
- `-M virt-6.2`：指定虚拟机类型为 `virt`（QEMU 通用 ARM 虚拟平台），版本 6.2。`virt` 是 QEMU 推荐的 ARM 虚拟机类型，支持 VirtIO 设备、PCIe 等现代特性
- `-kernel output/images/u-boot.img`：指定内核镜像路径，这里加载的是 U-Boot 编译产物
- `-nographic`：禁用图形输出，将串口重定向到标准输入/输出（stdin/stdout）
- `-append "console=ttyO0"`：传递内核命令行参数，指定串口控制台设备为 `ttyO0`

### 4. 串口控制台 `ttyO0`

`ttyO0` 中的字母 "O"（大写）代表 OMAP 系列 SoC 的串口命名约定。这与标准 Linux 中的 `ttyS0` 不同，常见于 TI OMAP、AM335x 等 ARM 平台。在 QEMU `virt` 机器中使用此名称是因为 U-Boot 或内核的设备树中配置了对应的串口节点。

### 5. U-Boot 镜像路径

Dockerfile 中 `qemu-system-arm` 使用的 `-kernel` 路径为 `output/images/u-boot.img`，这是一个相对路径，实际指向 Buildroot 或 Yocto 等构建系统的输出目录。在容器中运行时需要确保该路径存在，或者通过 volume 挂载将宿主机上的编译产物映射到正确位置。

## 关键代码/配置片段

### Dockerfile 完整内容

```dockerfile
# 构建命令 sudo docker build -t my-ubuntu-20_04:latest .
# 启动 sudo docker run -v /home/lijian:/app -it my-ubuntu-20_04:latest

# 使用 Ubuntu 20.04 基础镜像
FROM ubuntu:20.04

# 设置环境变量，避免安装过程中出现交互式配置提示
ENV DEBIAN_FRONTEND=noninteractive

# 更新包列表并安装常用工具
RUN apt-get update && \
    apt-get install -y \
    curl \
    wget \
    vim \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# 设置工作目录
WORKDIR /app

# 复制当前目录下的所有文件到工作目录（已注释）
# COPY . /app

# 暴露端口（已注释）
# EXPOSE 80

# 定义启动命令（已注释）
# CMD ["bash"]

# QEMU ARM U-Boot 启动命令
qemu-system-arm \
    -M virt-6.2 \
    -kernel output/images/u-boot.img \
    -nographic \
    -append "console=ttyO0"
```

### 构建与运行命令

```bash
# 构建 Docker 镜像
sudo docker build -t my-ubuntu-20_04:latest /home/lijian/Dockerfile/

# 运行容器（挂载宿主机目录到 /app）
sudo docker run -v /home/lijian:/app -it my-ubuntu-20_04:latest

# 在容器内执行 QEMU 启动（需确保 u-boot.img 存在）
qemu-system-arm \
    -M virt-6.2 \
    -kernel output/images/u-boot.img \
    -nographic \
    -append "console=ttyO0"
```

### QEMU 常用调试参数补充

```bash
# 添加 GDB 调试支持（在 1234 端口等待 GDB 连接）
qemu-system-arm -M virt-6.2 \
    -kernel output/images/u-boot.img \
    -nographic \
    -append "console=ttyO0" \
    -s -S

# 指定内存大小
qemu-system-arm -M virt-6.2 \
    -m 256M \
    -kernel output/images/u-boot.img \
    -nographic \
    -append "console=ttyO0"

# 添加网络支持
qemu-system-arm -M virt-6.2 \
    -kernel output/images/u-boot.img \
    -nographic \
    -append "console=ttyO0" \
    -netdev user,id=net0 \
    -device virtio-net-device,netdev=net0
```

## 使用方法 / 构建步骤

### 前置条件

1. 宿主机已安装 Docker（`sudo apt install docker.io`）
2. U-Boot 源码已通过 Buildroot 或其他构建系统编译，产出 `u-boot.img`
3. 将编译产物放置或挂载到容器内 `output/images/` 路径下

### 完整操作流程

```bash
# Step 1: 进入 Dockerfile 所在目录
cd /home/lijian/Dockerfile/

# Step 2: 构建 Docker 镜像
sudo docker build -t my-ubuntu-20_04:latest .

# Step 3: 启动容器并挂载 U-Boot 编译产物目录
sudo docker run -v /home/lijian:/app -it my-ubuntu-20_04:latest

# Step 4: 在容器内确认 U-Boot 镜像存在
ls -la /app/output/images/u-boot.img

# Step 5: 执行 QEMU 启动
qemu-system-arm \
    -M virt-6.2 \
    -kernel /app/output/images/u-boot.img \
    -nographic \
    -append "console=ttyO0"

# Step 6: 退出 QEMU — 在串口控制台中按 Ctrl+A 然后按 X
```

### 注意事项

- **U-Boot 镜像路径**：Dockerfile 中使用的是相对路径 `output/images/u-boot.img`，在容器中运行时需确保 volume 挂载后该路径可达，或改为绝对路径
- **串口设备名**：`ttyO0`（大写字母 O）是 OMAP 系列命名，部分平台可能需要改为 `ttyS0` 或 `ttyAMA0`（ARM PL011 UART）
- **QEMU 版本**：`-M virt-6.2` 要求 QEMU 版本 >= 6.2，可通过 `qemu-system-arm --version` 检查
- **性能限制**：QEMU 全系统模拟性能远低于原生 ARM 执行，仅适用于功能验证，不适合性能测试
- **CMD 注释**：Dockerfile 中 `CMD ["bash"]` 被注释掉，`qemu-system-arm` 命令直接写在文件末尾（非标准写法），建议将其整合到 `CMD` 或 `ENTRYPOINT` 指令中

## 相关笔记

- [[h3]] — Allwinner H3 SoC 系统构建与 U-Boot 配置
- [[h618-buildroot]] — H618 Buildroot 完整开发笔记
- [[orangepi]] — OrangePi PC 嵌入式 Linux 开发
- [[docker-alicloud]] — Docker 镜像推送到阿里云 ACR
- [[rv1126-notes]] — Rockchip RV1126 嵌入式开发笔记
