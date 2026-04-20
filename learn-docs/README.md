# OpenClaw 项目深度分析文档

本文档系列旨在从不同角度和维度全面梳理 OpenClaw 开源项目的架构、机制与实现方式。

## 文档导航

1. **[架构总览](01-architecture-overview.md)** - 项目整体架构、核心组件与数据流
2. **[Agent 系统](02-agent-system.md)** - Agent 生命周期、命令执行与 ACP 协议
3. **[插件系统](03-plugin-system.md)** - 插件发现、加载、能力与生命周期
4. **[Hook 机制](04-hook-system.md)** - 事件驱动钩子系统与自动化
5. **[Skill 系统](05-skill-system.md)** - 技能定义、发现与远程技能
6. **[命令系统](06-command-system.md)** - CLI 架构、命令注册与执行
7. **[配置系统](07-config-system.md)** - 配置加载、验证与运行时管理
8. **[网关与协议](08-gateway-protocol.md)** - Gateway 架构、WebSocket 协议与节点通信
9. **[通道系统](09-channel-system.md)** - 消息通道、路由与多平台集成
10. **[模型与推理](10-model-inference.md)** - 模型提供者、推理循环与能力注册

## 项目简介

OpenClaw 是一个多通道 AI 网关，支持可扩展的消息集成。它通过统一的 Gateway 架构连接多种 AI 模型提供者（如 OpenAI、Anthropic）和消息通道（如 Discord、Slack、Telegram、WhatsApp 等），实现 AI Agent 与各种通信平台的无缝交互。

### 核心特性

- **多通道支持**：Discord、Slack、Telegram、WhatsApp、Signal、iMessage 等
- **多模型支持**：OpenAI、Anthropic、Google 等主流 AI 模型
- **插件化架构**：支持原生插件和 Bundled 插件扩展
- **Hook 自动化**：事件驱动的自动化脚本系统
- **Skill 系统**：声明式技能定义与动态加载
- **ACP 协议**：Agent Communication Protocol 支持远程 Agent 执行

## 快速参考

```bash
# 启动 Gateway
openclaw gateway

# 查看状态
openclaw status

# 列出插件
openclaw plugins list

# 列出钩子
openclaw hooks list

# Agent 命令
openclaw agent --message "Hello"
```

---

*文档生成日期：2026-04-01*
