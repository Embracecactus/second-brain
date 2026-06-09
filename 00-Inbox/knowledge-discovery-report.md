---
tags: [discovery, import-plan, meta]
created: 2026-06-09
scan_date: 2026-06-09
---

# 知识发现报告

> 扫描范围：Linux `/home/lijian/`（排除 project）+ Windows `/mnt/c/Users/lijian/`
> 目标 Vault：`/home/lijian/project/Obsidian/vault/`
> Vault 结构：PARA（00-Inbox, 02-Areas）

---

## Linux 发现

### 高价值 — 确认导入

| 路径 | 类别 | 内容 | 目标位置 |
|------|------|------|----------|
| `~/.claude/skills/zhuzhuxu-writer/` | 写作知识 | 竹竹叙写作风格词库、风格范例、模板 | `02-Areas/content/zhuzhuxu-writing-style.md` |
| `~/.claude/plans/` (20+ 计划) | 开发方法论 | boardroot/rv1126b 嵌入式 Linux 开发分阶段规划 | `02-Areas/embedded-linux/rockchip/boardroot-methodology.md` |
| `~/.agents/skills/html-ppt/` | 工具参考 | 30+ 演示文稿主题（赛博朋克、蒸汽波等）和动画模板 | `02-Areas/tools/html-ppt-themes.md` |

### 中等价值 — 按需导入

| 路径 | 类别 | 内容 | 目标位置 |
|------|------|------|----------|
| `~/.hermes/config.yaml` | AI 工具链 | AI 工作流配置（模型路由、12+ 人格模式、语音/记忆设置） | `02-Areas/tools/ai-workflow-config.md` |
| `~/.claude/skills/douyin-crawler/SKILL.md` | 内容创作 | 抖音爬虫工作流 | `02-Areas/content/douyin/douyin-crawler-workflow.md` |
| `/home/lijian/Dockerfile/Dockerfile` | 嵌入式参考 | QEMU ARM 启动命令模式 | `02-Areas/embedded-linux/qemu-arm-boot-pattern.md` |
| `/home/lijian/kitware-archive.sh` | 工具参考 | Ubuntu CMake APT 源配置 | `02-Areas/tools/cmake-apt-setup.md` |

### 不导入

| 路径 | 原因 |
|------|------|
| `~/.bashrc`, `~/.profile`, `~/.gitconfig` | 标准配置，无独特知识 |
| `~/.ssh/`, `~/.gnupg/`, `~/httppassword`, `~/.git-credentials` | 凭证/安全文件 |
| `~/bin/repo` | 二进制工具 |
| `~/.hermes/hermes-agent/` | 第三方开源项目源码 |
| `~/.claude-mem/`, `~/.claude/plans/`（运行时数据） | 工具运行时数据 |
| `~/.lingma/`, `~/.copilot/`, `~/.vscode-server/` | IDE 插件安装目录 |
| `~/.local/bin/`（90+ 工具） | 已安装工具的二进制文件 |
| `~/.config/`, `~/.docker/`, `~/.keras/`, `~/.paddleocr/` | 标准工具配置 |
| `~/.zai/`（36 个日志） | 运行时日志 |
| `~/.chelper/config.yaml`, `~/.f2/app.yaml` | 含明文 API Key / Cookie |
| `~/20250328_1953_test.img` | 二进制磁盘镜像 |
| `~/Work/zephyrproject/` | 空目录 |
| `~/go/` | Go 标准工作区 |

---

## Windows 发现

### 高价值 — 确认导入

