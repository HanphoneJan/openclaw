# OpenClaw 网关与协议详解

本文档深入剖析 OpenClaw 的 Gateway 架构和通信协议，包括 WebSocket 协议、连接管理和消息路由。

## 1. Gateway 架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                     GATEWAY ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   GATEWAY SERVER                             │   │
│  │  ┌─────────────────────────────────────────────────────┐   │   │
│  │  │  HTTP Server (Express/Fastify)                      │   │   │
│  │  │  ├── /__openclaw__/canvas/   (Canvas Host)          │   │   │
│  │  │  ├── /__openclaw__/a2ui/     (A2UI Host)            │   │   │
│  │  │  └── /health                (Health Check)          │   │   │
│  │  └─────────────────────────────────────────────────────┘   │   │
│  │                         │                                    │   │
│  │                         ▼                                    │   │
│  │  ┌─────────────────────────────────────────────────────┐   │   │
│  │  │  WebSocket Server (ws)                              │   │   │
│  │  │  ├── Connection Manager                             │   │   │
│  │  │  ├── Message Router                                 │   │   │
│  │  │  └── Protocol Handler                               │   │   │
│  │  └─────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   CLIENT CONNECTIONS                         │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │   │
│  │  │  macOS   │  │   CLI    │  │   Web    │  │  Nodes   │    │   │
│  │  │   App    │  │          │  │   UI     │  │          │    │   │
│  │  │(Control) │  │(Control) │  │(Control) │  │ (Device) │    │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 2. WebSocket 协议

### 2.1 协议概述

- **传输**: WebSocket (ws/wss)
- **默认端口**: 18789
- **帧格式**: JSON 文本帧
- **编码**: UTF-8

### 2.2 连接生命周期

```
┌─────────┐                              ┌──────────┐
│ Client  │                              │ Gateway  │
└────┬────┘                              └────┬─────┘
     │                                         │
     │  1. WebSocket Connect                   │
     │ ───────────────────────────────────────►│
     │                                         │
     │  2. Send "connect" request              │
     │ {                                       │
     │   type: "req",                          │
     │   id: "1",                              │
     │   method: "connect",                    │
     │   params: {                             │
     │     deviceId: "uuid",                   │
     │     platform: "macos",                  │
     │     version: "2026.4.1",                │
     │     role: "control",  // or "node"      │
     │     capabilities: [...],                │
     │     auth: { token: "..." }  // optional │
     │   }                                     │
     │ }                                       │
     │ ───────────────────────────────────────►│
     │                                         │
     │  3. Challenge (if new device)           │
     │ ◄───────────────────────────────────────│
     │ {                                       │
     │   type: "res",                          │
     │   id: "1",                              │
     │   ok: true,                             │
     │   payload: {                            │
     │     challenge: "nonce-to-sign",         │
     │     pairingRequired: true               │
     │   }                                     │
     │ }                                       │
     │                                         │
     │  4. Sign challenge                      │
     │ { type: "req", method: "pair", ... }    │
     │ ───────────────────────────────────────►│
     │                                         │
     │  5. Connection established              │
     │ ◄───────────────────────────────────────│
     │ {                                       │
     │   type: "event",                        │
     │   event: "hello-ok",                    │
     │   payload: {                            │
     │     presence: {...},                    │
     │     health: {...},                      │
     │     deviceToken: "..."                  │
     │   }                                     │
     │ }                                       │
```

### 2.3 消息格式

**请求:**
```typescript
interface RequestMessage {
  type: "req";
  id: string;           // 唯一请求 ID
  method: string;       // 方法名
  params?: unknown;     // 参数
  idempotencyKey?: string;  // 幂等键（用于重试）
}
```

**响应:**
```typescript
interface ResponseMessage {
  type: "res";
  id: string;           // 对应请求的 ID
  ok: boolean;          // 是否成功
  payload?: unknown;    // 成功时的数据
  error?: {             // 失败时的错误
    code: string;
    message: string;
    details?: unknown;
  };
}
```

**事件:**
```typescript
interface EventMessage {
  type: "event";
  event: string;        // 事件类型
  payload: unknown;     // 事件数据
  seq?: number;         // 序列号（用于排序）
  stateVersion?: number;  // 状态版本
}
```

## 3. 核心方法

### 3.1 Agent 方法

```typescript
// 启动 Agent 运行
interface AgentRequest {
  message: string;
  sessionKey?: string;
  sessionId?: string;
  agentId?: string;
  channel?: string;
  thinking?: string;
  verbose?: boolean;
  model?: string;
  provider?: string;
}

interface AgentResponse {
  runId: string;
  status: "accepted";
  acceptedAt: string;
}

// Agent 事件流
interface AgentEvent {
  stream: "lifecycle" | "assistant" | "tool" | "compaction";
  // lifecycle: { phase: "start" | "end" | "error", runId, ... }
  // assistant: { delta?, text?, reasoning? }
  // tool: { toolCallId, name, arguments?, result?, error? }
}
```

