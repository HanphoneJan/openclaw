# OpenClaw 通道系统详解

本文档深入剖析 OpenClaw 的通道系统，包括通道抽象、消息路由、多平台集成和回复管道。

## 1. 通道系统架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                     CHANNEL SYSTEM                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   CHANNEL ABSTRACTION                        │   │
│  │  ┌─────────────────────────────────────────────────────┐   │   │
│  │  │  interface Channel {                                │   │   │
│  │  │    send(message: OutboundMessage): Promise<void>   │   │   │
│  │  │    onMessage(handler: InboundHandler): void        │   │   │
│  │  │    describeActions(ctx): Promise<Action[]>         │   │   │
│  │  │    executeAction(action): Promise<Result>          │   │   │
│  │  │  }                                                  │   │   │
│  │  └─────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   CHANNEL IMPLEMENTATIONS                    │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │   │
│  │  │ Discord  │ │  Slack   │ │ Telegram │ │ WhatsApp │       │   │
│  │  │( bundled)│ │( bundled)│ │( bundled)│ │( bundled)│       │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │   │
│  │  │  Signal  │ │ iMessage │ │   IRC    │ │  Matrix  │       │   │
│  │  │( plugin) │ │( plugin) │ │( plugin) │ │( plugin) │       │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   MESSAGE ROUTER                             │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │   │
│  │  │   Inbound   │  │   Session   │  │   Outbound  │         │   │
│  │  │   Handler   │  │   Binding   │  │   Queue     │         │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 2. 核心抽象

### 2.1 Channel 接口

```typescript
// src/plugin-sdk/channel-contract.ts

export interface ChannelRegistration {
  id: string;
  name: string;
  configSchema: JSONSchema;
  initialize(config: unknown): Promise<ChannelInstance>;
}

export interface ChannelInstance {
  // 发送消息到通道
  send(message: OutboundMessage): Promise<SendResult>;

  // 监听入站消息
  onMessage(handler: (message: InboundMessage) => void): void;

  // 发现可执行动作
  describeActions(context: ActionContext): Promise<ActionDescription[]>;

  // 执行动作
  executeAction(action: ActionRequest): Promise<ActionResult>;

  // 生命周期
  start?(): Promise<void>;
  stop?(): Promise<void>;
}
```

### 2.2 消息类型

```typescript
// src/channels/plugins/types.core.ts

export interface InboundMessage {
  // 消息标识
  id: string;
  channelId: string;
  accountId?: string;

  // 发送者信息
  sender: {
    id: string;
    name?: string;
    username?: string;
  };

  // 内容
  text: string;
  attachments?: Attachment[];

  // 上下文
  threadId?: string;
  replyTo?: string;  // 回复的消息ID
  timestamp: string;

  // 原始数据（通道特定）
  raw: unknown;
}

export interface OutboundMessage {
  // 目标
  to: string;  // "user:id" 或 "channel:id"

  // 内容
  text?: string;
  attachments?: OutboundAttachment[];

  // 选项
  replyTo?: string;
  silent?: boolean;  // 静默发送（不触发通知）

  // 富媒体
  embeds?: Embed[];
  components?: Component[];
}
```

## 3. 通道实现

### 3.1 捆绑通道

| 通道 | 实现 | 核心依赖 |
|------|------|----------|
| **Discord** | `src/discord/` | discord.js |
| **Slack** | `src/slack/` | @slack/bolt |
| **Telegram** | `src/telegram/` | grammy |
| **WhatsApp** | `src/web/` | Baileys |
| **Signal** | `src/signal/` | libsignal-client |
| **iMessage** | `src/imessage/` | macOS Messages.framework |

### 3.2 插件通道

通过插件系统加载的通道：

| 通道 | 插件包 |
|------|--------|
| **Matrix** | `@openclaw/matrix` |
| **MSTeams** | `@openclaw/msteams` |
| **Zalo** | `@openclaw/zalo` |

### 3.3 Discord 实现示例

