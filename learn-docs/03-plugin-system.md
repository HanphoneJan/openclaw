# OpenClaw 插件系统详解

本文档深入剖析 OpenClaw 的插件系统，包括发现机制、加载流程、能力注册和运行时行为。

## 1. 插件系统架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                     PLUGIN SYSTEM                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    DISCOVERY LAYER                           │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │   │
│  │  │ Bundled  │  │ Workspace│  │  Global  │  │  Plugin  │     │   │
│  │  │  Plugins │  │  Plugins │  │  Plugins │  │  Hooks   │     │   │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘     │   │
│  │       └─────────────┴─────────────┴─────────────┘            │   │
│  │                         │                                    │   │
│  │                         ▼                                    │   │
│  │              ┌─────────────────────┐                        │   │
│  │              │  Plugin Candidates  │                        │   │
│  │              └─────────────────────┘                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                  VALIDATION LAYER                            │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │   │
│  │  │ Manifest │  │ Security │  │  Enable  │  │   Shape  │     │   │
│  │  │  Check   │  │  Check   │  │  Check   │  │  Detect  │     │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   RUNTIME LAYER                              │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │   │
│  │  │  Registry│  │ Capability│  │  Hooks   │  │   CLI    │     │   │
│  │  │          │  │  Registration│  │          │  │ Commands │     │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 2. 插件发现机制

### 2.1 发现来源

OpenClaw 从四个来源发现插件，按优先级排序：

| 来源 | 路径 | 优先级 | 说明 |
|------|------|--------|------|
| **Bundled** | `dist/plugins/bundled/` | 1 | 随 OpenClaw 一起发布的插件 |
| **Workspace** | `<workspace>/plugins/` | 2 | 工作区本地插件 |
| **Global** | `~/.openclaw/plugins/` | 3 | 用户全局安装的插件 |
| **Plugin Hooks** | 插件内部声明 | 4 | 其他插件声明的钩子 |

### 2.2 发现流程

```typescript
// src/plugins/discovery.ts

export interface PluginCandidate {
  idHint: string;
  source: string;
  rootDir: string;
  origin: PluginOrigin;  // 'bundled' | 'workspace' | 'global' | 'plugin'
  format?: PluginFormat;
  bundledManifest?: PluginManifest;
  packageManifest?: OpenClawPackageManifest;
}

export async function discoverPlugins(params: {
  workspaceDir?: string;
  extraPaths?: string[];
}): Promise<PluginDiscoveryResult> {
  const candidates: PluginCandidate[] = [];
  const diagnostics: PluginDiagnostic[] = [];

  // 1. 发现捆绑插件
  const bundledRoot = resolveBundledPluginsRoot();
  for (const dir of await scanPluginDirs(bundledRoot)) {
    candidates.push(await readBundledPluginCandidate(dir));
  }

  // 2. 发现全局插件
  const globalRoot = resolveGlobalPluginsRoot();
  for (const dir of await scanPluginDirs(globalRoot)) {
    candidates.push(await readGlobalPluginCandidate(dir));
  }

  // 3. 发现工作区插件
  if (params.workspaceDir) {
    const workspaceRoot = path.join(params.workspaceDir, 'plugins');
    for (const dir of await scanPluginDirs(workspaceRoot)) {
      candidates.push(await readWorkspacePluginCandidate(dir));
    }
  }

  // 4. 安全检查
  for (const candidate of candidates) {
    const issues = checkSourceEscapesRoot(candidate);
    if (issues.length > 0) {
      diagnostics.push(...issues);
    }
  }

  return { candidates, diagnostics };
}
```

### 2.3 安全边界检查

```typescript
// src/plugins/path-safety.ts

export function checkSourceEscapesRoot(candidate: PluginCandidate): CandidateBlockIssue[] {
  const issues: CandidateBlockIssue[] = [];

  // 检查路径是否越界
  const sourceRealPath = safeRealpathSync(candidate.source);
  const rootRealPath = safeRealpathSync(candidate.rootDir);

  if (!isPathInside(rootRealPath, sourceRealPath)) {
    issues.push({
      reason: 'source_escapes_root',
      sourcePath: candidate.source,
      rootPath: candidate.rootDir,
      targetPath: candidate.source,
    });
  }

  // 检查文件权限
  const stat = safeStatSync(candidate.source);
  if (stat && isWorldWritable(stat.mode)) {
    issues.push({
      reason: 'path_world_writable',
      sourcePath: candidate.source,
      modeBits: stat.mode,
    });
  }

  return issues;
}
```