### 3.2 Status 方法

```typescript
// 获取 Gateway 状态
interface StatusRequest {
  probe?: boolean;  // 是否执行深度探测
}

interface StatusResponse {
  version: string;
  uptime: number;
  connections: number;
  health: {
    status: "healthy" | "degraded" | "unhealthy";
    checks: Record<string, HealthCheck>;
  };
  presence: PresenceState;
}
```

### 3.3 Send 方法

```typescript
// 发送消息到通道
interface SendRequest {
  channel: string;
  to: string;           // 目标（user:id 或 channel:id）
  message: string;
  media?: string;       // 媒体文件路径
  replyTo?: string;     // 回复的消息 ID
  silent?: boolean;     // 静默发送（不触发通知）
}

interface SendResponse {
  messageId: string;
  sentAt: string;
}
```

## 4. 连接管理

### 4.1 设备配对

```typescript
// src/gateway/pairing.ts

export interface PairedDevice {
  deviceId: string;
  deviceToken: string;
  platform: string;
  deviceFamily: string;
  publicKey: string;
  pairedAt: string;
  lastConnectedAt: string;
  role: "control" | "node";
  capabilities?: string[];
  metadata?: Record<string, unknown>;
}

export async function pairDevice(
  params: PairDeviceRequest
): Promise<PairedDevice> {
  // 1. 验证挑战签名
  const valid = verifyChallengeSignature(
    params.challenge,
    params.signature,
    params.publicKey
  );

  if (!valid) {
    throw new PairingError('Invalid challenge signature');
  }

  // 2. 创建设备记录
  const device: PairedDevice = {
    deviceId: generateDeviceId(),
    deviceToken: generateSecureToken(),
    platform: params.platform,
    deviceFamily: params.deviceFamily,
    publicKey: params.publicKey,
    pairedAt: new Date().toISOString(),
    lastConnectedAt: new Date().toISOString(),
    role: params.role,
    capabilities: params.capabilities,
  };

  // 3. 保存到存储
  await savePairedDevice(device);

  return device;
}
```

### 4.2 连接认证

```typescript
// src/gateway/auth.ts

export async function authenticateConnection(
  ws: WebSocket,
  connectRequest: ConnectRequest
): Promise<AuthResult> {
  // 1. 验证 Gateway Token（如果配置）
  if (config.gateway?.auth?.token) {
    if (connectRequest.auth?.token !== config.gateway.auth.token) {
      return { ok: false, error: 'Invalid gateway token' };
    }
  }

  // 2. 查找设备
  const device = await findPairedDevice(connectRequest.deviceId);

  if (!device) {
    // 新设备，需要配对
    return {
      ok: true,
      requirePairing: true,
      challenge: generateChallenge(),
    };
  }

  // 3. 验证设备令牌
  if (connectRequest.deviceToken !== device.deviceToken) {
    return { ok: false, error: 'Invalid device token' };
  }

  // 4. 更新最后连接时间
  await updateDeviceLastConnected(device.deviceId);

  return { ok: true, device };
}
```

## 5. 消息路由

### 5.1 路由表

```typescript
// src/gateway/router.ts

interface RouteTable {
  // 按 sessionKey 路由
  sessions: Map<string, WebSocket[]>;

  // 按 deviceId 路由
  devices: Map<string, WebSocket>;

  // 按 channel 路由
  channels: Map<string, Set<WebSocket>>;
}

export class MessageRouter {
  private routes: RouteTable = {
    sessions: new Map(),
    devices: new Map(),
    channels: new Map(),
  };

  register(ws: WebSocket, context: ConnectionContext): void {
    // 注册设备
    this.routes.devices.set(context.deviceId, ws);

    // 注册会话订阅
    for (const sessionKey of context.subscribedSessions ?? []) {
      if (!this.routes.sessions.has(sessionKey)) {
        this.routes.sessions.set(sessionKey, []);
      }
      this.routes.sessions.get(sessionKey)!.push(ws);
    }
  }

  unregister(ws: WebSocket, context: ConnectionContext): void {
    this.routes.devices.delete(context.deviceId);

    for (const sessionKey of context.subscribedSessions ?? []) {
      const subscribers = this.routes.sessions.get(sessionKey);
      if (subscribers) {
        const index = subscribers.indexOf(ws);
        if (index !== -1) {
          subscribers.splice(index, 1);
        }
      }
    }
  }

  // 广播到所有订阅了 session 的客户端
  broadcastToSession(sessionKey: string, message: EventMessage): void {
    const subscribers = this.routes.sessions.get(sessionKey) ?? [];
    for (const ws of subscribers) {
      if (ws.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify(message));
      }
    }
  }
}
```