```typescript
// 简化示例：Discord 通道实现

import { Client, GatewayIntentBits } from 'discord.js';

class DiscordChannel implements ChannelInstance {
  private client: Client;

  async initialize(config: DiscordConfig): Promise<void> {
    this.client = new Client({
      intents: [
        GatewayIntentBits.Guilds,
        GatewayIntentBits.GuildMessages,
        GatewayIntentBits.MessageContent,
      ],
    });

    await this.client.login(config.token);
  }

  async send(message: OutboundMessage): Promise<SendResult> {
    const [targetType, targetId] = message.to.split(':');

    let channel;
    if (targetType === 'channel') {
      channel = await this.client.channels.fetch(targetId);
    } else if (targetType === 'user') {
      const user = await this.client.users.fetch(targetId);
      channel = await user.createDM();
    }

    if (!channel?.isTextBased()) {
      throw new Error('Invalid target');
    }

    const sent = await channel.send({
      content: message.text,
      reply: message.replyTo ? { messageReference: message.replyTo } : undefined,
      components: message.components,
    });

    return { messageId: sent.id, timestamp: sent.createdAt.toISOString() };
  }

  onMessage(handler: (msg: InboundMessage) => void): void {
    this.client.on('messageCreate', (discordMsg) => {
      if (discordMsg.author.bot) return;

      handler({
        id: discordMsg.id,
        channelId: discordMsg.channelId,
        sender: {
          id: discordMsg.author.id,
          name: discordMsg.author.displayName,
          username: discordMsg.author.username,
        },
        text: discordMsg.content,
        timestamp: discordMsg.createdAt.toISOString(),
        raw: discordMsg,
      });
    });
  }

  async describeActions(context: ActionContext): Promise<ActionDescription[]> {
    return [
      {
        name: 'send',
        description: 'Send a message',
        parameters: {
          to: { type: 'string', description: 'Target channel or user' },
          message: { type: 'string' },
        },
      },
      {
        name: 'react',
        description: 'Add a reaction',
        parameters: {
          messageId: { type: 'string' },
          emoji: { type: 'string' },
        },
      },
      {
        name: 'thread-create',
        description: 'Create a thread',
        parameters: {
          channelId: { type: 'string' },
          name: { type: 'string' },
        },
      },
    ];
  }
}
```

## 4. 消息路由

### 4.1 入站消息处理

```typescript
// src/channels/inbound-router.ts

export class InboundMessageRouter {
  constructor(
    private sessionResolver: SessionResolver,
    private agentCommand: AgentCommandService,
  ) {}

  async handleInboundMessage(message: InboundMessage): Promise<void> {
    // 1. 解析目标会话
    const sessionBinding = await this.sessionResolver.resolve({
      channelId: message.channelId,
      senderId: message.sender.id,
      threadId: message.threadId,
    });

    if (!sessionBinding) {
      // 未绑定，可能需要创建新会话或忽略
      return;
    }

    // 2. 检查是否应该处理
    if (!await this.shouldProcess(message, sessionBinding)) {
      return;
    }

    // 3. 构造 Agent 请求
    const agentRequest: AgentRequest = {
      message: message.text,
      sessionKey: sessionBinding.sessionKey,
      agentId: sessionBinding.agentId,
      channel: message.channelId,
      // 传递原始消息上下文
      context: {
        sender: message.sender,
        threadId: message.threadId,
        replyTo: message.replyTo,
      },
    };

    // 4. 提交到 Agent 队列
    await this.agentCommand.submit(agentRequest);
  }

  private async shouldProcess(
    message: InboundMessage,
    binding: SessionBinding,
  ): Promise<boolean> {
    // 检查消息是否以命令前缀开头
    if (message.text.startsWith('!')) {
      return false; // 忽略命令消息
    }

    // 检查是否需要 @mention（群组中）
    if (binding.requireMention && !message.text.includes(`<@bot_id>`)) {
      return false;
    }

    return true;
  }
}
```

### 4.2 会话绑定

