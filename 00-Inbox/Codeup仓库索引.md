---
tags: [codeup, git, repository, index]
created: 2026-06-09
---

# Codeup 仓库索引

> 扫描时间: 2026-06-09
> 平台: https://codeup.aliyun.com

I now have all the information needed. Here is the comprehensive report.

---

## Comprehensive Codeup Repository Scan Report

### Platform Information

**Codeup (codeup.aliyun.com)** is Alibaba Cloud's code hosting platform (similar to GitHub/GitLab). The web interface requires authentication -- it redirected to an Alibaba Cloud login page, so no public repository listing is accessible without credentials.

**User Identity (from .gitconfig)**:
- Name: 李健 (Linux) / li jian (Windows)
- Email: 15588296118@163.com
- Credential helper: store (credentials saved locally)
- Codeup credential provider configured in Windows .gitconfig

**Codeup Organizations/Groups Identified**:
- `687efa92a72fbe627240276f` -- Personal account (hosts `codeup/` group repos)
- `677635f36d55f722de2765fc` -- Another account (hosts `yinghai-ai`)

---

### Repository 1: boke (Personal Technical Blog)

| Field | Value |
|-------|-------|
| **Repository URL** | `https://codeup.aliyun.com/687efa92a72fbe627240276f/codeup/boke.git` |
| **Local Path** | `/home/lijian/project/web/boke` |
| **Current Branch** | `main` |
| **Remote Branches** | `origin/main` |
| **Last Commit** | `bf2aee4 Add project documentation and summary` |
| **Recent Commits** | Auto deploy testing, webhook integration, documentation |
| **Status** | 1 modified file (`.claude/settings.local.json`) |

**Description/Purpose**: A personal technical blog focused on embedded hardware and software topics. Uses Hugo static site generator with the FixIt theme, deployed on Azure (IP: 20.64.150.63) with automated CI/CD via Codeup Webhook + Python Flask.

**Key Technologies**:
- Hugo 0.160.1 (Extended) -- static site generator
- FixIt theme -- Hugo theme with Chinese support
- Nginx -- web server on Azure
- Python Flask -- webhook receiver for auto-deploy
- systemd -- service management
- Microsoft Azure -- hosting

**README Summary**: Documents the blog setup including local preview (`hugo server`), publishing workflow (`git push origin main` triggers auto-deploy), and production URL at http://20.64.150.63.

---

### Repository 2: ch32v208-project (RISC-V Embedded Development)

| Field | Value |
|-------|-------|
| **Repository URL** | `https://codeup.aliyun.com/687efa92a72fbe627240276f/codeup/ch32v208-project.git` |
| **Local Path** | `/home/lijian/project/wch/ch32v208-project` |
| **Current Branch** | `master` |
| **Remote Branches** | `origin/master` |
| **Last Commit** | `2e27813 change README.MD` |
| **Uses Git LFS** | Yes (compiler toolchain archives) |

**Description/Purpose**: Embedded firmware project for the WCH CH32V208 RISC-V microcontroller. Contains libraries移植 (ported) from the official CH32V208 EVT examples, a Makefile-based build system, and the MRS Toolchain compiler (RISC-V Embedded GCC).

**Key Technologies**:
- CH32V208 RISC-V MCU (WCH/QingKe V4B core)
- RISC-V Embedded GCC (MRS_Toolchain_Linux_x64_V210)
- Makefile build system
- Git LFS for large binary files (compiler toolchain)

**README Summary**: Instructions for cloning, pulling LFS files, setting up the RISC-V compiler toolchain (`riscv-none-embed-gcc` or `riscv-wch-elf-gcc`), and building with `make V=1`.

---

### Repository 3: ch32v208-windowAndLinux (Cross-Platform CH32V208 Project)

| Field | Value |
|-------|-------|
| **Repository URL** | `https://codeup.aliyun.com/687efa92a72fbe627240276f/ch32v208-windowAndLinux.git` |
| **Local Path** | `/home/lijian/project/wch/ch32v208-project-windowsAndlinux` |
| **Current Branch** | `useTmosAndcustomUsb` |
| **Remote Branches** | `master`, `useTmosAndcustomUsb`, `userTmos` |
| **Last Commit** | `7585d87 use tmos and custom usb` |
| **Recent Commits** | BLE + USB integration, encoder support, TMOS RTC clock init |

