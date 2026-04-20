# OpenClaw 命令系统详解

本文档深入剖析 OpenClaw 的命令系统，包括 CLI 架构、命令注册、参数解析和执行流程。

## 1. 命令系统架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                     COMMAND SYSTEM                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                     CLI ENTRY                               │   │
│  │  ┌─────────────────────────────────────────────────────┐   │   │
│  │  │  openclaw.mjs → entry.ts → index.ts                │   │   │
│  │  │                                                     │   │   │
│  │  │  1. Parse global options (--config, --verbose)     │   │   │
│  │  │  2. Load configuration                             │   │   │
│  │  │  3. Setup logging                                  │   │   │
│  │  │  4. Dispatch to subcommand                         │   │   │
│  │  └─────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   COMMAND REGISTRY                           │   │
│  │  ┌─────────────────────────────────────────────────────┐   │   │
│  │  │  Map<commandName, CommandRegistration>              │   │   │
│  │  │                                                     │   │   │
│  │  │  "agent" → { handler, options, subcommands }       │   │   │
│  │  │  "gateway" → { handler, options, lifecycle }       │   │   │
│  │  │  "plugins" → { handler, subcommands: [...] }       │   │   │
│  │  └─────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   COMMAND EXECUTION                          │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │   │
│  │  │   Parse     │  │   Validate  │  │   Execute   │         │   │
│  │  │   Args      │  │   & Resolve │  │   Handler   │         │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 2. CLI 入口点

### 2.1 入口流程

```typescript
// src/entry.ts

export async function main(argv: string[]): Promise<number> {
  // 1. 解析全局选项
  const globalOpts = parseGlobalOptions(argv);

  // 2. 加载配置（可能使用 --config 指定的路径）
  const config = await loadConfig(globalOpts.configPath);

  // 3. 设置日志
  setupLogging(config, globalOpts.verbose);

  // 4. 创建 CLI
  const cli = createCli({ config });

  // 5. 执行命令
  try {
    await cli.parseAsync(argv);
    return 0;
  } catch (err) {
    handleCliError(err);
    return 1;
  }
}
```

### 2.2 全局选项

```typescript
// src/cli/argv.ts

export interface GlobalOptions {
  // 配置文件路径
  config?: string;
  // 详细输出
  verbose?: boolean;
  // 安静模式
  quiet?: boolean;
  // 彩色输出
  color?: boolean;
  // 工作目录
  cwd?: string;
}

export function parseGlobalOptions(argv: string[]): GlobalOptions {
  const opts: GlobalOptions = {};

  for (let i = 0; i < argv.length; i++) {
    const arg = argv[i];

    if (arg === '--config' || arg === '-c') {
      opts.config = argv[++i];
    } else if (arg === '--verbose' || arg === '-v') {
      opts.verbose = true;
    } else if (arg === '--quiet' || arg === '-q') {
      opts.quiet = true;
    } else if (arg === '--no-color') {
      opts.color = false;
    }
  }

  return opts;
}
```

## 3. 命令注册

### 3.1 命令定义

```typescript
// src/cli/types.ts

export interface CommandDefinition {
  name: string;
  description: string;
  aliases?: string[];
  options?: OptionDefinition[];
  arguments?: ArgumentDefinition[];
  subcommands?: CommandDefinition[];
  handler: CommandHandler;
}

export interface OptionDefinition {
  flags: string;  // "-m, --message <msg>"
  description: string;
  defaultValue?: unknown;
  required?: boolean;
}

export interface ArgumentDefinition {
  name: string;
  description: string;
  required?: boolean;
  variadic?: boolean;
}

export type CommandHandler = (ctx: CommandContext) => Promise<void>;

export interface CommandContext {
  args: Record<string, unknown>;
  options: Record<string, unknown>;
  config: OpenClawConfig;
  deps: CliDeps;
}
```

### 3.2 注册示例

```typescript
// src/cli/agent-cli.ts

export function registerAgentCli(cli: Command, deps: CliDeps) {
  const agent = cli
    .command('agent')
    .description('Run an agent with a message')
    .option('-m, --message <msg>', 'Message to send to the agent')
    .option('-a, --agent-id <id>', 'Agent ID to use')
    .option('-s, --session <key>', 'Session key')
    .option('--thinking <level>', 'Thinking level (low/medium/high)')
    .option('--verbose', 'Verbose output')
    .action(async (opts) => {
      // 处理命令
      await handleAgentCommand(deps, opts);
    });

  // 子命令
  agent
    .command('wait <runId>')
    .description('Wait for an agent run to complete')
    .option('--timeout <ms>', 'Timeout in milliseconds')
    .action(async (runId, opts) => {
      await handleAgentWait(deps, runId, opts);
    });
}
```

