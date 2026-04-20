# OpenClaw 配置系统详解

本文档深入剖析 OpenClaw 的配置系统，包括配置格式、加载机制、验证规则和运行时管理。

## 1. 配置系统架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                   CONFIGURATION SYSTEM                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   CONFIG SOURCES                             │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐            │   │
│  │  │   Default   │ │   File      │ │   Environment│           │   │
│  │  │   Values    │ │  (JSON5)    │ │   Variables  │           │   │
│  │  │             │ │ ~/.openclaw/│ │  OPENCLAW_*  │           │   │
│  │  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘            │   │
│  │         └───────────────┼───────────────┘                   │   │
│  │                         │                                    │   │
│  │                         ▼                                    │   │
│  │              ┌─────────────────────┐                        │   │
│  │              │   Config Merger     │                        │   │
│  │              │   (Layered Config)  │                        │   │
│  │              └─────────────────────┘                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   CONFIG VALIDATION                          │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │   │
│  │  │   Schema    │  │   Plugin    │  │   Type      │         │   │
│  │  │   Check     │  │   Config    │  │   Check     │         │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   RUNTIME CONFIG                             │   │
│  │  ┌─────────────────────────────────────────────────────┐   │   │
│  │  │  {                                                  │   │   │
│  │  │    gateway: { host, port, auth },                  │   │   │
│  │  │    agents: { defaults, timeout },                  │   │   │
│  │  │    channels: { discord, slack, ... },              │   │   │
│  │  │    providers: { openai, anthropic, ... },          │   │   │
│  │  │    plugins: { enabled, config },                   │   │   │
│  │  │    hooks: { internal: { enabled } },               │   │   │
│  │  │  }                                                  │   │   │
│  │  └─────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 2. 配置文件格式

### 2.1 文件位置

| 位置 | 用途 | 优先级 |
|------|------|--------|
| `~/.openclaw/config.json5` | 用户配置 | 最高 |
| `./.openclaw/config.json5` | 项目配置 | 中 |
| `/etc/openclaw/config.json5` | 系统配置 | 低 |

### 2.2 JSON5 格式

```json5
// ~/.openclaw/config.json5
{
  // Gateway 配置
  gateway: {
    host: '127.0.0.1',
    port: 18789,
    auth: {
      token: '${OPENCLAW_GATEWAY_TOKEN}',  // 环境变量引用
    },
  },

  // Agent 默认配置
  agents: {
    defaults: {
      provider: 'anthropic',
      model: 'claude-sonnet-4-6',
      thinking: 'medium',
      timeoutSeconds: 172800,  // 48 小时
    },
  },

  // 通道配置
  channels: {
    discord: {
      token: '${DISCORD_BOT_TOKEN}',
      actions: {
        roles: true,
        moderation: false,
      },
    },
    slack: {
      token: '${SLACK_BOT_TOKEN}',
      signingSecret: '${SLACK_SIGNING_SECRET}',
    },
  },

  // 提供者配置
  providers: {
    openai: {
      apiKey: '${OPENAI_API_KEY}',
      organization: '${OPENAI_ORG_ID}',
    },
    anthropic: {
      apiKey: '${ANTHROPIC_API_KEY}',
    },
  },

  // 插件配置
  plugins: {
    enabled: ['discord', 'slack', 'github'],
    config: {
      'my-plugin': {
        apiKey: 'xxx',
      },
    },
  },

  // 钩子配置
  hooks: {
    internal: {
      enabled: true,
      load: {
        extraDirs: ['/path/to/custom/hooks'],
      },
    },
  },

  // 技能配置
  skills: {
    remote: true,
    refreshInterval: 30000,
  },
}
```

### 2.3 环境变量引用

```typescript
// src/config/io.ts

export function resolveConfigEnvVars(value: string): string {
  // 支持 ${VAR} 和 ${VAR:-default} 语法
  return value.replace(/\$\{(\w+)(?::-(.*?))?\}/g, (match, varName, defaultValue) => {
    const envValue = process.env[varName];
    if (envValue !== undefined) {
      return envValue;
    }
    if (defaultValue !== undefined) {
      return defaultValue;
    }
    throw new Error(`Missing required environment variable: ${varName}`);
  });
}
```

## 3. 配置加载流程

### 3.1 加载流程

