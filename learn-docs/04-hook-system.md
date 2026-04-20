# OpenClaw Hook 机制详解

本文档深入剖析 OpenClaw 的 Hook 机制，包括事件类型、加载流程、执行模型和最佳实践。

## 1. Hook 系统架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                     HOOK SYSTEM                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    HOOK SOURCES                              │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐            │   │
│  │  │   Bundled   │ │   Managed   │ │   Workspace │            │   │
│  │  │   Hooks     │ │   Hooks     │ │   Hooks     │            │   │
│  │  │  (dist/)    │ │(~/.openclaw/)│ │ (<ws>/hooks)│            │   │
│  │  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘            │   │
│  │         └───────────────┼───────────────┘                   │   │
│  │                         │                                    │   │
│  │                         ▼                                    │   │
│  │              ┌─────────────────────┐                        │   │
│  │              │   Hook Discovery    │                        │   │
│  │              │   (HOOK.md + handler.ts)                     │   │
│  │              └─────────────────────┘                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    HOOK REGISTRY                             │   │
│  │  ┌─────────────────────────────────────────────────────┐   │   │
│  │  │  Event Map: eventType → Set<HookHandler>            │   │   │
│  │  │                                                      │   │   │
│  │  │  "command:new" → [sessionMemory, commandLogger]     │   │   │
│  │  │  "agent:bootstrap" → [bootstrapExtraFiles]          │   │   │
│  │  │  "gateway:start" → [bootMd]                         │   │   │
│  │  └─────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    HOOK EXECUTION                            │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │   │
│  │  │   Synchronous│  │   Sequential│  │   Error     │         │   │
│  │  │   Execution │  │   Execution │  │   Handling  │         │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 2. Hook 类型

### 2.1 内置钩子（Internal Hooks）

运行在主进程内，响应 Agent 生命周期事件。

| 事件类型 | 描述 | 触发时机 |
|----------|------|----------|
| `command:new` | /new 命令 | 用户发送 /new |
| `command:reset` | /reset 命令 | 用户发送 /reset |
| `command:stop` | /stop 命令 | 用户发送 /stop |
| `agent:bootstrap` | Agent 引导 | 构建系统 Prompt 时 |
| `gateway:start` | Gateway 启动 | Gateway 启动完成 |
| `gateway:stop` | Gateway 停止 | Gateway 即将停止 |
| `session:start` | 会话开始 | 新会话创建 |
| `session:end` | 会话结束 | 会话关闭 |

### 2.2 插件钩子（Plugin Hooks）

由插件注册，运行在主进程内。

| 事件类型 | 描述 | 用途 |
|----------|------|------|
| `before_model_resolve` | 模型解析前 | 覆盖模型选择 |
| `before_prompt_build` | Prompt 构建前 | 注入上下文 |
| `before_agent_start` | Agent 启动前 | 遗留兼容 |
| `agent_end` | Agent 结束 | 后处理 |
| `before_tool_call` | 工具调用前 | 拦截/修改工具调用 |
| `after_tool_call` | 工具调用后 | 处理工具结果 |
| `message_received` | 收到消息 | 入站消息处理 |
| `message_sending` | 发送消息 | 出站消息处理 |

### 2.3 Webhook（外部钩子）

接收外部 HTTP 请求触发工作。

```
外部系统 ──HTTP POST──► Gateway ──► 触发 Agent 执行
```

## 3. Hook 发现机制

### 3.1 发现来源

```typescript
// src/hooks/loader.ts

export async function loadInternalHooks(
  cfg: OpenClawConfig,
  workspaceDir: string
): Promise<number> {
  let loadedCount = 0;

  // 1. 从目录加载（新系统）
  const hookEntries = loadWorkspaceHookEntries(workspaceDir, {
    config: cfg,
    managedHooksDir: opts?.managedHooksDir,
    bundledHooksDir: opts?.bundledHooksDir,
  });

  // 2. 过滤符合条件的钩子
  const eligible = hookEntries.filter((entry) =>
    shouldIncludeHook({ entry, config: cfg })
  );

  // 3. 加载每个钩子
  for (const entry of eligible) {
    await loadHookHandler(entry);
    loadedCount++;
  }

  return loadedCount;
}
```

### 3.2 加载优先级

```
优先级从高到低：

1. Bundled Hooks (dist/hooks/bundled/)
   - 随 OpenClaw 一起发布
   - 可靠稳定

2. Plugin Hooks
   - 插件内部声明
   - 插件依赖的一部分

3. Managed Hooks (~/.openclaw/hooks/)
   - 用户安装
   - 可覆盖 Bundled 钩子

4. Workspace Hooks (<workspace>/hooks/)
   - 项目本地
   - 默认禁用，需显式启用
```

## 4. Hook 结构

### 4.1 目录结构

```
my-hook/
├── HOOK.md          # 元数据和文档
├── handler.ts       # TypeScript 实现
└── handler.js       # JavaScript 实现（备选）
```

### 4.2 HOOK.md 格式