### 3.3 插件命令注册

```typescript
// src/plugins/command-registration.ts

export interface PluginCliCommand {
  name: string;
  description: string;
  options?: Array<{
    flags: string;
    description: string;
    defaultValue?: unknown;
  }>;
  descriptorOnly?: boolean; // 延迟加载实际处理器
}

export function registerPluginCliCommands(
  cli: Command,
  registry: PluginRegistry
) {
  for (const [pluginId, commands] of registry.cliCommands) {
    for (const cmd of commands) {
      const command = cli
        .command(cmd.name)
        .description(cmd.description);

      for (const opt of cmd.options ?? []) {
        command.option(opt.flags, opt.description, opt.defaultValue);
      }

      if (cmd.descriptorOnly) {
        // 延迟加载：首次调用时加载实际处理器
        command.action(async (...args) => {
          const handler = await loadPluginCommandHandler(pluginId, cmd.name);
          await handler(...args);
        });
      }
    }
  }
}
```

## 4. 核心命令详解

### 4.1 gateway 命令

```typescript
// src/cli/gateway-cli.ts

export function registerGatewayCli(cli: Command, deps: CliDeps) {
  const gateway = cli
    .command('gateway')
    .description('Gateway management commands')
    .addCommand(createGatewayRunCommand(deps))
    .addCommand(createGatewayStatusCommand(deps));

  return gateway;
}

function createGatewayRunCommand(deps: CliDeps): Command {
  return new Command('run')
    .description('Start the Gateway server')
    .option('--bind <host>', 'Bind host', '127.0.0.1')
    .option('--port <port>', 'Bind port', '18789')
    .option('--token <token>', 'Gateway auth token')
    .option('--daemon', 'Run as daemon')
    .action(async (opts) => {
      await startGatewayServer({
        bindHost: opts.bind,
        bindPort: parseInt(opts.port, 10),
        authToken: opts.token,
        daemon: opts.daemon,
      });
    });
}
```

### 4.2 plugins 命令

```typescript
// src/cli/plugins-cli.ts

export function registerPluginsCli(cli: Command, deps: CliDeps) {
  const plugins = cli
    .command('plugins')
    .description('Plugin management commands');

  plugins
    .command('list')
    .description('List installed plugins')
    .option('--json', 'Output as JSON')
    .action(async (opts) => {
      const plugins = await listInstalledPlugins(deps.config);

      if (opts.json) {
        console.log(JSON.stringify(plugins, null, 2));
      } else {
        for (const plugin of plugins) {
          console.log(`${plugin.enabled ? '✓' : '✗'} ${plugin.id}@${plugin.version}`);
        }
      }
    });

  plugins
    .command('install <spec>')
    .description('Install a plugin')
    .option('--global', 'Install globally')
    .action(async (spec, opts) => {
      await installPlugin(spec, { global: opts.global });
    });

  plugins
    .command('uninstall <id>')
    .description('Uninstall a plugin')
    .action(async (id) => {
      await uninstallPlugin(id);
    });

  plugins
    .command('inspect <id>')
    .description('Inspect a plugin')
    .action(async (id) => {
      const info = await inspectPlugin(id);
      console.log(JSON.stringify(info, null, 2));
    });
}
```

### 4.3 config 命令

```typescript
// src/cli/config-cli.ts

export function registerConfigCli(cli: Command, deps: CliDeps) {
  const config = cli
    .command('config')
    .description('Configuration management');

  config
    .command('get <path>')
    .description('Get a configuration value')
    .action(async (path) => {
      const value = getConfigValue(deps.config, path);
      console.log(value);
    });

  config
    .command('set <path> <value>')
    .description('Set a configuration value')
    .option('--json', 'Parse value as JSON')
    .action(async (path, value, opts) => {
      const parsedValue = opts.json ? JSON.parse(value) : value;
      await setConfigValue(deps.config, path, parsedValue);
      console.log(`Set ${path} = ${JSON.stringify(parsedValue)}`);
    });

  config
    .command('list')
    .description('List all configuration')
    .option('--json', 'Output as JSON')
    .action(async (opts) => {
      if (opts.json) {
        console.log(JSON.stringify(deps.config, null, 2));
      } else {
        printConfigTable(deps.config);
      }
    });

  config
    .command('edit')
    .description('Open configuration in editor')
    .action(async () => {
      const configPath = getConfigFilePath();
      await openInEditor(configPath);
    });
}
```