```typescript
// src/config/io.ts

export async function loadConfig(
  configPath?: string
): Promise<OpenClawConfig> {
  // 1. 从默认配置开始
  let config = getDefaultConfig();

  // 2. 加载系统配置
  const systemConfig = await loadConfigFile('/etc/openclaw/config.json5');
  if (systemConfig) {
    config = mergeConfigs(config, systemConfig);
  }

  // 3. 加载项目配置
  const projectConfig = await loadConfigFile('./.openclaw/config.json5');
  if (projectConfig) {
    config = mergeConfigs(config, projectConfig);
  }

  // 4. 加载用户配置
  const userConfigPath = configPath ?? '~/.openclaw/config.json5';
  const userConfig = await loadConfigFile(userConfigPath);
  if (userConfig) {
    config = mergeConfigs(config, userConfig);
  }

  // 5. 应用环境变量覆盖
  config = applyEnvVarOverrides(config);

  // 6. 验证配置
  validateConfigObject(config);

  return config;
}
```

### 3.2 配置文件读取

```typescript
// src/config/io.ts

export async function loadConfigFile(
  filePath: string
): Promise<Partial<OpenClawConfig> | null> {
  const resolvedPath = resolveUserPath(filePath);

  try {
    const content = await fs.readFile(resolvedPath, 'utf-8');
    return parseConfigJson5(content);
  } catch (err) {
    if ((err as NodeJS.ErrnoException).code === 'ENOENT') {
      return null;
    }
    throw err;
  }
}

export function parseConfigJson5(content: string): unknown {
  // 解析 JSON5（支持注释、尾随逗号等）
  return JSON5.parse(content);
}
```

### 3.3 配置合并

```typescript
// src/config/merge.ts

export function mergeConfigs(
  base: OpenClawConfig,
  override: Partial<OpenClawConfig>
): OpenClawConfig {
  return deepMerge(base, override, {
    // 数组：替换而非合并
    arrayMerge: (target, source) => source,
  });
}

function deepMerge(
  target: Record<string, unknown>,
  source: Record<string, unknown>,
  options: { arrayMerge?: (t: unknown[], s: unknown[]) => unknown[] }
): Record<string, unknown> {
  const result = { ...target };

  for (const key of Object.keys(source)) {
    const sourceValue = source[key];
    const targetValue = target[key];

    if (
      typeof sourceValue === 'object' &&
      sourceValue !== null &&
      !Array.isArray(sourceValue) &&
      typeof targetValue === 'object' &&
      targetValue !== null &&
      !Array.isArray(targetValue)
    ) {
      // 递归合并对象
      result[key] = deepMerge(
        targetValue as Record<string, unknown>,
        sourceValue as Record<string, unknown>,
        options
      );
    } else if (Array.isArray(sourceValue) && options.arrayMerge) {
      // 使用自定义数组合并策略
      result[key] = Array.isArray(targetValue)
        ? options.arrayMerge(targetValue, sourceValue)
        : sourceValue;
    } else {
      // 直接覆盖
      result[key] = sourceValue;
    }
  }

  return result;
}
```

## 4. 配置验证

### 4.1 Schema 定义

```typescript
// src/config/schema.ts

import { Type, type Static } from '@sinclair/typebox';

export const ConfigSchema = Type.Object({
  gateway: Type.Optional(Type.Object({
    host: Type.String({ default: '127.0.0.1' }),
    port: Type.Number({ default: 18789 }),
    auth: Type.Optional(Type.Object({
      token: Type.Optional(Type.String()),
    })),
  })),

  agents: Type.Optional(Type.Object({
    defaults: Type.Optional(Type.Object({
      provider: Type.String(),
      model: Type.String(),
      thinking: Type.Optional(Type.String()),
      timeoutSeconds: Type.Optional(Type.Number()),
    })),
  })),

  channels: Type.Optional(Type.Record(Type.String(), Type.Object({
    token: Type.String(),
  }))),

  providers: Type.Optional(Type.Record(Type.String(), Type.Object({
    apiKey: Type.Optional(Type.String()),
    baseUrl: Type.Optional(Type.String()),
  }))),

  plugins: Type.Optional(Type.Object({
    enabled: Type.Optional(Type.Array(Type.String())),
    config: Type.Optional(Type.Record(Type.String(), Type.Unknown())),
  })),

  hooks: Type.Optional(Type.Object({
    internal: Type.Optional(Type.Object({
      enabled: Type.Boolean({ default: true }),
      load: Type.Optional(Type.Object({
        extraDirs: Type.Optional(Type.Array(Type.String())),
      })),
    })),
  })),
});

export type OpenClawConfig = Static<typeof ConfigSchema>;
```

### 4.2 验证实现

