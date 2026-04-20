# OpenClaw Skill 系统详解

本文档深入剖析 OpenClaw 的 Skill 系统，包括技能定义格式、发现机制、加载流程和动态技能。

## 1. Skill 系统架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                     SKILL SYSTEM                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   SKILL SOURCES                              │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐            │   │
│  │  │   Built-in  │ │   Workspace │ │   Remote    │            │   │
│  │  │   Skills    │ │   Skills    │ │   Skills    │            │   │
│  │  │  (skills/)  │ │ (<ws>/.open)│ │  (Clawhub)  │            │   │
│  │  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘            │   │
│  │         └───────────────┼───────────────┘                   │   │
│  │                         │                                    │   │
│  │                         ▼                                    │   │
│  │              ┌─────────────────────┐                        │   │
│  │              │   Skill Discovery   │                        │   │
│  │              │   (SKILL.md parsing)│                        │   │
│  │              └─────────────────────┘                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   SKILL SNAPSHOT                             │   │
│  │  ┌─────────────────────────────────────────────────────┐   │   │
│  │  │  {                                                  │   │   │
│  │  │    skills: [                                        │   │   │
│  │  │      { name: "github", description: "...", ... },   │   │   │
│  │  │      { name: "discord", description: "...", ... }   │   │   │
│  │  │    ],                                               │   │   │
│  │  │    version: "abc123",  // hash of skills content    │   │   │
│  │  │    refreshInterval: 30000  // ms                    │   │   │
│  │  │  }                                                  │   │   │
│  │  └─────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   SKILL USAGE                                │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │   │
│  │  │   Prompt    │  │   Tool      │  │   Remote    │         │   │
│  │  │   Injection │  │   Selection │  │   Execution │         │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 2. Skill 定义格式

### 2.1 SKILL.md 结构

```markdown
---
name: skill-name
description: "Short description of the skill"
metadata:
  openclaw:
    emoji: "🐙"
    requires:
      bins: ["gh"]           # 需要的二进制
      anyBins: ["git", "gh"] # 至少需要一个
      env: ["GITHUB_TOKEN"]  # 需要的环境变量
      config: ["channels.discord.token"] # 需要的配置
    install:
      - id: "brew"
        kind: "brew"
        formula: "gh"
        bins: ["gh"]
allowed-tools: ["bash", "message"] # 允许的工具
---

# Skill Name

## When to Use

✅ **USE this skill when:**
- 情况 1
- 情况 2

❌ **DON'T use this skill when:**
- 情况 A
- 情况 B

## Common Commands

```bash
# List PRs
gh pr list --repo owner/repo
```

## Notes

- Important note 1
- Important note 2
```

### 2.2 元数据字段

| 字段 | 类型 | 描述 |
|------|------|------|
| `name` | string | Skill 标识符（唯一） |
| `description` | string | 简短描述 |
| `metadata.openclaw.emoji` | string | 显示用的表情符号 |
| `metadata.openclaw.requires.bins` | string[] | 必需的二进制文件 |
| `metadata.openclaw.requires.anyBins` | string[] | 至少需要一个的二进制 |
| `metadata.openclaw.requires.env` | string[] | 必需的环境变量 |
| `metadata.openclaw.requires.config` | string[] | 必需的配置路径 |
| `metadata.openclaw.install` | InstallSpec[] | 安装说明 |
| `allowed-tools` | string[] | 此技能允许使用的工具 |

### 2.3 InstallSpec 格式

```typescript
interface InstallSpec {
  id: string;           // 安装方式标识
  kind: 'brew' | 'apt' | 'npm' | 'pip' | 'script';
  label: string;        // 显示标签
  // 特定类型的字段
  formula?: string;     // brew 用
  package?: string;     // apt/npm/pip 用
  script?: string;      // script 用
  bins?: string[];      // 安装后提供的二进制
}
```

## 3. Skill 发现机制

### 3.1 发现来源

