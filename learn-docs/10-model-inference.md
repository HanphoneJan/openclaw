# OpenClaw 模型与推理系统详解

本文档深入剖析 OpenClaw 的模型推理系统，包括提供者注册、模型选择、推理循环和能力管理。

## 1. 模型系统架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MODEL INFERENCE SYSTEM                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   PROVIDER REGISTRY                          │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐            │   │
│  │  │  Anthropic  │ │   OpenAI    │ │   Google    │            │   │
│  │  │  Provider   │ │  Provider   │ │  Provider   │            │   │
│  │  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘            │   │
│  │         └───────────────┼───────────────┘                   │   │
│  │                         │                                    │   │
│  │                         ▼                                    │   │
│  │              ┌─────────────────────┐                        │   │
│  │              │   Model Catalog     │                        │   │
│  │              │   (Available Models)│                        │   │
│  │              └─────────────────────┘                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   MODEL SELECTION                            │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │   │
│  │  │   Config    │  │   Override  │  │   Fallback  │         │   │
│  │  │   Default   │  │   (Session) │  │   Chain     │         │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   INFERENCE LOOP                             │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │   │
│  │  │   Build     │  │   Provider  │  │   Stream    │         │   │
│  │  │   Prompt    │  │   Inference │  │   Response  │         │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 2. 提供者注册

### 2.1 Provider 接口

```typescript
// src/plugin-sdk/provider-entry.ts

export interface ProviderRegistration {
  id: string;
  name: string;
  models: ModelDefinition[];
  defaultModel?: string;

  // 推理实现
  infer(params: InferenceParams): AsyncIterable<InferenceChunk>;

  // 认证
  auth?: {
    type: 'api_key' | 'oauth';
    resolveAuth(params: AuthParams): Promise<AuthInfo>;
  };

  // 模型能力检测
  supportsModel?(modelId: string): boolean;

  // 思考级别支持
  supportsThinking?(modelId: string): boolean;
}

export interface ModelDefinition {
  id: string;
  name: string;
  contextWindow: number;
  maxOutputTokens?: number;
  supportsThinking?: boolean;
  supportsVision?: boolean;
  supportsTools?: boolean;
}

export interface InferenceParams {
  model: string;
  messages: Message[];
  tools?: ToolDefinition[];
  temperature?: number;
  maxTokens?: number;
  thinking?: ThinkingConfig;
  signal?: AbortSignal;
}

export interface InferenceChunk {
  type: 'text' | 'thinking' | 'tool_call' | 'error';
  content?: string;
  toolCall?: {
    id: string;
    name: string;
    arguments: string;
  };
  error?: Error;
}
```

### 2.2 注册示例

```typescript
// 示例：Anthropic Provider 注册

import type { PluginApi } from 'openclaw/plugin-sdk';

export default function register(api: PluginApi) {
  api.registerProvider({
    id: 'anthropic',
    name: 'Anthropic',

    models: [
      {
        id: 'claude-opus-4-6',
        name: 'Claude Opus 4.6',
        contextWindow: 200000,
        maxOutputTokens: 4096,
        supportsThinking: true,
        supportsVision: true,
        supportsTools: true,
      },
      {
        id: 'claude-sonnet-4-6',
        name: 'Claude Sonnet 4.6',
        contextWindow: 200000,
        maxOutputTokens: 4096,
        supportsThinking: true,
        supportsVision: true,
        supportsTools: true,
      },
    ],

    defaultModel: 'claude-sonnet-4-6',

    async infer(params) {
      const client = createAnthropicClient(params.auth);

      const stream = await client.messages.create({
        model: params.model,
        messages: convertMessages(params.messages),
        tools: params.tools,
        max_tokens: params.maxTokens ?? 4096,
        stream: true,
      });

      for await (const chunk of stream) {
        yield convertChunk(chunk);
      }
    },
  });
}
```

## 3. 模型目录

### 3.1 目录结构

```typescript
// src/agents/model-catalog.ts

export interface ModelCatalog {
  version: string;
  generatedAt: string;
  providers: ProviderEntry[];
}

export interface ProviderEntry {
  id: string;
  name: string;
  models: ModelEntry[];
}

export interface ModelEntry {
  id: string;
  name: string;
  provider: string;
  contextWindow: number;
  capabilities: ModelCapabilities;
  pricing?: {
    input: number;
    output: number;
    unit: 'per_1m_tokens';
  };
}

export interface ModelCapabilities {
  thinking: boolean;
  vision: boolean;
  tools: boolean;
  streaming: boolean;
}
```

### 3.2 加载目录