```markdown
---
name: my-hook
description: "Short description"
homepage: https://example.com/docs
metadata:
  openclaw:
    emoji: "🔗"
    events: ["command:new", "command:reset"]
    requires:
      bins: ["node"]
      env: ["MY_VAR"]
      config: ["workspace.dir"]
    os: ["darwin", "linux"]
    always: false
    export: "default"
---

# My Hook

详细文档...
```

### 4.3 Handler 实现

```typescript
// handler.ts

import type { HookEvent, HookContext, HookResult } from 'openclaw/hooks';

// 默认导出
export default async function myHandler(
  event: HookEvent,
  context: HookContext
): Promise<HookResult> {
  // 只处理特定事件
  if (event.type !== 'command' || event.action !== 'new') {
    return { continue: true };
  }

  // 执行操作
  console.log(`Session ${event.sessionKey} reset`);

  // 返回结果
  return {
    continue: true,
    data: {
      timestamp: Date.now(),
    },
  };
}

// 或者命名导出
export async function namedHandler(event: HookEvent): Promise<HookResult> {
  // ...
}
```

## 5. Hook 执行模型

### 5.1 同步执行

```typescript
// src/hooks/internal-hooks.ts

export type InternalHookHandler = (
  event: InternalHookEvent
) => Promise<InternalHookResult> | InternalHookResult;

const hookRegistry = new Map<string, Set<InternalHookHandler>>();

export async function triggerInternalHook(
  eventType: string,
  event: InternalHookEvent
): Promise<InternalHookResult> {
  const handlers = hookRegistry.get(eventType) ?? new Set();

  for (const handler of handlers) {
    try {
      const result = await handler(event);

      // 处理阻止信号
      if (result.block) {
        return {
          continue: false,
          block: true,
          reason: result.reason,
        };
      }
    } catch (err) {
      log.error(`Hook handler failed for ${eventType}:`, err);
    }
  }

  return { continue: true };
}
```

### 5.2 执行决策规则

| 钩子类型 | 阻止信号 | 行为 |
|----------|----------|------|
| `before_tool_call` | `{ block: true }` | 终止执行，停止低优先级处理器 |
| `before_tool_call` | `{ block: false }` | 无操作，不清除之前的阻止 |
| `before_install` | `{ block: true }` | 终止执行，阻止安装 |
| `message_sending` | `{ cancel: true }` | 取消消息发送 |

### 5.3 Fire and Forget

```typescript
// src/hooks/fire-and-forget.ts

export function fireAndForgetHook(
  eventType: string,
  event: HookEvent
): void {
  // 不等待钩子完成
  triggerInternalHook(eventType, event).catch((err) => {
    log.error(`Fire-and-forget hook failed:`, err);
  });
}
```

## 6. 捆绑钩子详解

### 6.1 session-memory

保存会话上下文到工作区。

```typescript
// src/hooks/bundled/session-memory/handler.ts

export default async function sessionMemoryHandler(event: HookEvent) {
  if (event.type !== 'command' ||
      (event.action !== 'new' && event.action !== 'reset')) {
    return { continue: true };
  }

  // 保存会话上下文
  const memoryDir = path.join(event.workspaceDir, 'memory');
  const timestamp = new Date().toISOString();
  const fileName = `${timestamp}-${event.sessionKey}.json`;

  await fs.mkdir(memoryDir, { recursive: true });
  await fs.writeFile(
    path.join(memoryDir, fileName),
    JSON.stringify({
      sessionKey: event.sessionKey,
      timestamp,
      context: event.context,
    }, null, 2)
  );

  return { continue: true };
}
```

### 6.2 command-logger

记录所有命令事件。

```typescript
// src/hooks/bundled/command-logger/handler.ts

const logFile = path.join(
  process.env.OPENCLAW_STATE_DIR ?? '~/.openclaw',
  'logs/commands.log'
);

export default async function commandLogger(event: HookEvent) {
  if (event.type !== 'command') {
    return { continue: true };
  }

  const logEntry = {
    timestamp: new Date().toISOString(),
    type: event.type,
    action: event.action,
    sessionKey: event.sessionKey,
    agentId: event.agentId,
  };

  await fs.appendFile(
    logFile,
    JSON.stringify(logEntry) + '\n'
  );

  return { continue: true };
}
```

### 6.3 bootstrap-extra-files

在 Agent 引导时注入额外文件。

```typescript
// src/hooks/bundled/bootstrap-extra-files/handler.ts

export default async function bootstrapExtraFiles(event: HookEvent) {
  if (event.type !== 'agent:bootstrap') {
    return { continue: true };
  }

  const extraFiles = event.config.bootstrap?.extraFiles ?? [];
  const files: Array<{ name: string; content: string }> = [];

  for (const pattern of extraFiles) {
    const matches = await glob(pattern, {
      cwd: event.workspaceDir,
    });

    for (const file of matches) {
      const content = await fs.readFile(file, 'utf-8');
      files.push({
        name: path.basename(file),
        content,
      });
    }
  }

  return {
    continue: true,
    data: {
      bootstrapFiles: files,
    },
  };
}
```