## 3. 插件清单

### 3.1 openclaw.plugin.json

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "version": "1.0.0",
  "description": "A sample plugin",
  "entry": {
    "module": "./dist/index.js",
    "register": "register"
  },
  "capabilities": [
    "provider",
    "channel"
  ],
  "config": {
    "schema": {
      "apiKey": { "type": "string" }
    }
  },
  "hooks": {
    "before_agent_start": "./hooks/before-start.js"
  }
}
```

### 3.2 清单解析

```typescript
// src/plugins/manifest.ts

export interface PluginManifest {
  id: string;
  name: string;
  version: string;
  description?: string;
  entry?: {
    module: string;
    register?: string;
  };
  capabilities?: CapabilityType[];
  config?: {
    schema: JSONSchema;
  };
}

export async function loadPluginManifest(
  pluginDir: string
): Promise<PluginManifest | undefined> {
  const manifestPath = path.join(pluginDir, 'openclaw.plugin.json');

  try {
    const content = await fs.readFile(manifestPath, 'utf-8');
    const manifest = JSON.parse(content) as PluginManifest;

    // 验证必需字段
    if (!manifest.id) {
      throw new Error('Plugin manifest missing required field: id');
    }

    return manifest;
  } catch (err) {
    if ((err as NodeJS.ErrnoException).code === 'ENOENT') {
      return undefined;
    }
    throw err;
  }
}
```

## 4. 插件加载流程

### 4.1 加载架构

```
┌─────────────────────────────────────────────────────────────┐
│                     PLUGIN LOADING                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐                                        │
│  │ 1. Load Manifest│                                        │
│  │    (openclaw.plugin.json)                               │
│  └────────┬────────┘                                        │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                        │
│  │ 2. Check Enabled│                                        │
│  │    (config.plugins[id].enabled)                         │
│  └────────┬────────┘                                        │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                        │
│  │ 3. Import Module│                                        │
│  │    (jiti import)                                        │
│  └────────┬────────┘                                        │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                        │
│  │ 4. Call Register│                                        │
│  │    (api.registerXXX)                                    │
│  └────────┬────────┘                                        │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                        │
│  │ 5. Add to Registry│                                      │
│  │    (PluginRegistry)                                      │
│  └─────────────────┘                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 加载实现

```typescript
// src/plugins/loader.ts

export interface PluginLoadResult {
  pluginId: string;
  success: boolean;
  error?: Error;
  capabilities?: RegisteredCapability[];
}

export async function loadPlugin(
  candidate: PluginCandidate,
  api: PluginApi
): Promise<PluginLoadResult> {
  try {
    // 1. 加载清单
    const manifest = await loadPluginManifest(candidate.rootDir);
    if (!manifest) {
      return { pluginId: candidate.idHint, success: false, error: new Error('No manifest found') };
    }

    // 2. 检查是否启用
    if (!isPluginEnabled(manifest.id)) {
      return { pluginId: manifest.id, success: false, error: new Error('Plugin disabled') };
    }

    // 3. 动态导入模块
    const entryPath = path.join(candidate.rootDir, manifest.entry?.module ?? 'index.js');
    const importUrl = buildImportUrl(entryPath, candidate.origin);
    const mod = await import(importUrl);

    // 4. 获取注册函数
    const registerFnName = manifest.entry?.register ?? 'default';
    const registerFn = mod[registerFnName];

    if (typeof registerFn !== 'function') {
      throw new Error(`Plugin ${manifest.id} does not export a register function`);
    }

    // 5. 调用注册函数
    await registerFn(api);

    // 6. 获取注册的能力
    const capabilities = getRegisteredCapabilities(manifest.id);

    return {
      pluginId: manifest.id,
      success: true,
      capabilities,
    };
  } catch (err) {
    return {
      pluginId: candidate.idHint,
      success: false,
      error: err instanceof Error ? err : new Error(String(err)),
    };
  }
}
```

## 5. 能力注册系统

### 5.1 能力类型

```typescript
// src/plugins/types.ts

export type CapabilityType =
  | 'provider'           // 文本推理提供者
  | 'cli_backend'        // CLI 推理后端
  | 'speech'             // 语音合成
  | 'media_understanding'// 媒体理解
  | 'image_generation'   // 图像生成
  | 'web_search'         // 网络搜索
  | 'channel';           // 消息通道
```