```typescript
// src/channels/session-binding.ts

export interface SessionBinding {
  sessionKey: string;
  sessionId: string;
  agentId: string;
  channelId: string;
  accountId?: string;

  // 绑定配置
  requireMention?: boolean;
  autoReply?: boolean;
  queueMode: 'collect' | 'steer' | 'followup';
}

export class SessionBindingStore {
  private bindings = new Map<string, SessionBinding>();

  // 创建绑定键
  private makeKey(channelId: string, senderId?: string, threadId?: string): string {
    return [channelId, senderId, threadId].filter(Boolean).join(':');
  }

  async get(channelId: string, senderId?: string, threadId?: string): Promise<SessionBinding | undefined> {
    return this.bindings.get(this.makeKey(channelId, senderId, threadId));
  }

  async set(binding: SessionBinding): Promise<void> {
    this.bindings.set(
      this.makeKey(binding.channelId, undefined, binding.threadId),
      binding,
    );
  }

  // 查找绑定（从具体到通用）
  async find(channelId: string, senderId: string, threadId?: string): Promise<SessionBinding | undefined> {
    // 1. 精确匹配：channel + sender + thread
    let binding = await this.get(channelId, senderId, threadId);
    if (binding) return binding;

    // 2. 匹配：channel + thread（无特定 sender）
    if (threadId) {
      binding = await this.get(channelId, undefined, threadId);
      if (binding) return binding;
    }

    // 3. 匹配：channel（通用绑定）
    binding = await this.get(channelId);
    if (binding) return binding;

    return undefined;
  }
}
```

## 5. 共享消息工具

### 5.1 统一的 message 工具

OpenClaw 提供一个共享的 `message` 工具，所有通道操作通过它进行：

```typescript
// src/agents/tools/message.ts

export const messageTool: Tool = {
  name: 'message',
  description: 'Send messages to messaging channels',

  parameters: {
    type: 'object',
    properties: {
      action: {
        type: 'string',
        enum: ['send', 'edit', 'react', 'delete', 'read', 'thread-create'],
      },
      channel: { type: 'string' },
      to: { type: 'string' },
      message: { type: 'string' },
      messageId: { type: 'string' },
      emoji: { type: 'string' },
    },
    required: ['action', 'channel'],
  },

  async execute(params, context) {
    const channel = await getChannelInstance(params.channel);

    switch (params.action) {
      case 'send':
        return channel.send({
          to: params.to,
          text: params.message,
        });

      case 'react':
        return channel.executeAction({
          name: 'react',
          params: {
            messageId: params.messageId,
            emoji: params.emoji,
          },
        });

      case 'thread-create':
        return channel.executeAction({
          name: 'thread-create',
          params: {
            channelId: params.to,
            name: params.message,
          },
        });

      // ... 其他动作
    }
  },
};
```

### 5.2 动作发现

```typescript
// src/channels/plugins/types.adapters.ts

export interface ChannelMessageActionAdapter {
  // 统一发现接口
  describeMessageTool(context: ActionContext): Promise<{
    actions: MessageAction[];
    capabilities: ChannelCapabilities;
    schemaFragments: JSONSchema[];
  }>;
}

// 示例：Discord 适配器
class DiscordActionAdapter implements ChannelMessageActionAdapter {
  async describeMessageTool(context: ActionContext) {
    const isAdmin = await this.checkAdmin(context.accountId);
    const inThread = !!context.threadId;

    return {
      actions: [
        { name: 'send', description: 'Send a message' },
        { name: 'edit', description: 'Edit a message' },
        { name: 'react', description: 'Add a reaction' },
        ...(inThread ? [] : [{ name: 'thread-create', description: 'Create a thread' }]),
        ...(isAdmin ? [{ name: 'pin', description: 'Pin a message' }] : []),
      ],
      capabilities: {
        supportsEmbeds: true,
        supportsComponents: true,
        supportsPolls: true,
      },
      schemaFragments: [
        // Discord 特定的 schema 片段
        {
          silent: { type: 'boolean', description: 'Suppress notifications' },
        },
      ],
    };
  }
}
```

## 6. 回复管道

### 6.1 回复处理流程