| 路径 | 类别 | 内容 | 目标位置 |
|------|------|------|----------|
| `/mnt/c/Users/lijian/OneDrive/learn/` | 学习笔记 | **结构化嵌入式学习笔记库**：arm/h618（buildroot 全流程、RetroPie）、arm/rk3528、arm/rv1126、esp（ESP-IDF v5.0）、nxp/nxp（Zephyr RTOS）、nxp/nrf（nRF5340 应用/网络样本）、docker（阿里云镜像推送）、orangepi | 分散导入至各 Area 子目录 |
| `/mnt/c/Users/lijian/Documents/` (XMind 思维导图) | 学习笔记 | FPGA 点灯、Linux 系统、编译生成可执行文件、预处理、网页控制 LED/RGB 灯、CS 发射器功能 | `02-Areas/learning/xmind-notes.md` |
| `/mnt/c/Users/lijian/Documents/OK1126B-S...R1/` | 参考文档 | 飞凌 OK1126B Linux 6.1 完整 SDK 文档 | `02-Areas/embedded-linux/rockchip/ok1126b-reference.md` |
| `/mnt/c/Users/lijian/Documents/imx6ull-document/` | 参考文档 | 正点原子 IMX6ULL 完整文档集 | `02-Areas/embedded-linux/nxp/imx6ull-reference.md` |
| `/mnt/c/Users/lijian/Documents/esp32s3-st25dv-2/` | 项目 | ESP32-S3 + ST25DV NFC 标签项目 | `02-Areas/mcu/esp32/esp32s3-nfc-project.md` |
| `/mnt/c/Users/lijian/Documents/小宇rv1126b/` | 项目 | RV1126B 开发材料 | `02-Areas/embedded-linux/rockchip/rv1126b-dev-notes.md` |
| `/mnt/c/Users/lijian/Desktop/readme/readme` | 参考文档 | 硬件参考项目含配置文件 | `02-Areas/hardware-design/hardware-config-reference.md` |
| `/mnt/c/Users/lijian/OneDrive/Documents/` | 学术 | 研究生论文（SAR/RF、无线工业通信）、大学课程材料 | `02-Areas/career/academic-history.md` |

### 中等价值 — 按需导入

| 路径 | 类别 | 内容 | 目标位置 |
|------|------|------|----------|
| `/mnt/c/Users/lijian/Documents/LVGL/` | 项目 | LVGL 嵌入式 GUI 项目 | `02-Areas/mcu/common/lvgl-project-ref.md` |
| `/mnt/c/Users/lijian/Documents/MYZR-RV1126/` | 参考文档 | 明远智睿 RV1126 板级文档 | `02-Areas/embedded-linux/rockchip/myzr-rv1126-ref.md` |
| `/mnt/c/Users/lijian/Documents/resistorOcr2,3/` | 项目 | 电阻 OCR 识别（ML/CV） | `02-Areas/ai-ml/resistor-ocr.md` |
| `/mnt/c/Users/lijian/Documents/temperature/` | 项目 | 温度测量传感器项目 | `02-Areas/mcu/common/temperature-sensor.md` |
| `/mnt/c/Users/lijian/Documents/Source Insight 4.0/` | 工具参考 | 书签和代码片段 | `02-Areas/tools/source-insight-snippets.md` |
| `/mnt/c/Users/lijian/Desktop/orangepi官方dtb/` | 参考文档 | 官方 OrangePi H3 DTB 文件 | `02-Areas/embedded-linux/allwinner/h3-dtb-reference.md` |
| `/mnt/c/Users/lijian/Desktop/pzx_orangepi_dtb/` | 参考文档 | 自定义 OrangePi DTB 修改 | `02-Areas/embedded-linux/allwinner/h3-dtb-custom.md` |
| `/mnt/c/Users/lijian/Desktop/ImageToEpd.../` | 工具参考 | 电子墨水屏图片转换软件 | `02-Areas/tools/epaper-image-converter.md` |
| `/mnt/c/Users/lijian/Desktop/FRDM-MCXA346-DESIGN-FILES/` | 参考文档 | NXP FRDM-MCXA346 开发板设计文件 | `02-Areas/mcu/nxp/frdm-mcxa346-ref.md` |
| `/mnt/c/Users/lijian/Desktop/BPI_M4B_V10-DXF/` | 参考文档 | 香蕉派 M4 机械图纸 | `02-Areas/hardware-design/bpi-m4-mechanical.md` |
| `/mnt/c/Users/lijian/Documents/COMSOL/Batch/` | 项目 | COMSOL 多物理场仿真 | `02-Areas/hardware-design/comsol-simulation.md` |
| `/mnt/c/Users/lijian/DevEcoStudioProjects/` | 学习 | HarmonyOS 应用开发（3 个项目） | `02-Areas/app/harmonyos-dev-notes.md` |
| `/mnt/c/Users/lijian/.agents/skills/` | 工具参考 | 相机诊断和计算机视觉 Agent 技能 | `02-Areas/camera-isp/camera-diag-skills.md` |

