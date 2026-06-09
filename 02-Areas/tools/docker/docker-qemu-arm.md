---
title: Dockerfile Ubuntu 20.04 ARM QEMU 构建笔记
date: 2026-06-09
tags:
  - docker
  - ubuntu
  - qemu
  - arm
  - u-boot
  - embedded
category: 嵌入式开发/容器化
source: /home/lijian/Dockerfile/Dockerfile
---

# Dockerfile Ubuntu 20.04 ARM QEMU 构建笔记

## 概述

该 Dockerfile 用于构建一个基于 **Ubuntu 20.04** 的 Docker 开发镜像，主要面向嵌入式 ARM 开发场景。镜像内置常用工具（curl、wget、vim），并通过 **QEMU** 模拟器运行 ARM 架构的 U-Boot 引导加载程序，实现无需实体硬件即可进行 ARM 嵌入式开发与调试。

---

## 关键要点

- **基础镜像**: 使用官方 `ubuntu:20.04`，确保开发环境一致性和可复现性
- **非交互模式**: 设置 `DEBIAN_FRONTEND=noninteractive` 阻止安装过程中的交互式提示，适配自动化构建
- **工具集**: 安装 `curl`、`wget`、`vim` 等常用开发/调试工具
- **镜像精简**: 安装后执行 `apt-get clean` 并删除缓存目录，减小镜像体积
- **QEMU ARM 模拟**: 使用 `qemu-system-arm` 模拟 `virt-6.2` 机器类型，加载 U-Boot 镜像
- **Volume 挂载**: 通过 `-v` 参数将宿主机目录映射到容器内 `/app`，实现文件共享

---

## Dockerfile 详细解析

### 1. 基础镜像

```dockerfile
FROM ubuntu:20.04
```

采用 Ubuntu 20.04 LTS 作为基础镜像，提供长期支持和稳定的软件包生态。

### 2. 环境变量设置

```dockerfile
ENV DEBIAN_FRONTEND=noninteractive
```

禁用交互式前端，确保 `apt-get install` 在构建过程中不会弹出配置对话框（如时区选择、服务重启确认等），这是 CI/CD 和 Docker 自动化构建的最佳实践。

### 3. 安装依赖并清理缓存

```dockerfile
RUN apt-get update && \
    apt-get install -y \
    curl \
    wget \
    vim \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

将 `apt-get update` 和 `apt-get install` 合并到同一条 `RUN` 指令中，利用 Docker 层缓存机制，同时在安装完成后清理包管理器缓存以减小镜像体积。

### 4. 工作目录

```dockerfile
WORKDIR /app
```

设置容器内默认工作目录为 `/app`，后续命令均在此目录下执行。

### 5. 文件复制（已注释）

```dockerfile
# COPY . /app
```

该指令已被注释。当前方案使用运行时 Volume 挂载（`-v /home/lijian:/app`）替代构建时复制，便于实时同步宿主机文件。

### 6. QEMU ARM 启动命令

```bash
qemu-system-arm \
    -M virt-6.2 \
    -kernel output/images/u-boot.img \
    -nographic \
    -append "console=ttyO0"
```

| 参数 | 说明 |
|------|------|
| `qemu-system-arm` | QEMU 全系统 ARM 模拟器 |
| `-M virt-6.2` | 指定模拟的机器类型为 QEMU virt 平台（版本 6.2），这是 QEMU 推荐的通用 ARM 虚拟化平台 |
| `-kernel output/images/u-boot.img` | 加载 U-Boot 引导镜像文件 |
| `-nographic` | 禁用图形输出，所有串口输出重定向到终端，适合无 GUI 环境 |
| `-append "console=ttyO0"` | 向内核传递启动参数，指定串口控制台设备为 `ttyO0`（OMAP 系列常见串口命名） |

---

## 使用方法

### 构建镜像

```bash
sudo docker build -t my-ubuntu-20_04:latest /home/lijian/Dockerfile/
```

### 启动容器

```bash
sudo docker run -v /home/lijian:/app -it my-ubuntu-20_04:latest
```

| 参数 | 说明 |
|------|------|
| `-v /home/lijian:/app` | 将宿主机 `/home/lijian` 挂载到容器 `/app`，双向文件共享 |
| `-it` | 分配伪终端并保持 STDIN 打开，支持交互式操作 |

---

## 架构说明

```
宿主机 (x86_64)
  |
  +-- Docker 容器 (Ubuntu 20.04)
        |
        +-- QEMU ARM System Emulator
              |
              +-- virt-6.2 Machine
                    |
                    +-- U-Boot (ARM bootloader)
                          |
                          +-- console: ttyO0
```

---

## 注意事项

1. **U-Boot 镜像路径**: `output/images/u-boot.img` 需要在容器内 `/app` 目录下存在，即宿主机 `/home/lijian` 中应包含该文件（通常由 Buildroot 或 Yocto 构建生成）
2. **串口控制台**: `ttyO0` 是 OMAP 系列处理器的串口设备名，若目标平台不同可能需要调整为 `ttyAMA0`（ARM PrimeCell）或 `ttyS0`（标准串口）
3. **QEMU 版本**: `-M virt-6.2` 要求 QEMU 版本不低于 6.2，需确认容器内 QEMU 版本兼容性
4. **性能考量**: QEMU 全系统模拟性能有限，适合开发调试，不适合性能测试
5. **CMD 指令被注释**: 最终启动命令（QEMU）位于文件末尾但未封装为 `CMD` 或 `ENTRYPOINT`，实际使用时需要手动执行或补充 Dockerfile 指令