## 5. 参数解析与验证

### 5.1 参数解析

```typescript
// src/cli/command-options.ts

export interface ParsedCommandOptions {
  // 位置参数
  args: string[];
  // 命名选项
  options: Map<string, unknown>;
  // 标志
  flags: Set<string>;
}

export function parseCommandOptions(
  definition: CommandDefinition,
  argv: string[]
): ParsedCommandOptions {
  const result: ParsedCommandOptions = {
    args: [],
    options: new Map(),
    flags: new Set(),
  };

  for (let i = 0; i < argv.length; i++) {
    const arg = argv[i];

    // 处理选项
    if (arg.startsWith('--')) {
      const [key, value] = arg.slice(2).split('=', 2);
      const optionDef = definition.options?.find(o =>
        o.flags.includes(`--${key}`)
      );

      if (optionDef) {
        if (value !== undefined) {
          result.options.set(key, parseValue(value, optionDef));
        } else {
          result.flags.add(key);
        }
      }
    }
    // 处理短选项
    else if (arg.startsWith('-') && arg.length > 1) {
      const flags = arg.slice(1).split('');
      for (const flag of flags) {
        result.flags.add(flag);
      }
    }
    // 处理位置参数
    else {
      result.args.push(arg);
    }
  }

  return result;
}
```

### 5.2 参数验证

```typescript
// src/cli/command-validation.ts

export interface ValidationResult {
  valid: boolean;
  errors: string[];
}

export function validateCommandArgs(
  definition: CommandDefinition,
  parsed: ParsedCommandOptions
): ValidationResult {
  const errors: string[] = [];

  // 验证必需参数
  for (const arg of definition.arguments ?? []) {
    const index = definition.arguments!.indexOf(arg);
    if (arg.required && parsed.args[index] === undefined) {
      errors.push(`Missing required argument: ${arg.name}`);
    }
  }

  // 验证必需选项
  for (const opt of definition.options ?? []) {
    if (opt.required) {
      const optName = opt.flags.split(',')[0].trim().replace(/^-+/, '');
      if (!parsed.options.has(optName) && !parsed.flags.has(optName)) {
        errors.push(`Missing required option: ${opt.flags}`);
      }
    }
  }

  return {
    valid: errors.length === 0,
    errors,
  };
}
```

## 6. 命令执行上下文

### 6.1 依赖注入

```typescript
// src/cli/deps.ts

export interface CliDeps {
  // 配置
  config: OpenClawConfig;

  // Gateway 客户端
  gatewayClient?: GatewayClient;

  // 日志
  logger: Logger;

  // 终端输出
  terminal: Terminal;

  // 文件系统
  fs: FileSystem;

  // 环境
  env: NodeJS.ProcessEnv;
}

export function createDefaultDeps(
  config: OpenClawConfig
): CliDeps {
  return {
    config,
    logger: createLogger(config),
    terminal: createTerminal(),
    fs: createFileSystem(),
    env: process.env,
  };
}
```

### 6.2 执行包装器

```typescript
// src/cli/command-execution.ts

export async function executeCommand(
  definition: CommandDefinition,
  ctx: CommandContext
): Promise<void> {
  // 1. 前置钩子
  await runHook('command:before', { command: definition.name, ctx });

  try {
    // 2. 执行处理器
    await definition.handler(ctx);

    // 3. 成功钩子
    await runHook('command:success', { command: definition.name, ctx });
  } catch (err) {
    // 4. 错误处理
    await runHook('command:error', { command: definition.name, error: err, ctx });
    throw err;
  } finally {
    // 5. 后置钩子
    await runHook('command:after', { command: definition.name, ctx });
  }
}
```

## 7. 子命令系统

### 7.1 嵌套命令