### 不导入

| 路径 | 原因 |
|------|------|
| Desktop 快捷方式、WeChat 数据 | 非知识内容 |
| `~/.claude/`, `~/.codex/`, `~/.config/` | AI 工具运行时配置 |
| `~/esp/` (ESP-IDF SDK) | SDK 安装目录，学习笔记已在 OneDrive/learn |
| `~/STM32Cube/`, `~/SquareLine/` | IDE 工作区数据 |
| `~/TIK/` | 软件工具项目 |
| `~/codebuddy/` | AI 编码助手会话数据 |
| OneDrive/Pictures | 图片文件 |
| OneDrive/附件 | 未扫描，待人工确认 |
| OneDrive/lijian/ | 大学时代文档，低优先级 |

---

## 导入计划

### 第一批：高优先级（建议立即执行）

| 序号 | 源路径 | 目标笔记 | 目标 Vault 路径 | 操作说明 |
|------|--------|----------|-----------------|----------|
| 1 | `OneDrive/learn/arm/h618/` | H618 buildroot 全流程笔记 | `02-Areas/embedded-linux/allwinner/h618-buildroot-guide.md` | 提取 README、uboot/kernel/rootfs 文档、补丁说明 |
| 2 | `OneDrive/learn/arm/rk3528/` | RK3528 系统笔记 | `02-Areas/embedded-linux/rockchip/rk3528-notes.md` | 转换 Markdown 笔记 |
| 3 | `OneDrive/learn/arm/rv1126/` | RV1126 笔记 | `02-Areas/embedded-linux/rockchip/rv1126-notes.md` | 转换 Markdown 笔记 |
| 4 | `OneDrive/learn/esp/` | ESP-IDF v5.0 学习笔记 | `02-Areas/mcu/esp32/esp-idf-v5-guide.md` | 提取 README 和关键截图说明 |
| 5 | `OneDrive/learn/nxp/nxp/` | NXP Zephyr RTOS 笔记 | `02-Areas/mcu/nxp/zephyr-nxp-notes.md` | 提取 README 和截图说明 |
| 6 | `OneDrive/learn/nxp/nrf/` | nRF5340 Zephyr 笔记 | `02-Areas/mcu/nrf/zephyr-nrf-notes.md` | 提取 README、app/net 样本说明 |
| 7 | `OneDrive/learn/docker/` | Docker 阿里云镜像推送笔记 | `02-Areas/tools/docker-alicloud-deploy.md` | 提取 README |
| 8 | `Documents/小宇rv1126b/` | RV1126B 开发材料 | `02-Areas/embedded-linux/rockchip/rv1126b-dev-notes.md` | 整理开发笔记 |
| 9 | `Documents/esp32s3-st25dv-2/` | ESP32-S3 NFC 项目 | `02-Areas/mcu/esp32/esp32s3-nfc-project.md` | 提取项目文档和代码说明 |
| 10 | `Documents/OK1126B...R1/` | OK1126B SDK 文档索引 | `02-Areas/embedded-linux/rockchip/ok1126b-sdk-index.md` | 创建文档索引（不复制原始文件） |
| 11 | `Documents/imx6ull-document/` | IMX6ULL 文档索引 | `02-Areas/embedded-linux/nxp/imx6ull-doc-index.md` | 创建文档索引 |
| 12 | `OneDrive/Documents/` | 研生论文和课程材料索引 | `02-Areas/career/academic-papers.md` | 创建论文清单和摘要 |

