# OpenClaw 架构总览

本文档从宏观视角剖析 OpenClaw 的整体架构、核心组件以及它们之间的协作关系。

## 1. 系统架构图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    CLIENTS                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │ macOS    │  │ CLI      │  │ Web UI   │  │ Mobile   │  │ Nodes    │               │
│  │ App      │  │          │  │          │  │ Apps     │  │          │               │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘               │
│       │             │             │             │             │                      │
│       └─────────────┴─────────────┴─────────────┴─────────────┘                      │
│                                   │                                                  │
│                           WebSocket / HTTP                                          │
│                                   │                                                  │
└───────────────────────────────────┼──────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                  GATEWAY                                            │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                         WebSocket Server                                      │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │  │
│  │  │ Connection  │  │ Protocol    │  │ Auth        │  │ Event Streaming     │  │  │
│  │  │ Handshake   │  │ Validation  │  │ & Pairing   │  │ & Push              │  │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                         Agent Loop Core                                       │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │  │
│  │  │ Session     │  │ Prompt      │  │ Tool        │  │ Response            │  │  │
│  │  │ Management  │  │ Assembly    │  │ Execution   │  │ Streaming           │  │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                         Plugin Runtime                                        │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │  │
│  │  │ Discovery   │  │ Registry    │  │ Hooks       │  │ Capabilities        │  │  │
│  │  │ & Loading   │  │             │  │ Execution   │  │ Management          │  │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────┬──────────────────────────────────────────────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          │                         │                         │
          ▼                         ▼                         ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   PROVIDERS     │    │    CHANNELS     │    │   EXTENSIONS    │
│  ┌───────────┐  │    │  ┌───────────┐  │    │  ┌───────────┐  │
│  │ OpenAI    │  │    │  │ Discord   │  │    │  │ Skills    │  │
│  │ Anthropic │  │    │  │ Slack     │  │    │  │ Hooks     │  │
│  │ Google    │  │    │  │ Telegram  │  │    │  │ MCP       │  │
│  │ Mistral   │  │    │  │ WhatsApp  │  │    │  │ ...       │  │
│  └───────────┘  │    │  └───────────┘  │    │  └───────────┘  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## 2. 核心组件详解

### 2.1 Gateway（网关）

Gateway 是 OpenClaw 的核心进程，负责所有消息的集中处理。它是一个长期运行的守护进程，监听 WebSocket 连接（默认 `127.0.0.1:18789`）。

**主要职责：**
- 维护与所有消息通道的连接
- 提供类型化的 WebSocket API（请求、响应、服务器推送事件）
- 验证入站帧的 JSON Schema
- 分发事件（agent、chat、presence、health、cron）
- 提供 Canvas Host HTTP 服务（`/__openclaw__/canvas/` 和 `/__openclaw__/a2ui/`）

**代码位置：**
- Gateway 主入口：`src/cli/gateway-cli.ts`
- 服务器实现：`src/gateway/server.ts`
- 协议定义：`src/gateway/protocol/`

### 2.2 Agent Loop Core

Agent Loop 是 OpenClaw 的核心执行引擎，负责处理用户消息、调用 AI 模型、执行工具并流式返回响应。

**执行流程：**
```
1. 接收 agent RPC 请求 (agent / agent.wait)
2. 验证参数，解析 session (sessionKey/sessionId)
3. 解析模型 + 思考/verbose 默认值
4. 加载技能快照
5. 调用 runEmbeddedPiAgent（pi-agent-core 运行时）
6. 流式返回生命周期事件和助手响应
```

**关键模块：**
- Agent 命令：`src/commands/agent.ts`
- Agent 命令执行：`src/agents/agent-command.ts`
- Pi Agent 运行时：`src/agents/command/attempt-execution.ts`

### 2.3 Plugin Runtime

插件系统采用四层级架构：

**1. Manifest + Discovery（清单与发现）**
- 从配置路径、工作区根目录、全局扩展根目录和捆绑扩展中发现候选插件
- 读取原生 `openclaw.plugin.json` 清单

**2. Enablement + Validation（启用与验证）**
- 决定发现到的插件是启用、禁用、阻塞还是选择独占槽位

**3. Runtime Loading（运行时加载）**
- 原生 OpenClaw 插件通过 jiti 在进程内加载
- 注册能力到中央注册表

**4. Surface Consumption（表面消费）**
- 暴露工具、通道、提供者设置、钩子、HTTP 路由、CLI 命令和服务