### 5.2 事件广播

```typescript
// src/gateway/events.ts

export class GatewayEventEmitter {
  constructor(private router: MessageRouter) {}

  emitAgentEvent(sessionKey: string, event: AgentEvent): void {
    this.router.broadcastToSession(sessionKey, {
      type: "event",
      event: "agent",
      payload: event,
    });
  }

  emitPresenceUpdate(presence: PresenceState): void {
    this.router.broadcastToAll({
      type: "event",
      event: "presence",
      payload: presence,
    });
  }

  emitHeartbeat(): void {
    this.router.broadcastToAll({
      type: "event",
      event: "tick",
      payload: { timestamp: Date.now() },
    });
  }
}
```

## 6. 协议 Schema

### 6.1 TypeBox 定义

```typescript
// src/gateway/protocol/schema.ts

import { Type } from '@sinclair/typebox';

export const ConnectRequest = Type.Object({
  deviceId: Type.String(),
  platform: Type.String(),
  version: Type.String(),
  role: Type.Union([Type.Literal('control'), Type.Literal('node')]),
  capabilities: Type.Optional(Type.Array(Type.String())),
  commands: Type.Optional(Type.Array(Type.String())),
  auth: Type.Optional(Type.Object({
    token: Type.Optional(Type.String()),
    deviceToken: Type.Optional(Type.String()),
    signature: Type.Optional(Type.String()),
  })),
});

export const AgentRequest = Type.Object({
  message: Type.String(),
  sessionKey: Type.Optional(Type.String()),
  sessionId: Type.Optional(Type.String()),
  agentId: Type.Optional(Type.String()),
  channel: Type.Optional(Type.String()),
  thinking: Type.Optional(Type.String()),
  verbose: Type.Optional(Type.Boolean()),
  idempotencyKey: Type.Optional(Type.String()),
});

export const SendRequest = Type.Object({
  channel: Type.String(),
  to: Type.String(),
  message: Type.String(),
  media: Type.Optional(Type.String()),
  replyTo: Type.Optional(Type.String()),
  silent: Type.Optional(Type.Boolean()),
  idempotencyKey: Type.Optional(Type.String()),
});
```

### 6.2 验证中间件

```typescript
// src/gateway/protocol/validation.ts

import { Value } from '@sinclair/typebox/value';

export function createValidationMiddleware(schema: TSchema) {
  return (req: RequestMessage, res: ResponseMessage, next: () => void) => {
    const valid = Value.Check(schema, req.params);

    if (!valid) {
      const errors = [...Value.Errors(schema, req.params)];
      res.ok = false;
      res.error = {
        code: 'VALIDATION_ERROR',
        message: 'Request validation failed',
        details: errors,
      };
      return;
    }

    next();
  };
}
```

## 7. 关键代码文件

| 文件 | 描述 |
|------|------|
| `src/gateway/server.ts` | Gateway 服务器主入口 |
| `src/gateway/websocket.ts` | WebSocket 处理 |
| `src/gateway/pairing.ts` | 设备配对逻辑 |
| `src/gateway/auth.ts` | 连接认证 |
| `src/gateway/router.ts` | 消息路由 |
| `src/gateway/events.ts` | 事件广播 |
| `src/gateway/protocol/schema.ts` | 协议 Schema |
| `src/gateway/protocol/index.ts` | 协议导出 |
| `src/gateway/control-plane/` | 控制平面实现 |

## 8. 最佳实践

### 8.1 客户端连接

```typescript
// WebSocket 客户端示例

const ws = new WebSocket('ws://127.0.0.1:18789');

// 等待连接
ws.onopen = () => {
  // 发送 connect 请求
  ws.send(JSON.stringify({
    type: "req",
    id: "1",
    method: "connect",
    params: {
      deviceId: "my-device-id",
      platform: "web",
      version: "1.0.0",
      role: "control",
      auth: { deviceToken: "saved-token" },
    },
  }));
};

// 处理消息
ws.onmessage = (event) => {
  const msg = JSON.parse(event.data);

  if (msg.type === "event" && msg.event === "agent") {
    // 处理 Agent 事件
    console.log("Agent event:", msg.payload);
  }
};
```

### 8.2 重连策略

```typescript
function connectWithRetry(url: string, maxRetries = 5): WebSocket {
  let retries = 0;

  const tryConnect = (): WebSocket => {
    const ws = new WebSocket(url);

    ws.onclose = () => {
      if (retries < maxRetries) {
        retries++;
        setTimeout(tryConnect, 1000 * retries);
      }
    };

    return ws;
  };

  return tryConnect();
}
```
