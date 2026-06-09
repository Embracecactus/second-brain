---
tags: [mcp, eda, jlceda, ai]
category: tools/ai-eda
created: 2026-06-09
---
I now have enough information. Let me generate the Obsidian note.

```yaml
---
title: AI-EDA
category: tools/ai-eda
tags:
  - mcp
  - eda
  - hardware-design
  - jlceda
  - claude-code
  - typescript
  - websocket
  - schematic
  - pcb
  - ai-tools
created: 2026-06-09
status: active
version: v2.1.0
repository: https://github.com/keiller9/AI-EDA
---

# AI-EDA

## 项目概述

AI-EDA 是一个基于 MCP（Model Context Protocol）的桥接器，将 **Claude Code** 与**嘉立创EDA专业版**（JLCEDA Pro）连接，使硬件工程师能够通过自然语言操控原理图和 PCB 设计。项目提供 **122 个 MCP 工具**和 **15 个 Skill 技能**，覆盖原理图读写、PCB 读写、元件库、文档管理、DRC 规则管理、制造导出等完整 EDA 工作流。

## 技术栈

| 层级 | 技术 |
|------|------|
| MCP Server | Node.js, TypeScript, `@modelcontextprotocol/sdk` ^1.27.1, `ws` ^8.18.0, `zod` ^3.24.0 |
| EDA Extension | JLCEDA Pro Extension API, `@jlceda/pro-api-types` v0.2.15, TypeScript |
| 通信协议 | WebSocket（JSON 请求/响应，UUID 匹配），默认端口 8765 |
| 构建工具 | tsc（MCP Server）, esbuild（EDA Extension） |
| 测试框架 | Vitest（3 个测试套件，19 个测试） |
| 版本管理 | Node.js >= 18, JLCEDA Pro >= 2.3.0 |

## 架构与关键设计

### 整体架构

```
Claude Code <--Stdio--> MCP Server <--WebSocket(:8765)--> EDA Extension <--Native API--> JLCEDA Pro
```

项目采用**双进程桥接架构**，由三个核心组件构成：

1. **MCP Server**（`mcp-server/`）— Node.js 服务端，通过 stdio 与 Claude Code 通信，将 MCP 工具调用转发为 WebSocket 命令
2. **EDA Extension**（`eda-extension/`）— 运行在 JLCEDA Pro 内部的扩展，通过 WebSocket 接收命令，调用原生 EDA API 执行操作
3. **Skills**（`.claude/commands/`）— Claude Code 的 slash command 技能，注入 EDA 领域知识引导 LLM 完成复杂设计任务

### 请求/响应匹配机制

协议层（`protocol.ts`）定义了共享的 `BridgeCommand` 枚举（122 个命令），使用 UUID 进行请求-响应匹配：

```typescript
// protocol.ts — 核心协议定义
export interface BridgeRequest {
  id: string;        // req_${Date.now()}_${counter} 唯一标识
  type: 'request';
  command: BridgeCommand;
  params: Record<string, unknown>;
}