```typescript
// src/agents/skills.ts

export interface SkillDiscoveryOptions {
  // 内置技能目录
  builtinSkillsDir?: string;
  // 工作区技能目录
  workspaceSkillsDir?: string;
  // 远程技能
  remoteSkills?: boolean;
}

export async function discoverSkills(
  opts: SkillDiscoveryOptions
): Promise<Skill[]> {
  const skills: Skill[] = [];

  // 1. 发现内置技能
  if (opts.builtinSkillsDir) {
    const builtinSkills = await scanSkillDirectory(opts.builtinSkillsDir);
    skills.push(...builtinSkills);
  }

  // 2. 发现工作区技能
  if (opts.workspaceSkillsDir) {
    const workspaceSkills = await scanSkillDirectory(opts.workspaceSkillsDir);
    skills.push(...workspaceSkills);
  }

  // 3. 发现远程技能
  if (opts.remoteSkills) {
    const remoteSkills = await discoverRemoteSkills();
    skills.push(...remoteSkills);
  }

  return skills;
}
```

### 3.2 目录扫描

```typescript
// src/agents/skills/discovery.ts

export async function scanSkillDirectory(dir: string): Promise<Skill[]> {
  const skills: Skill[] = [];

  const entries = await fs.readdir(dir, { withFileTypes: true });

  for (const entry of entries) {
    if (!entry.isDirectory()) continue;

    const skillDir = path.join(dir, entry.name);
    const skill = await loadSkillFromDirectory(skillDir);

    if (skill) {
      skills.push(skill);
    }
  }

  return skills;
}

async function loadSkillFromDirectory(dir: string): Promise<Skill | null> {
  const skillMdPath = path.join(dir, 'SKILL.md');

  try {
    const content = await fs.readFile(skillMdPath, 'utf-8');
    return parseSkillMarkdown(content, dir);
  } catch (err) {
    // SKILL.md 不存在，跳过
    return null;
  }
}
```

### 3.3 Markdown 解析

```typescript
// src/agents/skills/parser.ts

import YAML from 'yaml';

export interface ParsedSkill {
  name: string;
  description: string;
  metadata: SkillMetadata;
  content: string;  // Markdown 内容（不含 frontmatter）
}

export function parseSkillMarkdown(
  content: string,
  sourceDir: string
): ParsedSkill {
  // 解析 YAML frontmatter
  const frontmatterMatch = content.match(/^---\n([\s\S]*?)\n---\n([\s\S]*)$/);

  if (!frontmatterMatch) {
    throw new Error('Invalid SKILL.md: missing frontmatter');
  }

  const [, yamlContent, markdownContent] = frontmatterMatch;
  const frontmatter = YAML.parse(yamlContent);

  return {
    name: frontmatter.name,
    description: frontmatter.description,
    metadata: frontmatter.metadata?.openclaw ?? {},
    content: markdownContent.trim(),
  };
}
```

## 4. Skill 快照与缓存

### 4.1 Skill Snapshot

```typescript
// src/agents/skills.ts

export interface SkillSnapshot {
  skills: SkillEntry[];
  version: string;  // 内容哈希
  refreshInterval: number;
  generatedAt: string;
}

export interface SkillEntry {
  name: string;
  description: string;
  emoji?: string;
  content: string;
  allowedTools?: string[];
  requirements?: SkillRequirements;
}
```

### 4.2 构建快照

```typescript
// src/agents/skills.ts

export async function buildWorkspaceSkillSnapshot(
  config: OpenClawConfig
): Promise<SkillSnapshot> {
  // 1. 发现所有技能
  const skills = await discoverSkills({
    builtinSkillsDir: resolveBuiltinSkillsDir(),
    workspaceSkillsDir: resolveWorkspaceSkillsDir(config),
    remoteSkills: config.skills?.remote !== false,
  });

  // 2. 检查技能可用性
  const eligibleSkills = await filterEligibleSkills(skills);

  // 3. 构建条目
  const entries = eligibleSkills.map(skill => ({
    name: skill.name,
    description: skill.description,
    emoji: skill.metadata?.emoji,
    content: skill.content,
    allowedTools: skill.allowedTools,
    requirements: skill.metadata?.requires,
  }));

  // 4. 计算版本哈希
  const version = hashSkillsContent(entries);

  return {
    skills: entries,
    version,
    refreshInterval: config.skills?.refreshInterval ?? 30000,
    generatedAt: new Date().toISOString(),
  };
}
```