```typescript
// src/agents/model-catalog.ts

export async function loadModelCatalog(
  config: OpenClawConfig
): Promise<ModelCatalog> {
  // 1. 从注册表获取所有提供者
  const providers = getRegisteredProviders();

  // 2. 构建目录
  const catalog: ModelCatalog = {
    version: getVersion(),
    generatedAt: new Date().toISOString(),
    providers: [],
  };

  for (const provider of providers) {
    const entry: ProviderEntry = {
      id: provider.id,
      name: provider.name,
      models: provider.models.map(m => ({
        id: m.id,
        name: m.name,
        provider: provider.id,
        contextWindow: m.contextWindow,
        capabilities: {
          thinking: m.supportsThinking ?? false,
          vision: m.supportsVision ?? false,
          tools: m.supportsTools ?? false,
          streaming: true,
        },
      })),
    };

    catalog.providers.push(entry);
  }

  return catalog;
}
```

## 4. 模型选择

### 4.1 选择流程

```typescript
// src/agents/model-selection.ts

export interface ModelSelectionParams {
  config: OpenClawConfig;
  session?: SessionEntry;
  preferredProvider?: string;
  preferredModel?: string;
  requireThinking?: boolean;
  requireVision?: boolean;
}

export function resolveConfiguredModelRef(
  params: ModelSelectionParams
): ModelRef {
  // 1. 检查会话覆盖
  if (params.session?.modelOverride) {
    return parseModelRef(params.session.modelOverride);
  }

  // 2. 检查显式偏好
  if (params.preferredModel) {
    return parseModelRef(params.preferredModel, params.preferredProvider);
  }

  // 3. 检查 Agent 配置
  if (params.session?.agentId) {
    const agentConfig = params.config.agents?.[params.session.agentId];
    if (agentConfig?.model) {
      return parseModelRef(agentConfig.model, agentConfig.provider);
    }
  }

  // 4. 使用默认配置
  const defaults = params.config.agents?.defaults;
  return {
    provider: defaults?.provider ?? 'anthropic',
    model: defaults?.model ?? 'claude-sonnet-4-6',
  };
}

export function parseModelRef(
  ref: string,
  defaultProvider?: string
): ModelRef {
  // 支持格式："provider:model" 或 "model"
  const parts = ref.split(':');

  if (parts.length === 2) {
    return { provider: parts[0], model: parts[1] };
  }

  return { provider: defaultProvider ?? 'anthropic', model: ref };
}
```

### 4.2 思考级别

```typescript
// src/auto-reply/thinking.ts

export type ThinkingLevel = 'low' | 'medium' | 'high' | 'extreme';

export interface ThinkingConfig {
  enabled: boolean;
  level: ThinkingLevel;
  budgetTokens?: number;
}

export function normalizeThinkLevel(level?: string): ThinkingLevel {
  switch (level) {
    case 'low':
    case 'minimal':
      return 'low';
    case 'medium':
    case 'normal':
      return 'medium';
    case 'high':
    case 'maximum':
      return 'high';
    case 'extreme':
      return 'extreme';
    default:
      return 'medium';
  }
}

export function resolveThinkingDefault(
  model: string,
  config?: OpenClawConfig
): ThinkingConfig {
  const defaultLevel = config?.agents?.defaults?.thinking ?? 'medium';

  // 检查模型是否支持思考
  const supportsThinking = checkModelSupportsThinking(model);

  if (!supportsThinking) {
    return { enabled: false, level: 'medium' };
  }

  return {
    enabled: true,
    level: normalizeThinkLevel(defaultLevel),
  };
}
```

## 5. 推理执行

### 5.1 推理循环

```typescript
// src/agents/inference.ts

export async function runInference(
  params: InferenceParams
): Promise<InferenceResult> {
  // 1. 解析模型引用
  const modelRef = parseModelRef(params.model);

  // 2. 获取提供者
  const provider = getProvider(modelRef.provider);
  if (!provider) {
    throw new ProviderNotFoundError(modelRef.provider);
  }

  // 3. 解析认证
  const auth = await resolveAuthForProvider(modelRef.provider, params.config);

  // 4. 运行推理
  const chunks: InferenceChunk[] = [];
  let toolCalls: ToolCall[] = [];
  let text = '';

  for await (const chunk of provider.infer({
    ...params,
    model: modelRef.model,
    auth,
  })) {
    chunks.push(chunk);

    switch (chunk.type) {
      case 'text':
        text += chunk.content ?? '';
        break;
      case 'tool_call':
        if (chunk.toolCall) {
          toolCalls.push(chunk.toolCall);
        }
        break;
      case 'error':
        throw chunk.error ?? new Error('Inference error');
    }

    // 流式回调
    if (params.onChunk) {
      params.onChunk(chunk);
    }
  }

  return {
    text,
    toolCalls,
    usage: estimateUsage(chunks),
  };
}
```

### 5.2 工具调用处理