```typescript
// src/channels/reply-pipeline.ts

export class ChannelReplyPipeline {
  async process(reply: AgentReply, context: ReplyContext): Promise<void> {
    // 1. 分块处理
    const chunks = await this.chunkReply(reply, context.channelCapabilities);

    for (const chunk of chunks) {
      // 2. 格式化
      const formatted = await this.formatChunk(chunk, context.channelId);

      // 3. 发送
      await this.sendChunk(formatted, context);

      // 4. 更新上下文（用于下一块）
      context.lastMessageId = formatted.messageId;
    }
  }

  private async chunkReply(
    reply: AgentReply,
    capabilities: ChannelCapabilities,
  ): Promise<ReplyChunk[]> {
    const maxLength = capabilities.maxMessageLength ?? 2000;

    if (reply.text.length <= maxLength) {
      return [{ type: 'text', content: reply.text }];
    }

    // 智能分块（按段落、句子边界）
    return smartChunk(reply.text, maxLength);
  }

  private async formatChunk(
    chunk: ReplyChunk,
    channelId: string,
  ): Promise<FormattedMessage> {
    const channel = await getChannelInstance(channelId);

    // 通道特定的格式化
    if (channelId.startsWith('discord:')) {
      return this.formatForDiscord(chunk);
    }

    return { text: chunk.content };
  }
}
```

### 6.2 块处理策略

```typescript
// src/plugin-sdk/reply-chunking.ts

export interface ChunkingStrategy {
  maxLength: number;
  respectParagraphs: boolean;
  respectCodeBlocks: boolean;
}

export function smartChunk(text: string, strategy: ChunkingStrategy): string[] {
  const chunks: string[] = [];
  let remaining = text;

  while (remaining.length > strategy.maxLength) {
    let cutPoint = strategy.maxLength;

    // 尝试在段落边界切割
    if (strategy.respectParagraphs) {
      const lastParagraphBreak = remaining.lastIndexOf('\n\n', strategy.maxLength);
      if (lastParagraphBreak > strategy.maxLength * 0.5) {
        cutPoint = lastParagraphBreak;
      }
    }

    // 尝试在句子边界切割
    const lastSentence = remaining.lastIndexOf('. ', cutPoint);
    if (lastSentence > cutPoint * 0.8) {
      cutPoint = lastSentence + 1;
    }

    chunks.push(remaining.slice(0, cutPoint).trim());
    remaining = remaining.slice(cutPoint).trim();
  }

  if (remaining) {
    chunks.push(remaining);
  }

  return chunks;
}
```

## 7. 队列模式

### 7.1 三种队列模式

```typescript
// src/channels/queue-modes.ts

export type QueueMode = 'collect' | 'steer' | 'followup';

// 1. Collect 模式（收集）
// - 收集多条消息
// - 批量处理或总结
export const QUEUE_MODE_COLLECT: QueueModeConfig = {
  mode: 'collect',
  batchWindowMs: 5000,  // 5秒收集窗口
  maxBatchSize: 10,
  separator: '\n---\n',
};

// 2. Steer 模式（引导）
// - Agent 主动引导对话
// - 发送多个消息
export const QUEUE_MODE_STEER: QueueModeConfig = {
  mode: 'steer',
  allowMultipleSends: true,
  pauseBetweenSendsMs: 1000,
};

// 3. Followup 模式（跟进）
// - 被动响应
// - 一问一答
export const QUEUE_MODE_FOLLOWUP: QueueModeConfig = {
  mode: 'followup',
  requireExplicitMention: true,
  autoReply: false,
};
```

### 7.2 队列实现

```typescript
// src/channels/message-queue.ts

export class ChannelMessageQueue {
  private queues = new Map<string, Message[]>();
  private timers = new Map<string, NodeJS.Timeout>();

  async enqueue(
    channelId: string,
    message: InboundMessage,
    config: QueueModeConfig,
    processor: (messages: Message[]) => Promise<void>,
  ): Promise<void> {
    // 获取或创建队列
    if (!this.queues.has(channelId)) {
      this.queues.set(channelId, []);
    }

    const queue = this.queues.get(channelId)!;
    queue.push(message);

    // 清除现有定时器
    if (this.timers.has(channelId)) {
      clearTimeout(this.timers.get(channelId)!);
    }

    // 立即处理或设置定时器
    if (queue.length >= config.maxBatchSize!) {
      await this.flush(channelId, processor);
    } else {
      const timer = setTimeout(
        () => this.flush(channelId, processor),
        config.batchWindowMs,
      );
      this.timers.set(channelId, timer);
    }
  }

  private async flush(
    channelId: string,
    processor: (messages: Message[]) => Promise<void>,
  ): Promise<void> {
    const queue = this.queues.get(channelId);
    if (!queue || queue.length === 0) return;

    // 清空队列
    this.queues.set(channelId, []);
    this.timers.delete(channelId);

    // 处理消息
    await processor(queue);
  }
}
```

