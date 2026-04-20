# OpenClaw Agent 系统详解

本文档深入剖析 OpenClaw 的 Agent 系统，包括生命周期管理、命令执行流程、ACP 协议以及核心代码实现。

## 1. Agent 架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                     AGENT SYSTEM                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │   Entry      │───►│   Agent      │───►│   Session    │      │
│  │   Points     │    │   Command    │    │   Manager    │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
│         │                   │                   │               │
│         ▼                   ▼                   ▼               │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │  Gateway     │    │  Pi Agent    │    │  Auth        │      │
│  │  RPC         │    │  Runtime     │    │  Profiles    │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  TOOL EXECUTION                         │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌────────────┐ │   │
│  │  │  Bash   │  │ Message │  │  Read   │  │   Edit     │ │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └────────────┘ │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 2. Agent 入口点

Agent 可以通过多种方式触发：

### 2.1 Gateway RPC 入口

```typescript
// src/gateway/server.ts
// 处理 'agent' RPC 方法
{
  type: "req",
  id: string,
  method: "agent",
  params: {
    message: string,
    sessionKey?: string,
    sessionId?: string,
    agentId?: string,
    channel?: string,
    thinking?: string,
    // ... 其他参数
  }
}
```

### 2.2 CLI 入口

```typescript
// src/cli/agent-cli.ts
export function registerAgentCli(cli: Command) {
  cli
    .command('agent')
    .option('-m, --message <msg>', 'Message to send')
    .option('-a, --agent-id <id>', 'Agent ID')
    .option('--thinking <level>', 'Thinking level')
    .action(async (opts) => {
      // 调用 agentCommand
    });
}
```

## 3. Agent 命令执行流程

### 3.1 主执行流程

```typescript
// src/agents/agent-command.ts (简化)

export async function agentCommand(
  deps: CliDeps,
  opts: AgentCommandOpts
): Promise<AgentCommandResult> {
  // 1. 加载配置
  const config = await loadConfig();

  // 2. 解析 session
  const session = await resolveSession(config, opts);

  // 3. 解析模型
  const model = await resolveConfiguredModelRef(config, opts);

  // 4. 加载技能
  const skills = await buildWorkspaceSkillSnapshot(config);

  // 5. 运行 Agent
  const result = await runEmbeddedPiAgent({
    config,
    session,
    model,
    skills,
    message: opts.message,
  });

  // 6. 返回结果
  return result;
}
```

### 3.2 详细执行步骤

```
agentCommand
    │
    ├──► loadConfig() ──────────────────────────────┐
    │                                                 │
    ├──► resolveSession() ──► SessionEntry          │
    │       ├── 解析 sessionKey/sessionId             │
    │       ├── 创建工作目录                          │
    │       └── 加载历史会话                          │
    │                                                 │
    ├──► resolveModel() ──► ModelRef                │
    │       ├── 应用模型覆盖                          │
    │       ├── 应用思考级别                          │
    │       └── 解析提供者配置                        │
    │                                                 │
    ├──► buildWorkspaceSkillSnapshot()              │
    │       ├── 扫描技能目录                          │
    │       ├── 解析 SKILL.md                         │
    │       └── 构建技能上下文                        │
    │                                                 │
    ├──► ensureAuthProfileStore() ──► AuthProfile   │
    │       ├── 加载认证配置                          │
    │       ├── 解析 OAuth 令牌                       │
    │       └── 轮换 API Key                          │
    │                                                 │
    └──► runEmbeddedPiAgent() ◄─────────────────────┘
            │
            ├── 序列化执行（通过队列）
            ├── 构建 Pi Session
            ├── 订阅 Pi 事件
            ├── 执行推理循环
            └── 流式返回结果
```

## 4. Pi Agent Runtime

Pi Agent Runtime 是 OpenClaw 的核心推理引擎，负责与 AI 模型交互。

### 4.1 核心实现