**代码位置：**
- 插件发现：`src/plugins/discovery.ts`
- 插件加载：`src/plugins/loader.ts`
- 插件注册表：`src/plugins/contracts/registry.ts`

## 3. 数据流分析

### 3.1 入站消息流

```
用户消息 (Discord/Slack/etc.)
    │
    ▼
通道适配器 (Channel Adapter)
    │
    ▼
Gateway WebSocket Server
    │
    ▼
Agent Loop Core
    │
    ├─► Session Management
    ├─► Prompt Assembly (Skills + Context)
    ├─► Model Inference (Provider)
    ├─► Tool Execution (如果需要)
    └─► Response Streaming
```

### 3.2 工具执行流

```
Agent Loop
    │
    ▼
Tool Dispatcher
    │
    ├─► Bash Tool ──► Sandbox/Host Execution
    ├─► Message Tool ──► Channel Send
    ├─► Read Tool ──► File System
    ├─► Edit Tool ──► File Modification
    └─► Custom Tool ──► Plugin Handler
```

### 3.3 配置流

```
配置文件 (~/.openclaw/config.json5)
    │
    ▼
Config Loader (src/config/io.ts)
    │
    ├─► Schema Validation
    ├─► Plugin Resolution
    └─► Runtime Snapshot
         │
         ▼
    Plugin Discovery ──► Registry
         │
         ▼
    Agent Loop (使用配置)
```

## 4. 目录结构解析

```
openclaw/
├── src/
│   ├── agents/           # Agent 核心逻辑
│   │   ├── agent-command.ts      # Agent 命令主入口
│   │   ├── command/              # 命令执行相关
│   │   ├── tools/                # 工具实现
│   │   └── auth-profiles/        # 认证配置管理
│   ├── cli/              # CLI 实现
│   │   ├── gateway-cli.ts        # Gateway 命令
│   │   ├── daemon-cli.ts         # 守护进程管理
│   │   └── ...                   # 其他 CLI 命令
│   ├── commands/         # 命令处理（与 CLI 对应）
│   ├── config/           # 配置系统
│   │   ├── io.ts                 # 配置加载/保存
│   │   ├── types.ts              # 配置类型定义
│   │   └── validation.ts         # 配置验证
│   ├── gateway/          # Gateway 实现
│   │   ├── server.ts             # WebSocket 服务器
│   │   ├── protocol/             # 协议定义
│   │   └── control-plane/        # 控制平面
│   ├── hooks/            # 钩子系统
│   │   ├── hooks.ts              # 钩子 API
│   │   ├── loader.ts             # 钩子加载器
│   │   └── bundled/              # 捆绑钩子
│   ├── plugins/          # 插件系统
│   │   ├── discovery.ts          # 插件发现
│   │   ├── loader.ts             # 插件加载
│   │   └── contracts/            # 插件契约
│   ├── channels/         # 通道实现
│   └── plugin-sdk/       # 插件 SDK
├── skills/               # 技能定义
├── docs/                 # 文档
└── extensions/           # 捆绑插件
```

## 5. 关键技术决策

### 5.1 为什么使用 WebSocket？

- **实时双向通信**：支持流式响应和服务器推送
- **单一连接管理**：减少连接开销，简化状态同步
- **统一协议**：客户端（macOS、CLI、Web）使用相同协议

### 5.2 为什么使用 Plugin SDK？

- **明确的边界**：插件与核心代码隔离，通过 SDK 交互
- **类型安全**：TypeScript 类型定义确保契约一致性
- **可测试性**：插件可以独立测试

### 5.3 为什么使用 Hooks？

- **事件驱动**：响应 Agent 生命周期事件
- **可扩展性**：无需修改核心代码即可扩展功能
- **可组合性**：多个钩子可以链式执行

## 6. 扩展点

OpenClaw 提供了多个层次的扩展点：

| 扩展点 | 类型 | 用途 |
|--------|------|------|
| **Plugins** | 原生 | 模型提供者、通道、工具、能力 |
| **Hooks** | 脚本 | 生命周期事件自动化 |
| **Skills** | 声明式 | 工具选择与提示增强 |
| **MCP** | 协议 | Model Context Protocol 服务器 |

## 7. 相关文档

- [Agent 系统详解](02-agent-system.md)
- [插件系统详解](03-plugin-system.md)
- [Hook 机制详解](04-hook-system.md)
- [Skill 系统详解](05-skill-system.md)
- [网关与协议详解](08-gateway-protocol.md)