**Description/Purpose**: A variant/fork of the CH32V208 project focused on cross-platform (Windows + Linux) development with Bluetooth Low Energy (BLE) via TMOS (Timer Management Operating System) and custom USB implementation. Note: this repo is in a different Codeup group path (no `/codeup/` subfolder).

**Key Technologies**:
- CH32V208 RISC-V MCU
- TMOS (BLE stack timer management)
- Custom USB implementation
- BLE (Bluetooth Low Energy)

**README Summary**: No README file found. The project appears to be an active development branch focused on TMOS + custom USB functionality.

---

### Repository 4: yinghai-ai (AI E-commerce Content Platform)

| Field | Value |
|-------|-------|
| **Repository URL** | `https://codeup.aliyun.com/677635f36d55f722de2765fc/yinghai-ai.git` |
| **Local Path** | `/home/lijian/project/rv1126b/web/ai/yinghai-ai` |
| **Current Branch** | `lijian_dev` |
| **Remote Branches** | `main`, `cbc_dev`, `lijian_dev` |
| **Last Commit** | `733d58f refactor: API routes use unified database client` |
| **Recent Commits** | Config/dependency updates, deployment docs, database init scripts, login bugfixes |

**Description/Purpose**: "赢海老陈 AIGC 站" -- An AI-driven e-commerce content generation platform. Integrates multiple AI capabilities including AI image generation (product detail pages, product main images, model outfit images, image conversion, image optimization). Built by 扣子编程 CLI (Coze Programming CLI).

**Key Technologies**:
- Next.js 16 (App Router)
- shadcn/ui component library
- pnpm package manager
- TypeScript
- Custom server (server/index.ts)
- Database (SQL backup present)
- Vitest for testing

**README Summary**: Full-stack application created with Coze Programming CLI. Development via `pnpm dev` (port 5000), production build via `pnpm build`. Uses shadcn/ui components as the primary UI foundation. The `LIJIAN_DEV_ANALYSIS.md` contains a comprehensive 10-section project analysis covering architecture, database design, API documentation, and deployment.

---

### Summary Table

| # | Repository | URL | Local Path | Tech Stack | Status |
|---|-----------|-----|------------|------------|--------|
| 1 | boke | `codeup.aliyun.com/.../codeup/boke.git` | `/home/lijian/project/web/boke` | Hugo, Nginx, Flask, Azure | Active (main) |
| 2 | ch32v208-project | `codeup.aliyun.com/.../codeup/ch32v208-project.git` | `/home/lijian/project/wch/ch32v208-project` | CH32V208, RISC-V GCC, Makefile | Stable (master) |
| 3 | ch32v208-windowAndLinux | `codeup.aliyun.com/.../ch32v208-windowAndLinux.git` | `/home/lijian/project/wch/ch32v208-project-windowsAndlinux` | CH32V208, TMOS, BLE, USB | Active dev (useTmosAndcustomUsb) |
| 4 | yinghai-ai | `codeup.aliyun.com/.../yinghai-ai.git` | `/home/lijian/project/rv1126b/web/ai/yinghai-ai` | Next.js 16, shadcn/ui, TypeScript | Active dev (lijian_dev) |

### Additional Notes

- **No Codeup repos found** on the Windows side (`/mnt/c/Users/lijian/`) -- all repos are cloned under WSL2 (`/home/lijian/`).
- **Credential store access** was restricted by security policy. The Windows `.gitconfig` confirms a Codeup credential provider is configured, meaning authentication is set up for pushing/pulling.
- **Two Codeup accounts** are in use: `687efa92a72fbe627240276f` (repos 1-3) and `677635f36d55f722de2765fc` (repo 4).
- The Codeup web interface requires login, so no public repository discovery was possible.

---

## 导入建议

对于每个仓库，建议：
- 有 README 的项目 → 提取为独立知识笔记
- 纯代码项目 → 记录技术栈和架构
- 学习项目 → 提取学习收获