```typescript
// src/agents/command/attempt-execution.ts

export async function runEmbeddedPiAgent(
  params: EmbeddedPiAgentParams
): Promise<PiAgentResult> {
  // 1. 获取队列（按 sessionKey 序列化）
  const lane = getSessionLane(params.sessionKey);

  // 2. 加入队列等待执行
  return lane.run(async () => {
    // 3. 构建 Pi Session
    const piSession = await buildPiSession(params);

    // 4. 订阅事件
    const unsubscribe = subscribeEmbeddedPiSession({
      piSession,
      onEvent: (event) => {
        // 流式发送事件到客户端
        emitAgentEvent(event);
      }
    });

    try {
      // 5. 运行推理
      const result = await piSession.run({
        messages: buildMessages(params),
        tools: buildTools(params),
        timeoutMs: params.timeoutMs,
      });

      // 6. 返回结果
      return result;
    } finally {
      unsubscribe();
    }
  });
}
```

### 4.2 事件流处理

```typescript
// src/agents/command/attempt-execution.ts

export function subscribeEmbeddedPiSession(params: {
  piSession: PiSession;
  onEvent: (event: AgentEvent) => void;
}): () => void {
  return params.piSession.subscribe((piEvent) => {
    switch (piEvent.type) {
      case 'lifecycle':
        // 生命周期事件：start / end / error
        params.onEvent({
          stream: 'lifecycle',
          phase: piEvent.phase,
          runId: piEvent.runId,
        });
        break;

      case 'assistant':
        // 助手响应增量
        params.onEvent({
          stream: 'assistant',
          delta: piEvent.delta,
          text: piEvent.text,
        });
        break;

      case 'tool':
        // 工具执行事件
        params.onEvent({
          stream: 'tool',
          toolCallId: piEvent.toolCallId,
          name: piEvent.name,
          arguments: piEvent.arguments,
          result: piEvent.result,
        });
        break;
    }
  });
}
```

## 5. 队列与并发控制

### 5.1 Session Lane 机制

```typescript
// src/agents/lanes.ts

// 每个 sessionKey 有一个独立的队列
const laneMap = new Map<string, SessionLane>();

export function getSessionLane(sessionKey: string): SessionLane {
  if (!laneMap.has(sessionKey)) {
    laneMap.set(sessionKey, createLane({
      name: `session:${sessionKey}`,
      concurrency: 1  // 每个 session 串行执行
    }));
  }
  return laneMap.get(sessionKey)!;
}

// 全局 Lane（可选）
export const AGENT_LANE_SUBAGENT = createLane({
  name: 'subagent',
  concurrency: 4  // 子 Agent 并发数
});
```

### 5.2 队列模式

消息通道可以选择不同的队列模式：

```typescript
// 收集模式：收集消息，批量处理
export const QUEUE_MODE_COLLECT = 'collect';

// 引导模式：主动引导对话方向
export const QUEUE_MODE_STEER = 'steer';

// 跟进模式：被动响应
export const QUEUE_MODE_FOLLOWUP = 'followup';
```

## 6. 会话管理

### 6.1 Session Entry 结构

```typescript
// src/config/types.ts

export interface SessionEntry {
  sessionKey: string;
  sessionId: string;
  agentId: string;

  // 模型覆盖
  providerOverride?: string;
  modelOverride?: string;

  // 认证覆盖
  authProfileOverride?: string;

  // 运行时状态
  lastRunAt?: string;
  messageCount?: number;

  // 回退通知状态
  fallbackNoticeSelectedModel?: string;
  fallbackNoticeActiveModel?: string;
  fallbackNoticeReason?: string;
}
```

### 6.2 Session Store

```typescript
// src/commands/agent/session-store.ts

export interface SessionStore {
  // 加载会话
  load(sessionKey: string): Promise<SessionEntry | undefined>;

  // 保存会话
  save(entry: SessionEntry): Promise<void>;

  // 删除会话
  delete(sessionKey: string): Promise<void>;

  // 列出所有会话
  list(): Promise<SessionEntry[]>;
}

// 实现：基于文件系统
export class FileSystemSessionStore implements SessionStore {
  constructor(private baseDir: string) {}

  async load(sessionKey: string): Promise<SessionEntry | undefined> {
    const filePath = path.join(this.baseDir, `${sessionKey}.json`);
    try {
      const content = await fs.readFile(filePath, 'utf-8');
      return JSON.parse(content);
    } catch {
      return undefined;
    }
  }
  // ...
}
```

## 7. 认证配置管理

### 7.1 Auth Profile 系统