### 第二批：中优先级（按需处理）

| 序号 | 源路径 | 目标笔记 | 目标 Vault 路径 | 操作说明 |
|------|--------|----------|-----------------|----------|
| 13 | `Documents/` (XMind 文件) | 思维导图内容汇总 | `02-Areas/learning/xmind-notes-summary.md` | 导出 XMind 为 Markdown 后导入 |
| 14 | `Documents/LVGL/` | LVGL 项目参考 | `02-Areas/mcu/common/lvgl-project-ref.md` | 提取代码结构和关键实现 |
| 15 | `Documents/MYZR-RV1126/` | 明远智睿 RV1126 参考 | `02-Areas/embedded-linux/rockchip/myzr-rv1126-ref.md` | 提取板级文档摘要 |
| 16 | `Documents/resistorOcr2,3/` | 电阻 OCR 项目 | `02-Areas/ai-ml/resistor-ocr.md` | 提取模型和算法说明 |
| 17 | `Documents/temperature/` | 温度传感器项目 | `02-Areas/mcu/common/temperature-sensor.md` | 提取电路和代码说明 |
| 18 | `Documents/COMSOL/Batch/` | COMSOL 仿真 | `02-Areas/hardware-design/comsol-simulation.md` | 提取仿真参数和结果 |
| 19 | `~/.claude/plans/` | boardroot 开发方法论 | `02-Areas/embedded-linux/rockchip/boardroot-methodology.md` | 整理分阶段规划为方法论文档 |
| 20 | `~/.claude/skills/zhuzhuxu-writer/` | 竹竹叙写作风格 | `02-Areas/content/zhuzhuxu-writing-style.md` | 导入词库和风格范例 |
| 21 | `~/.agents/skills/html-ppt/` | 演示主题库 | `02-Areas/tools/html-ppt-themes.md` | 整理主题和模板清单 |
| 22 | `~/.claude/skills/douyin-crawler/` | 抖音爬虫工作流 | `02-Areas/content/douyin/douyin-crawler-workflow.md` | 导入 SKILL.md |
| 23 | `Desktop/orangepi官方dtb/` | H3 官方 DTB | `02-Areas/embedded-linux/allwinner/h3-dtb-reference.md` | 提取 DTS 内容 |
| 24 | `Desktop/pzx_orangepi_dtb/` | H3 自定义 DTB | `02-Areas/embedded-linux/allwinner/h3-dtb-custom.md` | 提取 DTS 内容 |
| 25 | `Desktop/readme/readme` | 硬件配置参考 | `02-Areas/hardware-design/hardware-config-reference.md` | 提取配置文件 |
| 26 | `Desktop/FRDM-MCXA346-DESIGN-FILES/` | NXP 开发板参考 | `02-Areas/mcu/nxp/frdm-mcxa346-ref.md` | 提取设计文件说明 |
| 27 | `DevEcoStudioProjects/` | HarmonyOS 开发笔记 | `02-Areas/app/harmonyos-dev-notes.md` | 整理 3 个项目经验 |

### 第三批：低优先级（可选）