```typescript
// src/agents/tools.ts

export async function handleToolCalls(
  toolCalls: ToolCall[],
  context: ToolContext
): Promise<ToolResult[]> {
  const results: ToolResult[] = [];

  for (const toolCall of toolCalls) {
    const tool = findTool(toolCall.name);

    if (!tool) {
      results.push({
        toolCallId: toolCall.id,
        error: `Tool not found: ${toolCall.name}`,
      });
      continue;
    }

    try {
      // 执行工具前钩子
      const hookResult = await runPluginHooks('before_tool_call', {
        toolName: tool.name,
        params: toolCall.arguments,
      });

      if (hookResult.block) {
        results.push({
          toolCallId: toolCall.id,
          error: `Tool call blocked: ${hookResult.reason}`,
        });
        continue;
      }

      // 执行工具
      const result = await tool.execute(toolCall.arguments, context);

      // 执行工具后钩子
      await runPluginHooks('after_tool_call', {
        toolName: tool.name,
        params: toolCall.arguments,
        result,
      });

      results.push({
        toolCallId: toolCall.id,
        result,
      });
    } catch (err) {
      results.push({
        toolCallId: toolCall.id,
        error: err instanceof Error ? err.message : String(err),
      });
    }
  }

  return results;
}
```

## 6. 模型回退

### 6.1 回退链

```typescript
// src/agents/model-fallback.ts

export interface FallbackChain {
  currentIndex: number;
  models: ModelRef[];
}

export function buildFallbackChain(
  primary: ModelRef,
  config: OpenClawConfig
): FallbackChain {
  const chain: ModelRef[] = [primary];

  // 添加全局回退配置
  const fallbackRefs = config.agents?.fallbacks ?? [
    { provider: 'anthropic', model: 'claude-sonnet-4-6' },
    { provider: 'openai', model: 'gpt-5.4' },
  ];

  for (const fallback of fallbackRefs) {
    // 避免重复
    if (fallback.provider !== primary.provider ||
        fallback.model !== primary.model) {
      chain.push(fallback);
    }
  }

  return {
    currentIndex: 0,
    models: chain,
  };
}

export async function runWithModelFallback<T>(
  chain: FallbackChain,
  operation: (model: ModelRef) => Promise<T>,
  onFallback?: (from: ModelRef, to: ModelRef, reason: string) => void
): Promise<T> {
  const errors: Error[] = [];

  for (let i = chain.currentIndex; i < chain.models.length; i++) {
    const model = chain.models[i];

    try {
      return await operation(model);
    } catch (err) {
      errors.push(err as Error);

      // 通知回退
      if (i < chain.models.length - 1 && onFallback) {
        onFallback(model, chain.models[i + 1], (err as Error).message);
      }
    }
  }

  throw new FallbackExhaustedError(errors);
}
```

## 7. 能力检测

### 7.1 能力查询

```typescript
// src/plugins/capability-runtime.ts

export function checkModelCapability(
  providerId: string,
  modelId: string,
  capability: keyof ModelCapabilities
): boolean {
  const provider = getProvider(providerId);
  if (!provider) return false;

  const model = provider.models.find(m => m.id === modelId);
  if (!model) return false;

  switch (capability) {
    case 'thinking':
      return model.supportsThinking ?? false;
    case 'vision':
      return model.supportsVision ?? false;
    case 'tools':
      return model.supportsTools ?? false;
    default:
      return false;
  }
}

export function getModelsWithCapability(
  capability: keyof ModelCapabilities
): Array<{ provider: string; model: string }> {
  const results = [];

  for (const provider of getAllProviders()) {
    for (const model of provider.models) {
      if (checkModelCapability(provider.id, model.id, capability)) {
        results.push({ provider: provider.id, model: model.id });
      }
    }
  }

  return results;
}
```

## 8. 关键代码文件

| 文件 | 描述 |
|------|------|
| `src/plugin-sdk/provider-entry.ts` | Provider 注册接口 |
| `src/plugin-sdk/provider-auth.ts` | 提供者认证 |
| `src/agents/model-catalog.ts` | 模型目录 |
| `src/agents/model-selection.ts` | 模型选择 |
| `src/agents/model-fallback.ts` | 模型回退 |
| `src/agents/inference.ts` | 推理执行 |
| `src/auto-reply/thinking.ts` | 思考级别 |
| `src/plugins/capability-runtime.ts` | 能力运行时 |

## 9. 最佳实践

### 9.1 提供者实现指南

```typescript
// 1. 优雅处理错误
async infer(params) {
  try {
    const response = await fetch(...);

    if (!response.ok) {
      const error = await response.json();

      // 区分错误类型
      if (error.error?.type === 'rate_limit') {
        throw new RateLimitError(error.error.message);
      }

      throw new ProviderError(error.error?.message ?? 'Unknown error');
    }

    yield* streamResponse(response);
  } catch (err) {
    if (err instanceof ProviderError) {
      throw err;
    }
    throw new ProviderError(`Request failed: ${err.message}`);
  }
}

// 2. 支持流式响应
async function* streamResponse(response: Response) {
  const reader = response.body?.getReader();
  if (!reader) return;

  try {
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      yield parseChunk(value);
    }
  } finally {
    reader.releaseLock();
  }
}

// 3. 正确处理信号
async infer(params) {
  const abortController = new AbortController();

  if (params.signal) {
    params.signal.addEventListener('abort', () => {
      abortController.abort();
    });
  }

  const response = await fetch(url, {
    signal: abortController.signal,
  });
}
```
