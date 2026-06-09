---
tags: [fusion360, mcp, cad]
category: tools/fusion360
created: 2026-06-09
---
Based on my thorough exploration, this project directory contains only a single file: `.claude/settings.local.json`. There is no source code, README, configuration, or any other project files present. The directory appears to be an early-stage or placeholder project. I will generate the Obsidian note accordingly.

---
category: tools/fusion360
tags:
  - mcp
  - fusion360
  - autodesk
  - cad
  - 智能体工具
created: 2026-06-09
status: 🟡 规划中
---

# MCP Fusion360

## 项目概述

`mcp_fusion360` 是一个基于 **MCP (Model Context Protocol)** 的 Autodesk Fusion 360 集成工具项目，旨在为 AI 智能体提供与 Fusion 360 CAD 软件进行交互的能力。该项目当前处于初始化阶段，尚无实际代码实现。

## 技术栈

- **协议**: MCP (Model Context Protocol) — Anthropic 推出的智能体工具协议
- **目标软件**: Autodesk Fusion 360 — 专业级 3D CAD/CAM/CAE 软件
- **开发环境**: Claude Code (本地 AI 编程助手)
- **运行平台**: Linux (WSL2)

## 架构与设计决策

### 核心设计思路

该项目的设计目标是通过 MCP 协议将 Fusion 360 的功能暴露给 AI 智能体，典型的应用场景包括：

- 通过自然语言驱动 CAD 建模
- 自动化参数化设计流程
- 批量处理工程图或装配体
- AI 辅助的设计迭代与优化

### 权限配置

从现有的 `.claude/settings.local.json` 可以推断项目的工作环境：

```json
{
  "permissions": {
    "allow": [
      "Read(//home/lijian/.config/claude-code/**)",
      "Read(//home/lijian/AppData/Roaming/claude-code/**)",
      "Read(//mnt/c/Users/**)",
      "Read(//mnt/c/Users/lijian/AppData/Roaming/**)",
      "Bash(netstat -ano)"
    ]
  }
}
```

关键发现：
- 项目运行在 **WSL2** 环境下（`/mnt/c/` 路径挂载）
- 需要访问 **Windows 侧的用户配置目录**（`AppData/Roaming`）
- `netstat -ano` 权限暗示可能需要**网络端口监控**，推测 Fusion 360 的通信方式可能涉及本地 socket/API 端口
- 配置了读取 `claude-code` 配置目录的权限，说明项目与 Claude Code 工具链深度集成

## 项目状态

当前项目目录仅包含以下文件：

```
mcp_fusion360/
└── .claude/
    └── settings.local.json
```

**状态判断**: 项目处于非常早期的规划/初始化阶段，尚无源代码、文档或构建配置。

## 关键洞察

1. **MCP 生态扩展**: 该项目属于将 MCP 协议扩展到专业领域软件（CAD/工程设计）的尝试，是 MCP 在工业设计领域应用的探索
2. **跨平台通信挑战**: Fusion 360 运行在 Windows 上，而 MCP 服务端可能运行在 WSL2 Linux 环境中，跨平台通信是需要解决的关键技术问题
3. **Fusion 360 API**: 正式实现时可能需要利用 Fusion 360 的 Python API（Fusion 360 内置 Python 解释器）或 REST API 进行交互
4. **端口监控**: `netstat` 权限的配置暗示通信架构可能基于本地网络端口，可能是 Fusion 360 插件通过 HTTP/WebSocket 与 MCP 服务端通信

## 后续开发建议

- [ ] 确定 Fusion 360 的通信接口（Python Add-in / REST API / Socket）
- [ ] 定义 MCP 工具集（创建草图、拉伸、倒角、导出等）
- [ ] 实现 Fusion 360 端的 Add-in 或插件
- [ ] 实现 MCP Server 端逻辑
- [ ] 编写 README 和使用文档
- [ ] 添加构建配置（pyproject.toml / setup.py）

## 相关概念

- [[MCP Protocol]] — Model Context Protocol，智能体工具调用标准协议
- [[Autodesk Fusion 360]] — 云端 3D CAD/CAM/CAE 平台
- [[Claude Code]] — Anthropic CLI 编程工具
- [[WSL2]] — Windows Subsystem for Linux 2
- [[CAD 自动化]] — 计算机辅助设计自动化技术
- [[参数化设计]] — 通过参数驱动的 CAD 设计方法论

## 相关笔记

- [[AI-EDA]] — AI-EDA 智能 EDA 工具（同为 MCP 工具）
- [[pcb]] — PCB 电源管理与电池充电电路设计
- [[qt-projects]] — LinuxCNC CAM 五轴数控软件（CAD/CAM 相关）