```typescript
// src/agents/auth-profiles/types.ts

export interface AuthProfile {
  id: string;
  provider: string;
  type: 'api_key' | 'oauth' | 'custom';

  // API Key 配置
  apiKey?: string;

  // OAuth 配置
  oauth?: {
    accessToken: string;
    refreshToken?: string;
    expiresAt?: string;
  };

  // 自定义配置
  config?: Record<string, unknown>;

  // 使用统计
  lastUsedAt?: string;
  failureCount?: number;
  successCount?: number;
}
```

### 7.2 认证配置解析流程

```
resolveAuthProfile()
    │
    ├──► 检查 session 覆盖
    │       └── authProfileOverride?
    │
    ├──► 检查 provider 配置
    │       └── config.providers[provider].auth?
    │
    ├──► 检查环境变量
    │       └── OPENAI_API_KEY, ANTHROPIC_API_KEY, etc.
    │
    ├──► 检查密钥管理器
    │       └── 1Password, macOS Keychain, etc.
    │
    └──► 返回 AuthProfile
```

## 8. ACP (Agent Communication Protocol)

ACP 允许远程 Agent 执行和节点通信。

### 8.1 ACP Session

```typescript
// src/acp/control-plane/manager.ts

export interface AcpSession {
  sessionId: string;
  nodeId: string;
  role: 'agent' | 'node';
  capabilities: string[];

  // 发送消息
  send(message: AcpMessage): Promise<void>;

  // 接收消息
  onMessage(handler: (msg: AcpMessage) => void): void;

  // 关闭会话
  close(): Promise<void>;
}
```

### 8.2 ACP 绑定架构

```
┌─────────────────────────────────────────┐
│           Control Plane                 │
│  ┌─────────────────────────────────┐   │
│  │      ACP Session Manager        │   │
│  │                                 │   │
│  │  ┌─────────┐    ┌─────────┐    │   │
│  │  │ Session │◄──►│ Session │    │   │
│  │  │   A     │    │   B     │    │   │
│  │  └────┬────┘    └────┬────┘    │   │
│  │       │              │         │   │
│  │       └──────────────┘         │   │
│  │              │                 │   │
│  │              ▼                 │   │
│  │      ┌─────────────┐           │   │
│  │      │   Router    │           │   │
│  │      └─────────────┘           │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
              │
    ┌─────────┴─────────┐
    ▼                   ▼
┌─────────┐      ┌─────────┐
│  Node   │      │  Node   │
│ Agent   │      │ Agent   │
└─────────┘      └─────────┘
```

## 9. 工具系统

### 9.1 工具注册

```typescript
// src/agents/tools.ts

export interface Tool {
  name: string;
  description: string;
  parameters: JSONSchema;

  execute(params: unknown, context: ToolContext): Promise<ToolResult>;
}

export interface ToolContext {
  sessionKey: string;
  agentId: string;
  workspaceDir: string;
  config: OpenClawConfig;
  // ...
}
```

### 9.2 内置工具

| 工具 | 描述 | 代码位置 |
|------|------|----------|
| **bash** | 执行 shell 命令 | `src/agents/bash-tools.ts` |
| **read** | 读取文件 | `src/agents/tools/read.ts` |
| **edit** | 编辑文件 | `src/agents/tools/edit.ts` |
| **write** | 写入文件 | `src/agents/tools/write.ts` |
| **message** | 发送消息到通道 | `src/agents/tools/message.ts` |
| **web_search** | 网络搜索 | `src/agents/tools/web-search.ts` |
| **web_fetch** | 获取网页内容 | `src/agents/tools/web-fetch.ts` |
| **glob** | 文件匹配 | `src/agents/tools/glob.ts` |
| **grep** | 内容搜索 | `src/agents/tools/grep.ts` |

### 9.3 Bash 工具详解