### 5.2 Provider 能力注册

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
}

export interface PluginApi {
  registerProvider(provider: ProviderRegistration): void;
  registerCliBackend(backend: CliBackendRegistration): void;
  registerSpeechProvider(provider: SpeechProviderRegistration): void;
  registerChannel(channel: ChannelRegistration): void;
  // ...
}
```

### 5.3 通道能力注册

```typescript
// src/plugin-sdk/channel-contract.ts

export interface ChannelRegistration {
  id: string;
  name: string;

  // 通道配置
  configSchema: JSONSchema;

  // 初始化
  initialize(config: unknown): Promise<ChannelInstance>;
}

export interface ChannelInstance {
  // 发送消息
  send(message: OutboundMessage): Promise<void>;

  // 监听入站消息
  onMessage(handler: (message: InboundMessage) => void): void;

  // 通道操作
  describeActions(context: ActionContext): Promise<ActionDescription[]>;
  executeAction(action: ActionRequest): Promise<ActionResult>;
}
```

## 6. 插件运行时 API

### 6.1 注册 API

```typescript
// src/plugin-sdk/core.ts

export interface PluginApi {
  // 基础信息
  readonly pluginId: string;
  readonly pluginDir: string;

  // 日志
  log: {
    info(msg: string): void;
    warn(msg: string): void;
    error(msg: string): void;
    debug(msg: string): void;
  };

  // 能力注册
  registerProvider(provider: ProviderRegistration): void;
  registerCliBackend(backend: CliBackendRegistration): void;
  registerSpeechProvider(provider: SpeechProviderRegistration): void;
  registerMediaUnderstandingProvider(provider: MediaUnderstandingRegistration): void;
  registerImageGenerationProvider(provider: ImageGenerationRegistration): void;
  registerWebSearchProvider(provider: WebSearchRegistration): void;
  registerChannel(channel: ChannelRegistration): void;

  // 钩子注册
  registerHook(event: string, handler: HookHandler): void;

  // 工具注册
  registerTool(tool: ToolDefinition): void;

  // CLI 命令注册
  registerCliCommand(command: CliCommandDefinition): void;

  // HTTP 路由注册
  registerRoute(route: RouteDefinition): void;
}
```

### 6.2 插件入口示例

```typescript
// 示例插件入口

import type { PluginApi } from 'openclaw/plugin-sdk';

export default function register(api: PluginApi) {
  api.log.info(`Registering plugin: ${api.pluginId}`);

  // 注册提供者
  api.registerProvider({
    id: 'my-provider',
    name: 'My Provider',
    models: [
      { id: 'model-1', name: 'Model 1' },
      { id: 'model-2', name: 'Model 2' },
    ],
    async infer(params) {
      // 实现推理
      yield { type: 'text', content: 'Hello' };
    },
  });

  // 注册钩子
  api.registerHook('before_agent_start', async (event) => {
    api.log.info('Agent starting...');
  });

  // 注册工具
  api.registerTool({
    name: 'my_tool',
    description: 'My custom tool',
    parameters: {
      type: 'object',
      properties: {
        input: { type: 'string' },
      },
    },
    async execute(params, context) {
      return { result: `Processed: ${params.input}` };
    },
  });
}
```

## 7. 插件形状检测

### 7.1 形状分类

```typescript
// src/plugins/bundled-plugin-metadata.ts

export type PluginShape =
  | 'plain-capability'   // 单一能力
  | 'hybrid-capability'  // 多种能力
  | 'hook-only'          // 仅钩子
  | 'non-capability';    // 工具/命令/服务，无能力

export function detectPluginShape(
  plugin: LoadedPlugin
): PluginShape {
  const capabilityCount = plugin.capabilities?.length ?? 0;
  const hasHooks = plugin.hooks && plugin.hooks.length > 0;
  const hasTools = plugin.tools && plugin.tools.length > 0;
  const hasCommands = plugin.commands && plugin.commands.length > 0;

  if (capabilityCount === 0) {
    if (hasHooks && !hasTools && !hasCommands) {
      return 'hook-only';
    }
    return 'non-capability';
  }

  if (capabilityCount === 1) {
    return 'plain-capability';
  }

  return 'hybrid-capability';
}
```

### 7.2 示例插件形状

| 插件 | 形状 | 能力 |
|------|------|------|
| `openai` | hybrid-capability | provider, speech, media_understanding, image_generation |
| `anthropic` | plain-capability | provider |
| `discord` | plain-capability | channel |
| `elevenlabs` | plain-capability | speech |

## 8. 插件依赖管理

### 8.1 依赖声明

```json
{
  "id": "my-plugin",
  "dependencies": {
    "other-plugin": "^1.0.0"
  },
  "peerDependencies": {
    "openclaw": ">=2026.4.0"
  }
}
```

### 8.2 依赖解析

```typescript
// src/plugins/dependencies.ts

