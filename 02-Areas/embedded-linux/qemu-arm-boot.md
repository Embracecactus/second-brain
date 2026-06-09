---
tags: [qemu, arm, docker]
category: embedded-linux
created: 2026-06-09
---
Obsidian note generated at:

`/home/lijian/project/Obsidian/Dockerfile Ubuntu 20.04 ARM QEMU 构建笔记.md`

The note covers:

- **YAML frontmatter** with title, date, tags (docker/ubuntu/qemu/arm/u-boot/embedded), category, and source path
- **Overview** summarizing the Dockerfile's purpose: Ubuntu 20.04 base image for embedded ARM development with QEMU
- **Key points** covering the base image, non-interactive mode, tool installation, cache cleanup, QEMU ARM emulation, and volume mounting
- **Detailed analysis** of each Dockerfile instruction (FROM, ENV, RUN, WORKDIR, COPY, QEMU command) with parameter tables
- **Usage instructions** for building and running the container
- **Architecture diagram** showing the host -> Docker -> QEMU -> U-Boot stack
- **Caveats** about U-Boot image path, serial console naming (`ttyO0`), QEMU version requirements, performance limitations, and the commented-out CMD instruction

## 相关笔记

- [[h3]] — Allwinner H3 系统构建
- [[h618-buildroot]] — H618 完整开发笔记
- [[orangepi]] — OrangePi PC 嵌入式 Linux 开发
- [[docker-alicloud]] — Docker 镜像推送到阿里云 ACR