```typescript
// src/config/validation.ts

import { Value } from '@sinclair/typebox/value';

export interface ValidationError {
  path: string;
  message: string;
  value: unknown;
}

export function validateConfigObject(
  config: unknown
): { valid: boolean; errors: ValidationError[] } {
  const errors: ValidationError[] = [];

  // 使用 TypeBox 验证
  const check = Value.Check(ConfigSchema, config);

  if (!check) {
    const iterator = Value.Errors(ConfigSchema, config);
    for (const error of iterator) {
      errors.push({
        path: error.path,
        message: error.message,
        value: error.value,
      });
    }
  }

  // 额外验证
  const extraErrors = validateExtraRules(config as OpenClawConfig);
  errors.push(...extraErrors);

  return {
    valid: errors.length === 0,
    errors,
  };
}

function validateExtraRules(config: OpenClawConfig): ValidationError[] {
  const errors: ValidationError[] = [];

  // 验证通道配置
  if (config.channels) {
    for (const [name, channelConfig] of Object.entries(config.channels)) {
      if (!channelConfig.token) {
        errors.push({
          path: `channels.${name}.token`,
          message: 'Token is required for channel',
          value: undefined,
        });
      }
    }
  }

  return errors;
}
```

## 5. 运行时配置管理

### 5.1 运行时快照

```typescript
// src/config/runtime-overrides.ts

let runtimeConfigSnapshot: OpenClawConfig | null = null;

export function setRuntimeConfigSnapshot(config: OpenClawConfig): void {
  runtimeConfigSnapshot = deepFreeze(config);
}

export function getRuntimeConfigSnapshot(): OpenClawConfig | null {
  return runtimeConfigSnapshot;
}

export function clearRuntimeConfigSnapshot(): void {
  runtimeConfigSnapshot = null;
}

function deepFreeze<T>(obj: T): T {
  Object.freeze(obj);

  for (const key of Object.keys(obj as Record<string, unknown>)) {
    const value = (obj as Record<string, unknown>)[key];
    if (value !== null && typeof value === 'object') {
      deepFreeze(value);
    }
  }

  return obj;
}
```

### 5.2 配置变更监听

```typescript
// src/config/io.ts

const configWriteListeners = new Set<ConfigWriteListener>();

export interface ConfigWriteListener {
  (notification: ConfigWriteNotification): void;
}

export interface ConfigWriteNotification {
  path: string;
  previousValue: unknown;
  newValue: unknown;
}

export function registerConfigWriteListener(
  listener: ConfigWriteListener
): () => void {
  configWriteListeners.add(listener);
  return () => configWriteListeners.delete(listener);
}

export function notifyConfigWrite(notification: ConfigWriteNotification): void {
  for (const listener of configWriteListeners) {
    try {
      listener(notification);
    } catch (err) {
      log.error('Config write listener failed:', err);
    }
  }
}
```

### 5.3 配置修改

```typescript
// src/config/mutate.ts

export async function mutateConfigFile(
  configPath: string,
  mutations: ConfigMutation[]
): Promise<void> {
  // 1. 读取现有配置
  const content = await fs.readFile(configPath, 'utf-8');
  const config = JSON5.parse(content);

  // 2. 应用变更
  for (const mutation of mutations) {
    applyMutation(config, mutation);
  }

  // 3. 验证
  const validation = validateConfigObject(config);
  if (!validation.valid) {
    throw new ConfigMutationConflictError(validation.errors);
  }

  // 4. 原子写入
  await writeConfigFileAtomically(configPath, config);
}

function applyMutation(
  config: Record<string, unknown>,
  mutation: ConfigMutation
): void {
  const keys = mutation.path.split('.');
  let current: Record<string, unknown> = config;

  // 遍历到倒数第二个 key
  for (let i = 0; i < keys.length - 1; i++) {
    const key = keys[i];
    if (!(key in current)) {
      current[key] = {};
    }
    current = current[key] as Record<string, unknown>;
  }

  const lastKey = keys[keys.length - 1];

  switch (mutation.type) {
    case 'set':
      current[lastKey] = mutation.value;
      break;
    case 'delete':
      delete current[lastKey];
      break;
    case 'push':
      if (!Array.isArray(current[lastKey])) {
        current[lastKey] = [];
      }
      (current[lastKey] as unknown[]).push(mutation.value);
      break;
  }
}
```

## 6. 环境变量覆盖

### 6.1 映射规则