### 4.3 技能可用性检查

```typescript
// src/agents/skills/eligibility.ts

export async function checkSkillEligibility(
  skill: Skill,
  context: EligibilityContext
): Promise<EligibilityResult> {
  const requirements = skill.metadata?.requires;
  if (!requirements) {
    return { eligible: true };
  }

  // 检查二进制
  if (requirements.bins) {
    for (const bin of requirements.bins) {
      if (!context.hasBin(bin)) {
        return {
          eligible: false,
          reason: `Missing required binary: ${bin}`,
        };
      }
    }
  }

  // 检查环境变量
  if (requirements.env) {
    for (const env of requirements.env) {
      if (!process.env[env]) {
        return {
          eligible: false,
          reason: `Missing required environment variable: ${env}`,
        };
      }
    }
  }

  // 检查配置
  if (requirements.config) {
    for (const configPath of requirements.config) {
      if (!getConfigValue(context.config, configPath)) {
        return {
          eligible: false,
          reason: `Missing required config: ${configPath}`,
        };
      }
    }
  }

  return { eligible: true };
}
```

## 5. 内置技能详解

### 5.1 github

GitHub 操作技能，使用 `gh` CLI。

```yaml
---
name: github
description: "GitHub operations via `gh` CLI"
metadata:
  openclaw:
    emoji: "🐙"
    requires:
      bins: ["gh"]
    install:
      - id: "brew"
        kind: "brew"
        formula: "gh"
        bins: ["gh"]
---
```

### 5.2 discord

Discord 操作技能。

```yaml
---
name: discord
description: "Discord ops via the message tool"
metadata:
  openclaw:
    emoji: "🎮"
    requires:
      config: ["channels.discord.token"]
allowed-tools: ["message"]
---
```

### 5.3 coding-agent

编码 Agent 技能，用于委托任务给 Codex/Claude Code。

```yaml
---
name: coding-agent
description: "Delegate coding tasks to Codex, Claude Code, or Pi agents"
metadata:
  openclaw:
    emoji: "🧩"
    requires:
      anyBins: ["claude", "codex", "opencode", "pi"]
    install:
      - id: "node-claude"
        kind: "node"
        package: "@anthropic-ai/claude-code"
        bins: ["claude"]
---
```

## 6. 远程技能（Clawhub）

### 6.1 远程技能发现

```typescript
// src/infra/skills-remote.ts

export async function discoverRemoteSkills(): Promise<Skill[]> {
  const clawhubUrl = process.env.CLAWHUB_URL ?? 'https://clawhub.openclaw.ai';

  try {
    const response = await fetch(`${clawhubUrl}/api/v1/skills`);
    const data = await response.json();

    return data.skills.map((remoteSkill: RemoteSkill) => ({
      name: remoteSkill.name,
      description: remoteSkill.description,
      metadata: remoteSkill.metadata,
      content: remoteSkill.content,
      source: 'remote',
      remoteUrl: remoteSkill.url,
    }));
  } catch (err) {
    log.warn('Failed to fetch remote skills:', err);
    return [];
  }
}
```

### 6.2 远程技能缓存

```typescript
// src/infra/skills-remote.ts

const remoteSkillCache = new Map<string, {
  skills: Skill[];
  expiresAt: number;
}>();

const REMOTE_SKILL_CACHE_TTL = 5 * 60 * 1000; // 5 分钟

export async function getCachedRemoteSkills(): Promise<Skill[]> {
  const cacheKey = 'default';
  const cached = remoteSkillCache.get(cacheKey);

  if (cached && cached.expiresAt > Date.now()) {
    return cached.skills;
  }

  const skills = await discoverRemoteSkills();
  remoteSkillCache.set(cacheKey, {
    skills,
    expiresAt: Date.now() + REMOTE_SKILL_CACHE_TTL,
  });

  return skills;
}
```

## 7. Skill 在 Agent 中的使用

### 7.1 Prompt 注入