```typescript
// src/agents/bash-tools.ts

export async function executeBash(
  params: BashParams,
  context: ToolContext
): Promise<BashResult> {
  // 1. 安全检查
  if (!isCommandAllowed(params.command)) {
    throw new SecurityError('Command not allowed');
  }

  // 2. 申请执行权限
  const approval = await requestApproval({
    tool: 'bash',
    command: params.command,
    description: params.description,
  });

  if (!approval.granted) {
    throw new ApprovalDeniedError('User denied approval');
  }

  // 3. 执行命令
  const result = await spawnCommand({
    command: params.command,
    cwd: context.workspaceDir,
    timeout: params.timeout ?? 120000,
    pty: params.pty,
    background: params.background,
  });

  // 4. 返回结果
  return {
    exitCode: result.exitCode,
    stdout: result.stdout,
    stderr: result.stderr,
    sessionId: params.background ? result.sessionId : undefined,
  };
}
```

## 10. 流式响应

### 10.1 事件类型

```typescript
// src/agents/events.ts

export type AgentEvent =
  | LifecycleEvent
  | AssistantEvent
  | ToolEvent
  | CompactionEvent;

export interface LifecycleEvent {
  stream: 'lifecycle';
  phase: 'start' | 'end' | 'error';
  runId: string;
  error?: string;
}

export interface AssistantEvent {
  stream: 'assistant';
  delta?: string;      // 增量文本
  text?: string;       // 完整文本（可选）
  reasoning?: string;  // 思考过程
}

export interface ToolEvent {
  stream: 'tool';
  toolCallId: string;
  name: string;
  arguments?: unknown;
  result?: unknown;
  error?: string;
}
```

### 10.2 流式实现

```typescript
// Gateway WebSocket 流式推送

async function streamAgentResponse(
  ws: WebSocket,
  runId: string,
  message: string
) {
  // 发送生命周期开始
  ws.send(JSON.stringify({
    type: 'event',
    event: 'agent',
    payload: {
      stream: 'lifecycle',
      phase: 'start',
      runId,
    }
  }));

  // 订阅 Agent 事件
  const events = await runAgent({ message, runId });

  for await (const event of events) {
    ws.send(JSON.stringify({
      type: 'event',
      event: 'agent',
      payload: event,
    }));

    // 生命周期结束则停止
    if (event.stream === 'lifecycle' &&
        (event.phase === 'end' || event.phase === 'error')) {
      break;
    }
  }
}
```

## 11. 超时与取消

### 11.1 超时处理

```typescript
// src/agents/timeout.ts

export function resolveAgentTimeoutMs(
  config: OpenClawConfig,
  overrides?: { timeoutMs?: number }
): number {
  // 优先级：调用参数 > Agent 配置 > 全局默认
  return overrides?.timeoutMs
    ?? config.agents?.defaults?.timeoutSeconds * 1000
    ?? 172800000; // 48 小时默认
}

// 在 runEmbeddedPiAgent 中使用
const timeoutMs = resolveAgentTimeoutMs(config, opts);

const timeoutHandle = setTimeout(() => {
  piSession.abort(new TimeoutError(`Agent timed out after ${timeoutMs}ms`));
}, timeoutMs);
```

### 11.2 取消信号

```typescript
export interface AgentCommandOpts {
  // ...
  abortSignal?: AbortSignal;
}

// 使用
if (opts.abortSignal) {
  opts.abortSignal.addEventListener('abort', () => {
    piSession.abort(new AbortError('Agent execution cancelled'));
  });
}
```

## 12. 关键代码文件

| 文件 | 描述 |
|------|------|
| `src/agents/agent-command.ts` | Agent 命令主入口 |
| `src/commands/agent.ts` | CLI Agent 命令实现 |
| `src/agents/command/attempt-execution.ts` | Pi Agent 运行时封装 |
| `src/agents/lanes.ts` | 队列与并发控制 |
| `src/agents/auth-profiles.ts` | 认证配置管理 |
| `src/agents/tools.ts` | 工具注册与分发 |
| `src/agents/bash-tools.ts` | Bash 工具实现 |
| `src/acp/control-plane/manager.ts` | ACP 会话管理 |

## 13. 总结

OpenClaw 的 Agent 系统是一个精心设计的异步执行引擎，具有以下特点：

1. **队列化执行**：通过 Session Lane 保证每个会话的串行执行
2. **流式响应**：实时推送生命周期、助手和工具事件
3. **可扩展工具**：内置丰富工具，支持插件扩展
4. **灵活认证**：支持 API Key、OAuth 和自定义认证方式
5. **ACP 协议**：支持远程 Agent 执行和分布式节点