```typescript
// src/config/env-overrides.ts

const ENV_VAR_MAPPINGS: Record<string, string> = {
  // Gateway
  'OPENCLAW_GATEWAY_HOST': 'gateway.host',
  'OPENCLAW_GATEWAY_PORT': 'gateway.port',
  'OPENCLAW_GATEWAY_TOKEN': 'gateway.auth.token',

  // Agent defaults
  'OPENCLAW_AGENT_PROVIDER': 'agents.defaults.provider',
  'OPENCLAW_AGENT_MODEL': 'agents.defaults.model',
  'OPENCLAW_AGENT_THINKING': 'agents.defaults.thinking',

  // Channel tokens
  'DISCORD_BOT_TOKEN': 'channels.discord.token',
  'SLACK_BOT_TOKEN': 'channels.slack.token',
  'TELEGRAM_BOT_TOKEN': 'channels.telegram.token',

  // Provider API keys
  'OPENAI_API_KEY': 'providers.openai.apiKey',
  'ANTHROPIC_API_KEY': 'providers.anthropic.apiKey',
  'GOOGLE_API_KEY': 'providers.google.apiKey',
};

export function applyEnvVarOverrides(
  config: OpenClawConfig
): OpenClawConfig {
  const result = { ...config };

  for (const [envVar, configPath] of Object.entries(ENV_VAR_MAPPINGS)) {
    const envValue = process.env[envVar];
    if (envValue !== undefined) {
      setConfigValue(result, configPath, envValue);
    }
  }

  return result;
}
```

### 6.2 数组环境变量

```typescript
// 支持 OPENCLAW_PLUGINS_ENABLED=plugin1,plugin2,plugin3
export function parseArrayEnvVar(value: string): string[] {
  return value.split(',').map(s => s.trim()).filter(Boolean);
}

// 在 applyEnvVarOverrides 中
if (envVar === 'OPENCLAW_PLUGINS_ENABLED') {
  setConfigValue(result, 'plugins.enabled', parseArrayEnvVar(envValue));
}
```

## 7. 配置路径工具

### 7.1 路径解析

```typescript
// src/config/paths.ts

export function resolveUserPath(inputPath: string): string {
  // 展开 ~ 为用户主目录
  if (inputPath.startsWith('~/')) {
    return path.join(os.homedir(), inputPath.slice(2));
  }
  return path.resolve(inputPath);
}

export function getConfigFilePath(): string {
  const envPath = process.env.OPENCLAW_CONFIG_PATH;
  if (envPath) {
    return resolveUserPath(envPath);
  }
  return resolveUserPath('~/.openclaw/config.json5');
}

export function getStateDir(): string {
  const envDir = process.env.OPENCLAW_STATE_DIR;
  if (envDir) {
    return resolveUserPath(envDir);
  }
  return resolveUserPath('~/.openclaw');
}

export function getWorkspaceDir(config: OpenClawConfig): string {
  if (config.workspace?.dir) {
    return resolveUserPath(config.workspace.dir);
  }
  return path.join(getStateDir(), 'workspace');
}
```

### 7.2 配置值访问

```typescript
// src/config/utils.ts

export function getConfigValue(
  config: OpenClawConfig,
  path: string
): unknown {
  const keys = path.split('.');
  let current: unknown = config;

  for (const key of keys) {
    if (current === null || typeof current !== 'object') {
      return undefined;
    }
    current = (current as Record<string, unknown>)[key];
  }

  return current;
}

export function setConfigValue(
  config: OpenClawConfig,
  path: string,
  value: unknown
): void {
  const keys = path.split('.');
  let current: Record<string, unknown> = config;

  for (let i = 0; i < keys.length - 1; i++) {
    const key = keys[i];
    if (!(key in current) || typeof current[key] !== 'object') {
      current[key] = {};
    }
    current = current[key] as Record<string, unknown>;
  }

  current[keys[keys.length - 1]] = value;
}
```

## 8. 关键代码文件

| 文件 | 描述 |
|------|------|
| `src/config/types.ts` | 配置类型定义 |
| `src/config/schema.ts` | JSON Schema 定义 |
| `src/config/io.ts` | 配置加载/保存 |
| `src/config/merge.ts` | 配置合并 |
| `src/config/validation.ts` | 配置验证 |
| `src/config/mutate.ts` | 配置修改 |
| `src/config/paths.ts` | 配置路径 |
| `src/config/runtime-overrides.ts` | 运行时配置 |
| `src/config/env-overrides.ts` | 环境变量覆盖 |

## 9. 最佳实践

### 9.1 配置组织建议

```json5
{
  // 1. 基础配置
  gateway: { ... },

  // 2. Agent 配置
  agents: { ... },

  // 3. 通道配置（按字母顺序）
  channels: {
    discord: { ... },
    slack: { ... },
    telegram: { ... },
  },

  // 4. 提供者配置
  providers: {
    anthropic: { ... },
    openai: { ... },
  },

  // 5. 插件配置
  plugins: { ... },

  // 6. 钩子配置
  hooks: { ... },
}
```

### 9.2 敏感信息处理

```json5
{
  // ✅ 使用环境变量引用
  providers: {
    openai: {
      apiKey: '${OPENAI_API_KEY}',
    },
  },

  // ❌ 不要直接写入敏感信息
  providers: {
    openai: {
      apiKey: 'sk-abc123...',
    },
  },
}
```