### 6.4 boot-md

Gateway 启动时运行 BOOT.md。

```typescript
// src/hooks/bundled/boot-md/handler.ts

export default async function bootMdHandler(event: HookEvent) {
  if (event.type !== 'gateway:start') {
    return { continue: true };
  }

  const bootFile = path.join(
    event.workspaceDir,
    'BOOT.md'
  );

  try {
    const content = await fs.readFile(bootFile, 'utf-8');
    // 执行 BOOT.md 中的指令
    await executeBootInstructions(content);
  } catch (err) {
    // BOOT.md 不存在或读取失败，静默处理
  }

  return { continue: true };
}
```

## 7. 插件钩子详解

### 7.1 before_model_resolve

在模型解析前运行，用于确定性覆盖模型选择。

```typescript
// 插件注册
api.registerHook('before_model_resolve', async (context) => {
  // context 中没有 messages，只有配置信息
  const { agentId, config, preferredProvider } = context;

  // 基于规则选择模型
  if (agentId === 'coding-agent') {
    return {
      continue: true,
      data: {
        provider: 'anthropic',
        model: 'claude-sonnet-4-6',
      },
    };
  }

  return { continue: true };
});
```

### 7.2 before_prompt_build

在 Prompt 构建后运行，可以注入上下文。

```typescript
api.registerHook('before_prompt_build', async (context) => {
  // context 中有 messages
  const { messages, sessionKey } = context;

  return {
    continue: true,
    data: {
      // 注入到系统 Prompt
      prependSystemContext: 'You are a helpful assistant.',
      // 或添加到消息前
      prependContext: 'Context from hook:\n...',
    },
  };
});
```

### 7.3 before_tool_call

在工具调用前拦截。

```typescript
api.registerHook('before_tool_call', async (context) => {
  const { toolName, params } = context;

  // 阻止危险命令
  if (toolName === 'bash') {
    const command = params.command as string;
    if (command.includes('rm -rf /')) {
      return {
        continue: false,
        block: true,
        reason: 'Dangerous command blocked',
      };
    }
  }

  return { continue: true };
});
```

## 8. Hook Pack（钩子包）

### 8.1 结构

```
my-hook-pack/
├── package.json
├── hooks/
│   ├── hook-a/
│   │   ├── HOOK.md
│   │   └── handler.ts
│   └── hook-b/
│       ├── HOOK.md
│       └── handler.ts
└── node_modules/
```

### 8.2 package.json

```json
{
  "name": "@myorg/openclaw-hooks",
  "version": "1.0.0",
  "openclaw": {
    "hooks": [
      "./hooks/hook-a",
      "./hooks/hook-b"
    ]
  }
}
```

### 8.3 安装

```bash
openclaw plugins install @myorg/openclaw-hooks
```

## 9. 关键代码文件

| 文件 | 描述 |
|------|------|
| `src/hooks/types.ts` | Hook 类型定义 |
| `src/hooks/hooks.ts` | 公共 Hook API |
| `src/hooks/internal-hooks.ts` | 内部 Hook 实现 |
| `src/hooks/loader.ts` | Hook 加载器 |
| `src/hooks/workspace.ts` | 工作区钩子发现 |
| `src/hooks/config.ts` | 钩子配置 |
| `src/hooks/bundled/` | 捆绑钩子 |

## 10. 最佳实践

### 10.1 钩子开发指南

1. **事件过滤**：只处理感兴趣的事件
2. **快速返回**：避免长时间阻塞
3. **错误处理**：捕获并记录错误
4. **幂等性**：同一事件多次触发应安全
5. **资源清理**：使用 try/finally

### 10.2 示例模板

```typescript
import type { HookEvent, HookContext, HookResult } from 'openclaw/hooks';
import { createSubsystemLogger } from 'openclaw/logging';

const log = createSubsystemLogger('hooks:my-hook');

export default async function myHook(
  event: HookEvent,
  context: HookContext
): Promise<HookResult> {
  // 1. 过滤事件
  if (!shouldHandle(event)) {
    return { continue: true };
  }

  log.debug(`Handling event: ${event.type}`);

  try {
    // 2. 执行操作
    await doSomething(event, context);

    // 3. 返回成功
    return {
      continue: true,
      data: { processed: true },
    };
  } catch (err) {
    // 4. 错误处理
    log.error('Hook failed:', err);

    // 不阻止其他钩子
    return { continue: true };
  }
}

function shouldHandle(event: HookEvent): boolean {
  return event.type === 'command' && event.action === 'new';
}

async function doSomething(
  event: HookEvent,
  context: HookContext
): Promise<void> {
  // 实现逻辑
}
```

### 10.3 调试技巧

```bash
# 查看所有钩子
openclaw hooks list

# 查看钩子详情
openclaw hooks info session-memory

# 检查钩子状态
openclaw hooks check

# 启用/禁用钩子
openclaw hooks enable my-hook
openclaw hooks disable my-hook
```