| 序号 | 源路径 | 目标笔记 | 目标 Vault 路径 |
|------|--------|----------|-----------------|
| 28 | `Documents/Source Insight 4.0/` | 代码片段 | `02-Areas/tools/source-insight-snippets.md` |
| 29 | `~/.hermes/config.yaml` | AI 工作流配置 | `02-Areas/tools/ai-workflow-config.md` |
| 30 | `~/Dockerfile/Dockerfile` | QEMU ARM 启动模式 | `02-Areas/embedded-linux/qemu-arm-boot-pattern.md` |
| 31 | `~/kitware-archive.sh` | CMake APT 源 | `02-Areas/tools/cmake-apt-setup.md` |
| 32 | `Desktop/ImageToEpd.../` | 电子墨水屏工具 | `02-Areas/tools/epaper-image-converter.md` |
| 33 | `Desktop/BPI_M4B_V10-DXF/` | 香蕉派机械图纸 | `02-Areas/hardware-design/bpi-m4-mechanical.md` |
| 34 | `.agents/skills/` (Windows) | 相机诊断技能 | `02-Areas/camera-isp/camera-diag-skills.md` |

---

## 新增目录建议

当前 Vault 已有以下 Area 子目录：

```
02-Areas/
├── ai-ml/
├── app/          (android, feishu, flutter, qt, uniapp, wechat)
├── camera-isp/   (genicam, sensor-driver)
├── career/
├── cnc-cam/
├── content/      (blog, douyin, xiaohongshu)
├── embedded-linux/ (allwinner, rockchip)
├── hardware-design/ (antenna, fpga, pcb, power)
├── iot/          (tuya)
├── learning/
├── mcu/          (common, esp32, nrf, stm32, wch-riscv)
├── multimedia/   (ffmpeg)
└── tools/        (ai-eda, build-systems, fusion360, hugo, vuepress)
```

为容纳新发现内容，建议新增：

| 新目录 | 用途 |
|--------|------|
| `02-Areas/embedded-linux/nxp/` | IMX6ULL、Zephyr NXP 相关笔记 |
| `02-Areas/mcu/nxp/` | NXP MCU 开发（FRDM-MCXA346 等） |
| `02-Areas/app/harmonyos/` | HarmonyOS 应用开发 |
| `02-Areas/academic/` | 研究生论文和学术材料（或归入 career） |

---

## 安全提醒

以下文件包含明文凭证，**不应导入 Vault**：

| 文件 | 内容 | 建议 |
|------|------|------|
| `~/httppassword` | 明文 HTTP 密码 | 删除或加密存储 |
| `~/.chelper/config.yaml` | GLM API Key | 移至环境变量 |
| `~/.f2/app.yaml` | 抖音 Session Cookie | 移至加密存储 |
| `~/.hermes/.env`（17KB） | 可能含多个 API Key | 审查并移至加密存储 |
| `~/.git-credentials` | Git 凭证 | 已由系统管理，无需操作 |

---

## 统计

| 指标 | 数量 |
|------|------|
| Linux 扫描路径 | 25 |
| Windows 扫描路径 | 30+ |
| 总发现项 | 55+ |
| 确认导入（高优先级） | 12 |
| 按需导入（中优先级） | 15 |
| 可选导入（低优先级） | 7 |
| 不导入 | 21+ |
| 安全风险项 | 5 |
| Vault 已有笔记 | 40 |
| 建议新增目录 | 4 |
| **导入后预计 Vault 笔记总数** | **~74** |

---

## 导入优先级路线图

```
Week 1: 高优先级 12 项
├── OneDrive/learn/* (7 项嵌入式学习笔记)
├── Documents/小宇rv1126b + esp32s3-st25dv (2 项活跃项目)
├── Documents/OK1126B + imx6ull (2 项参考文档索引)
└── OneDrive/Documents (1 项学术论文索引)

Week 2: 中优先级 15 项
├── Documents/ XMind + LVGL + 其他项目 (8 项)
├── ~/.claude/ plans + skills (4 项)
└── Desktop/ DTB + 硬件参考 (3 项)

Week 3+: 低优先级 7 项 + 安全整改 5 项
```

---

> 本报告由全盘扫描自动生成，建议人工复核后再执行批量导入。特别注意 OneDrive/learn/ 是最高价值的知识来源，建议优先处理。