export interface BridgeResponse {
  id: string;        // 与请求 id 匹配
  type: 'response';
  success: boolean;
  data?: unknown;
  error?: string;
}
```

WSBridge 维护一个 `pendingRequests` Map，发送请求时注册 Promise，收到响应时通过 id 查找并 resolve，超时 30 秒自动 reject。

### 命令分发器（Dispatcher）

Extension 端的 `dispatcher.ts` 实现了命令路由、日志记录和统计功能：

```typescript
// dispatcher.ts — 命令分发核心
export async function dispatch(request: BridgeRequest): Promise<BridgeResponse> {
  const handler = handlers.get(request.command);
  if (!handler) {
    return createResponse(request.id, false, undefined, `Unknown command: ${request.command}`);
  }
  // 记录日志：command, timestamp, durationMs, success
  // 维护统计：totalCommands, totalSucceeded, totalDurationMs
}
```

### 工具注册模式

MCP Server 端使用 Zod schema 定义工具参数，通过 `server.tool()` 注册：

```typescript
// 典型工具注册模式
server.tool(
  'eda_sch_list_components',
  'List all components in the current schematic...',
  { filter: z.string().optional().describe('Optional filter string') },
  async ({ filter }) => {
    const data = await bridge.sendCommand(BridgeCommand.SCH_LIST_COMPONENTS, { filter });
    return { content: [{ type: 'text', text: JSON.stringify(data) }] };
  },
);
```

### 渐进式信息获取（Progressive Disclosure）

组件信息设计了三级查询深度：
1. `eda_sch_list_components` — 紧凑列表（id, designator, value, x, y, rotation）
2. `eda_sch_get_component` — 完整属性 + 引脚信息
3. `eda_sch_get_component_context` — 组件 + 连接网络 + 空间邻近组件

## 核心知识点

### 122 个 MCP 工具分布

| 分类 | 数量 | 说明 |
|------|------|------|
| 连接 | 1 | 连接状态检查 |
| 原理图读取 | 10 | 状态、器件、网络、导线、图元、选中、鼠标位置、包围盒 |
| 原理图写入 | 18 | 放置、画线、网络标识/标签、导航、批量操作、自动布局/布线 |
| PCB 读取 | 16 | 状态、器件、网络、层叠、图元、坐标变换、选中 |
| PCB 写入 | 45 | 放置、画线/弧/折线、过孔、文字、铺铜、DRC 规则 CRUD、网络类/差分对/等长组、路由控制、制造导出 |
| 文档管理 | 11 | 查看/打开文档、创建项目/原理图/PCB/Board |
| 元件库 | 5 | 搜索器件/封装、LCSC 编号查询 |
| 系统与复合 | 12 | DRC、BOM、设计概览、智能搜索、单位转换 |

### 15 个 Skill 技能

技能结合**领域知识**（EDA 设计规则、数值阈值）和 **MCP 工具序列**（调用顺序）：

- **API 参考类**（7 个）：`eda`, `eda-sch`, `eda-pcb`, `eda-lib`, `eda-dmt`, `eda-sys`, `eda-ref` — 覆盖 120 个类、62 个枚举、70 个接口
- **设计工作流类**（6 个）：`generate-schematic`（自然语言生成原理图）, `review-sch`, `review-pcb`, `design-check`, `place-components`, `route-traces`
- **知识类**（2 个）：`electrical-rules`, `component-research`

### 扩展 UI 面板

EDA Extension 提供两个 IFrame 面板：
- **状态面板**（`panel.html`）— 连接状态、命令统计、命令日志
- **AI 助手面板**（`ai-panel.html`）— 仪表盘、快捷操作、活动日志、关于页面

面板数据通过 `globalThis.__AI_EDA_PANEL_DATA__` 共享，每 2 秒更新面板数据，每 5 秒采集仪表盘数据。

### 上下文菜单集成

Extension 在 JLCEDA Pro 的原理图和 PCB 编辑器中注册了右键菜单：
- 原理图：AI 分析选中元件、AI 检查设计
- PCB：AI 审查布局、AI 检查 DRC

## 重要代码片段

### MCP Server 入口（index.ts）

```typescript
async function main() {
  const bridge = new WSBridge();
  const server = new McpServer({ name: 'jlceda-bridge', version: '1.0.0' });

  // 注册 6 类工具
  registerConnectionTools(server, bridge);
  registerSchematicReadTools(server, bridge);
  registerPcbReadTools(server, bridge);
  registerSchematicWriteTools(server, bridge);
  registerPcbWriteTools(server, bridge);
  registerSystemTools(server, bridge);

  await bridge.start();                    // 启动 WebSocket 服务器
  const transport = new StdioServerTransport();
  await server.connect(transport);         // 通过 stdio 连接 Claude Code
}
```

### WebSocket Bridge 核心（ws-bridge.ts）

```typescript
async sendCommand(command: BridgeCommand, params: Record<string, unknown> = {}): Promise<unknown> {
  const request = createRequest(command, params);
  return new Promise<unknown>((resolve, reject) => {
    const timer = setTimeout(() => {
      this.pendingRequests.delete(request.id);
      reject(new Error(`Request timed out after ${REQUEST_TIMEOUT_MS}ms`));
    }, REQUEST_TIMEOUT_MS);

    this.pendingRequests.set(request.id, {
      resolve: (response: BridgeResponse) => {
        response.success ? resolve(response.data) : reject(new Error(response.error));
      },
      reject, timer,
    });
    this.client!.send(JSON.stringify(request));
  });
}
```

### Extension 激活与自动连接（eda-extension/index.ts）

```typescript
export function activate(): void {
  const storedPort = eda.sys_Storage.getExtensionUserConfig('wsPort');
  if (typeof storedPort === 'number' && storedPort >= 1024 && storedPort <= 65535) {
    setPort(storedPort);
  }
  registerPanelActions();
  startPanelUpdates();
  connectToServer();  // 自动连接 MCP Server
}
```

## 构建/运行方法

### 前提条件

- Node.js >= 18
- 嘉立创EDA专业版（JLCEDA Pro）>= 2.3.0（桌面客户端，非网页版）
- Claude Code

### 构建步骤

```bash
# 1. 编译 MCP Server
cd mcp-server && npm install && npm run build

# 2. 编译 EDA Extension
cd eda-extension && npm install && npm run build
# 生成 eda-extension/build/dist/*.eext 文件

# 3. 在 JLCEDA Pro 中安装扩展
# 扩展管理器 → 导入 .eext 文件

# 4. 注册 MCP Server（.mcp.json 已配置）
# 确保 mcp-server/dist/index.js 路径正确

# 5. 连接
# JLCEDA Pro 桌面客户端 → AI Bridge → 连接 AI
```

### 测试

```bash
cd mcp-server
npm test            # 运行所有测试（vitest）
npm run test:watch  # 监听模式
```

### 可选：安装完整 API 参考 Skill

```bash
npx clawhub@latest install easyeda-api
```

## 相关笔记链接

- [[MCP-Protocol]] — Model Context Protocol 协议规范
- [[Claude-Code]] — Claude Code CLI 工具
- [[JLCEDA-Pro]] — 嘉立创EDA专业版
- [[WebSocket]] — WebSocket 通信协议
- [[Zod]] — TypeScript Schema 验证库
- [[Vitest]] — 测试框架
- [[PCB-Design]] — PCB 设计基础知识
- [[Schematic-Design]] — 原理图设计基础知识
- [[Hardware-Engineering]] — 硬件工程笔记索引

---

*项目路径: `/mnt/c/Users/lijian/workspace/codebuudy/AI-EDA`*
*版本: v2.1.0 | 122 MCP Tools | 15 Skills | 19 Tests*
*协议同步检查: `scripts/check-protocol-sync.sh` 验证两端 `protocol.ts` 枚举一致性*
```

## 相关笔记

- [[pcb]] — PCB 电源管理与电池充电电路设计
- [[power]] — 模块化可拆卸摄像头系统
- [[frdm-mcxa346]] — FRDM-MCXA346 开发板设计文件
- [[hardware-config]] — Jailhouse H3 嵌入式虚拟化配置
