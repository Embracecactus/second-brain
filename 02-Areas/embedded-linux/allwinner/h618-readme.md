---
tags: [allwinner, h618, docker, retroarch]
category: embedded-linux/allwinner
created: 2026-06-09
---
The Obsidian note has been generated at `/mnt/c/Users/lijian/workspace/h618-readme/h618-readme.md`. It contains comprehensive documentation covering:

- YAML frontmatter with relevant tags and category
- Project overview of the Allwinner H618 AI TV Box BSP
- Full technology stack listing
- Architecture details including build pipeline, boot flow, device tree strategy, and rootfs construction
- Key design decisions around patch management, network configuration, and Docker support
- Notable code snippets from pack.sh, boot.cmd, genimage.cfg, and sd2emmc.sh
- Six key learnings from the project (chroot systemd limitations, Ubuntu APT format changes, network naming, desktop suspend issues, VPU codec support, offline Docker deployment)
- Complete build/run instructions for all stages
- Links to related concepts and external references

## 相关笔记

- [[h3]] — Allwinner H3 系统构建全栈笔记
- [[h5]] — Allwinner H5 Crust Firmware 项目
- [[h618]] — H618 TV Box 定制 Linux 系统
- [[h618-buildroot]] — H618 完整开发笔记
- [[orangepi]] — OrangePi PC 嵌入式 Linux 开发
- [[docker-alicloud]] — Docker 镜像推送到阿里云 ACR