```typescript
// src/agents/prompt-builder.ts

export function buildSystemPrompt(params: {
  basePrompt: string;
  skills: SkillSnapshot;
  context?: string;
}): string {
  const sections: string[] = [params.basePrompt];

  // 添加技能上下文
  if (params.skills.skills.length > 0) {
    sections.push('## Available Skills\n');

    for (const skill of params.skills.skills) {
      sections.push(`### ${skill.emoji ?? '🔧'} ${skill.name}`);
      sections.push(skill.description);
      sections.push('');
      sections.push(skill.content);
      sections.push('');
    }
  }

  // 添加自定义上下文
  if (params.context) {
    sections.push('## Context\n');
    sections.push(params.context);
  }

  return sections.join('\n');
}
```

### 7.2 工具选择限制

```typescript
// src/agents/tools.ts

export function getAllowedToolsForSkills(
  skills: SkillSnapshot
): Set<string> | null {
  const allowedTools = new Set<string>();
  let hasRestrictions = false;

  for (const skill of skills.skills) {
    if (skill.allowedTools && skill.allowedTools.length > 0) {
      hasRestrictions = true;
      for (const tool of skill.allowedTools) {
        allowedTools.add(tool);
      }
    }
  }

  return hasRestrictions ? allowedTools : null;
}

export function isToolAllowed(
  toolName: string,
  skills: SkillSnapshot
): boolean {
  const allowed = getAllowedToolsForSkills(skills);

  if (allowed === null) {
    // 无限制
    return true;
  }

  return allowed.has(toolName);
}
```

## 8. Skill 管理 CLI

### 8.1 命令

```bash
# 列出可用技能
openclaw skills list

# 查看技能详情
openclaw skills info github

# 刷新技能缓存
openclaw skills refresh

# 启用/禁用远程技能
openclaw skills config set remote true
```

### 8.2 实现

```typescript
// src/cli/skills-cli.ts

export function registerSkillsCli(cli: Command) {
  const skills = cli.command('skills');

  skills
    .command('list')
    .description('List available skills')
    .action(async () => {
      const snapshot = await buildWorkspaceSkillSnapshot(await loadConfig());

      for (const skill of snapshot.skills) {
        console.log(`${skill.emoji ?? '🔧'} ${skill.name}`);
        console.log(`   ${skill.description}`);
      }
    });

  skills
    .command('info <name>')
    .description('Show skill details')
    .action(async (name) => {
      const snapshot = await buildWorkspaceSkillSnapshot(await loadConfig());
      const skill = snapshot.skills.find(s => s.name === name);

      if (!skill) {
        console.error(`Skill not found: ${name}`);
        process.exit(1);
      }

      console.log(`# ${skill.emoji ?? '🔧'} ${skill.name}`);
      console.log(skill.description);
      console.log('');
      console.log(skill.content);
    });
}
```

## 9. 关键代码文件

| 文件 | 描述 |
|------|------|
| `src/agents/skills.ts` | 技能发现和快照构建 |
| `src/agents/skills/discovery.ts` | 技能目录扫描 |
| `src/agents/skills/parser.ts` | SKILL.md 解析 |
| `src/agents/skills/eligibility.ts` | 技能可用性检查 |
| `src/agents/skills/refresh.ts` | 技能刷新逻辑 |
| `src/infra/skills-remote.ts` | 远程技能获取 |
| `skills/` | 内置技能目录 |

## 10. 最佳实践

### 10.1 Skill 开发指南

1. **清晰的描述**：说明何时使用该技能
2. **具体示例**：提供常用命令示例
3. **明确限制**：说明何时不使用
4. **完整的元数据**：包括要求和安装说明
5. **简洁的内容**：避免冗余信息

### 10.2 示例模板

```markdown
---
name: my-skill
description: "One-line description"
metadata:
  openclaw:
    emoji: "🔨"
    requires:
      bins: ["my-cli"]
    install:
      - id: "brew"
        kind: "brew"
        formula: "my-cli"
        bins: ["my-cli"]
allowed-tools: ["bash"]
---

# My Skill

## When to Use

✅ **USE when:**
- You need to do X
- You need to do Y

❌ **DON'T use when:**
- You can use Z instead

## Common Commands

```bash
# Do something
my-cli command arg

# Do something else
my-cli other-command
```

## Notes

- Important note 1
- Important note 2
```