```
openclaw
├── agent
│   ├── run (default)
│   └── wait
├── gateway
│   ├── run
│   ├── status
│   └── stop
├── plugins
│   ├── list
│   ├── install
│   ├── uninstall
│   └── inspect
├── config
│   ├── get
│   ├── set
│   ├── list
│   └── edit
├── channels
│   ├── add
│   ├── remove
│   ├── list
│   └── status
└── hooks
    ├── list
    ├── enable
    ├── disable
    └── check
```

### 7.2 子命令实现

```typescript
// src/cli/nested-commands.ts

export function createNestedCommand(
  name: string,
  description: string
): Command {
  const cmd = new Command(name).description(description);

  // 添加帮助子命令
  cmd
    .command('help [command]')
    .description('Show help for a command')
    .action(async (subcommand) => {
      if (subcommand) {
        const sub = cmd.commands.find(c => c.name() === subcommand);
        if (sub) {
          sub.help();
        } else {
          console.error(`Unknown command: ${subcommand}`);
        }
      } else {
        cmd.help();
      }
    });

  return cmd;
}
```

## 8. 自动补全

### 8.1 Bash 补全

```typescript
// src/cli/completion-cli.ts

export function registerCompletionCli(cli: Command) {
  cli
    .command('completion')
    .description('Generate shell completion scripts')
    .argument('<shell>', 'Shell type (bash, zsh, fish)')
    .action(async (shell) => {
      const script = generateCompletionScript(shell);
      console.log(script);
    });
}

function generateCompletionScript(shell: string): string {
  switch (shell) {
    case 'bash':
      return generateBashCompletion();
    case 'zsh':
      return generateZshCompletion();
    case 'fish':
      return generateFishCompletion();
    default:
      throw new Error(`Unsupported shell: ${shell}`);
  }
}

function generateBashCompletion(): string {
  return `
_openclaw_completions() {
  local cur prev opts
  COMPREPLY=()
  cur="\${COMP_WORDS[COMP_CWORD]}"
  prev="\${COMP_WORDS[COMP_CWORD-1]}"

  opts="agent gateway plugins config channels hooks help"

  case "\${prev}" in
    openclaw)
      COMPREPLY=( $(compgen -W "\${opts}" -- \${cur}) )
      return 0
      ;;
    agent)
      local agent_opts="--message --agent-id --session --thinking --verbose"
      COMPREPLY=( $(compgen -W "\${agent_opts}" -- \${cur}) )
      return 0
      ;;
    # ... more cases
  esac
}

complete -F _openclaw_completions openclaw
  `.trim();
}
```

## 9. 关键代码文件

| 文件 | 描述 |
|------|------|
| `src/entry.ts` | CLI 入口点 |
| `src/index.ts` | 主程序 |
| `src/cli/deps.ts` | 依赖注入 |
| `src/cli/argv.ts` | 参数解析 |
| `src/cli/agent-cli.ts` | Agent 命令 |
| `src/cli/gateway-cli.ts` | Gateway 命令 |
| `src/cli/daemon-cli.ts` | 守护进程命令 |
| `src/cli/plugins-cli.ts` | 插件命令 |
| `src/cli/config-cli.ts` | 配置命令 |
| `src/cli/channels-cli.ts` | 通道命令 |
| `src/cli/cron-cli.ts` | Cron 命令 |
| `src/cli/hooks-cli.ts` | 钩子命令 |
| `src/cli/completion-cli.ts` | 自动补全 |

## 10. 命令开发指南

### 10.1 创建新命令

```typescript
// src/cli/my-command-cli.ts

import { Command } from 'commander';
import type { CliDeps } from './deps.js';

export function registerMyCommand(cli: Command, deps: CliDeps) {
  cli
    .command('my-command')
    .description('Description of my command')
    .argument('<required-arg>', 'Required argument description')
    .option('-o, --optional <value>', 'Optional option')
    .option('-f, --flag', 'Boolean flag')
    .action(async (arg, opts) => {
      // 命令实现
      console.log(`Arg: ${arg}`);
      console.log(`Opts:`, opts);

      // 使用依赖
      const result = await doSomething(deps);
      console.log(result);
    });
}
```

### 10.2 注册到主 CLI

```typescript
// src/cli/index.ts

import { registerMyCommand } from './my-command-cli.js';

export function createCli(deps: CliDeps): Command {
  const cli = new Command('openclaw');

  // 注册现有命令...
  registerAgentCli(cli, deps);
  registerGatewayCli(cli, deps);

  // 注册新命令
  registerMyCommand(cli, deps);

  return cli;
}
```