export function resolvePluginDependencies(
  plugin: PluginManifest,
  availablePlugins: Map<string, PluginManifest>
): DependencyResolutionResult {
  const resolved: string[] = [];
  const missing: string[] = [];
  const incompatible: Array<{ dep: string; version: string }> = [];

  for (const [depId, versionRange] of Object.entries(plugin.dependencies ?? {})) {
    const depPlugin = availablePlugins.get(depId);

    if (!depPlugin) {
      missing.push(depId);
      continue;
    }

    if (!satisfies(depPlugin.version, versionRange)) {
      incompatible.push({ dep: depId, version: depPlugin.version });
      continue;
    }

    resolved.push(depId);
  }

  return { resolved, missing, incompatible };
}
```

## 9. 插件生命周期钩子

### 9.1 钩子类型

```typescript
// src/plugins/hooks.ts

export type PluginHookType =
  // 模型解析前
  | 'before_model_resolve'
  // Prompt 构建前
  | 'before_prompt_build'
  // Agent 启动前（遗留）
  | 'before_agent_start'
  // Agent 结束后
  | 'agent_end'
  // 压缩前/后
  | 'before_compaction'
  | 'after_compaction'
  // 工具调用前/后
  | 'before_tool_call'
  | 'after_tool_call'
  // 消息相关
  | 'message_received'
  | 'message_sending'
  | 'message_sent'
  // 会话相关
  | 'session_start'
  | 'session_end'
  // Gateway 相关
  | 'gateway_start'
  | 'gateway_stop';
```

### 9.2 钩子注册与执行

```typescript
// src/plugins/hook-runner-global.ts

const hookRegistry = new Map<PluginHookType, Set<HookHandler>>();

export function registerPluginHook(
  type: PluginHookType,
  handler: HookHandler
): void {
  if (!hookRegistry.has(type)) {
    hookRegistry.set(type, new Set());
  }
  hookRegistry.get(type)!.add(handler);
}

export async function runPluginHooks<T>(
  type: PluginHookType,
  context: HookContext<T>
): Promise<HookResult<T>> {
  const handlers = hookRegistry.get(type) ?? new Set();
  let result: HookResult<T> = { continue: true, data: context.data };

  for (const handler of handlers) {
    try {
      const handlerResult = await handler(context);

      // 处理阻止信号
      if (handlerResult.block) {
        return { continue: false, data: result.data, reason: handlerResult.reason };
      }

      // 合并结果
      if (handlerResult.data) {
        result.data = { ...result.data, ...handlerResult.data };
      }
    } catch (err) {
      log.error(`Hook handler failed for ${type}:`, err);
    }
  }

  return result;
}
```

## 10. 关键代码文件

| 文件 | 描述 |
|------|------|
| `src/plugins/discovery.ts` | 插件发现逻辑 |
| `src/plugins/loader.ts` | 插件加载实现 |
| `src/plugins/manifest.ts` | 清单解析 |
| `src/plugins/contracts/registry.ts` | 插件注册表 |
| `src/plugins/bundled-dir.ts` | 捆绑插件目录管理 |
| `src/plugin-sdk/core.ts` | 插件 SDK 核心 API |
| `src/plugin-sdk/provider-entry.ts` | Provider 注册 API |
| `src/plugin-sdk/channel-contract.ts` | Channel 注册 API |

## 11. 最佳实践

### 11.1 插件开发指南

1. **清单完整性**：始终提供完整的 `openclaw.plugin.json`
2. **类型安全**：使用 TypeScript 和 SDK 类型
3. **错误处理**：优雅处理错误，避免崩溃
4. **日志记录**：使用 `api.log` 记录关键操作
5. **配置验证**：提供 JSON Schema 验证配置

### 11.2 性能考虑

1. **懒加载**：重量级操作延迟到首次使用时
2. **缓存**：缓存昂贵的计算结果
3. **异步**：使用异步 API 避免阻塞
4. **资源清理**：在 `gateway_stop` 钩子中释放资源