## 8. 通道配置

### 8.1 配置结构

```json5
{
  channels: {
    discord: {
      // 认证
      token: '${DISCORD_BOT_TOKEN}',

      // 行为配置
      actions: {
        roles: true,
        moderation: false,
        presence: false,
        channels: false,
      },

      // 绑定配置
      bindings: [
        {
          channelId: '123456789',
          agentId: 'support-bot',
          queueMode: 'collect',
          requireMention: true,
        },
      ],
    },

    slack: {
      token: '${SLACK_BOT_TOKEN}',
      signingSecret: '${SLACK_SIGNING_SECRET}',

      socketMode: true,
      appToken: '${SLACK_APP_TOKEN}',
    },

    telegram: {
      token: '${TELEGRAM_BOT_TOKEN}',
      polling: true,
    },
  },
}
```

### 8.2 动态配置

```typescript
// src/channels/channel-config.ts

export async function loadChannelConfig(
  channelId: string,
  config: OpenClawConfig,
): Promise<ChannelConfig> {
  const channelType = channelId.split(':')[0];
  const baseConfig = config.channels?.[channelType];

  if (!baseConfig) {
    throw new ChannelNotConfiguredError(channelType);
  }

  // 合并特定账户配置
  const accountId = extractAccountId(channelId);
  const accountConfig = baseConfig.accounts?.[accountId];

  return {
    ...baseConfig,
    ...accountConfig,
    channelId,
  };
}
```

## 9. 关键代码文件

| 文件 | 描述 |
|------|------|
| `src/channels/plugins/types.core.ts` | 核心通道类型 |
| `src/channels/plugins/types.plugin.ts` | 插件通道类型 |
| `src/channels/plugins/types.adapters.ts` | 适配器类型 |
| `src/plugin-sdk/channel-contract.ts` | Channel SDK 接口 |
| `src/plugin-sdk/channel-runtime.ts` | Channel 运行时 |
| `src/channels/inbound-router.ts` | 入站消息路由 |
| `src/channels/session-binding.ts` | 会话绑定管理 |
| `src/channels/reply-pipeline.ts` | 回复管道 |
| `src/plugin-sdk/reply-chunking.ts` | 回复分块 |
| `src/discord/`, `src/slack/`, etc. | 具体通道实现 |

## 10. 最佳实践

### 10.1 通道实现指南

1. **错误处理**：网络错误应该重试，认证错误应该标记
2. **速率限制**：遵守平台限制，实现退避策略
3. **消息确认**：异步平台需要确认机制
4. **资源清理**：stop() 方法释放所有资源

### 10.2 示例实现模板

```typescript
class MyChannel implements ChannelInstance {
  private ws?: WebSocket;
  private messageHandlers: Array<(msg: InboundMessage) => void> = [];

  async initialize(config: MyConfig): Promise<void> {
    this.ws = new WebSocket(config.wsUrl);

    this.ws.on('message', (data) => {
      const parsed = JSON.parse(data);
      this.handleIncomingMessage(parsed);
    });

    this.ws.on('error', (err) => {
      log.error('WebSocket error:', err);
    });

    // 等待连接
    await new Promise((resolve, reject) => {
      this.ws!.once('open', resolve);
      this.ws!.once('error', reject);
    });
  }

  async send(message: OutboundMessage): Promise<SendResult> {
    if (!this.ws || this.ws.readyState !== WebSocket.OPEN) {
      throw new Error('Channel not connected');
    }

    const payload = this.buildPayload(message);
    this.ws.send(JSON.stringify(payload));

    // 等待确认（如果平台支持）
    const confirmation = await this.waitForConfirmation();

    return {
      messageId: confirmation.id,
      timestamp: confirmation.timestamp,
    };
  }

  onMessage(handler: (msg: InboundMessage) => void): void {
    this.messageHandlers.push(handler);
  }

  async stop(): Promise<void> {
    this.ws?.close();
    this.messageHandlers = [];
  }
}
```
