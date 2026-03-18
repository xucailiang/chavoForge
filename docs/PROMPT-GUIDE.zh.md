# OpenCode 提示词系统完全指南

本文档详细描述 OpenCode 的提示词（System Prompt）架构设计、组装流程、配置方法和修改思路。

## 一、提示词架构总览

OpenCode 的系统提示词不是一个单一字符串，而是由多个层次动态拼接而成的结构。最终发送给 LLM 的 system 消息由以下部分按顺序组装：

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    最终 system 消息（数组）                          │
│                                                                     │
│  system[0]:  以下所有内容 join("\n") 为一个字符串：                 │
│              ┌─ Agent Prompt 或 Provider Prompt（身份与行为指引）   │
│              ├─ 环境信息（模型名、工作目录、平台、日期）            │
│              ├─ 技能列表（Skill 描述，如果 agent 有 skill 权限）    │
│              ├─ 指令文件（AGENTS.md / CLAUDE.md 等）                │
│              ├─ 结构化输出指令（如需要）                            │
│              └─ 用户自定义 system（如果有）                         │
│                                                                     │
│  ── Plugin 钩子可修改 system 数组（追加、修改、删除元素）──        │
│  ── Plugin 修改后如果 system[0] 未变，会重组为 2 元素结构用于缓存 ─│
│                                                                     │
│  消息流中的动态注入（不在 system 中，而是插入到消息历史里）：       │
│              Plan 模式提醒（plan.txt / plan-reminder-anthropic.txt）│
│              Build 切换提醒（build-switch.txt）                     │
│              步数上限提醒（max-steps.txt）                          │
│              目录级 AGENTS.md（read 工具触发时动态加载）            │
└─────────────────────────────────────────────────────────────────────┘
```

## 二、提示词组装流程

### 2.1 入口：LLM.stream()

`packages/opencode/src/session/llm.ts`

这是所有 LLM 调用的统一入口。system 数组的组装逻辑如下：

```typescript
// 第一步：在 prompt.ts 的主循环中构建 input.system 数组
const skills = await SystemPrompt.skills(agent)
const system = [
  ...(await SystemPrompt.environment(model)),  // 环境信息（1个字符串）
  ...(skills ? [skills] : []),                 // 技能列表（0或1个字符串）
  ...(await InstructionPrompt.system()),       // 指令文件（多个字符串）
]
// 如果需要结构化输出，追加指令
if (format.type === "json_schema") system.push(STRUCTURED_OUTPUT_SYSTEM_PROMPT)

// 第二步：在 LLM.stream() 中，将 agent prompt 和 input.system 合并为一个字符串
const system = []
system.push(
  [
    // 优先级：agent.prompt > provider prompt
    ...(input.agent.prompt
      ? [input.agent.prompt]           // 如果 agent 定义了 prompt，直接使用
      : isCodex
        ? []                           // Codex 模式跳过（通过 options.instructions 传递）
        : SystemPrompt.provider(model) // 否则根据模型选择 provider 级别的 prompt
    ),
    ...input.system,                   // 环境信息 + 技能列表 + 指令文件
    ...(input.user.system ? [input.user.system] : []),  // 用户自定义 system
  ]
    .filter(x => x)
    .join("\n"),                        // 全部 join 为一个字符串，成为 system[0]
)

// 第三步：Plugin 钩子可修改 system 数组
await Plugin.trigger("experimental.chat.system.transform", { sessionID, model }, { system })

// 第四步：如果 Plugin 没有修改 system[0]，重组为 2 元素结构以利用 API 缓存
if (system.length > 2 && system[0] === header) {
  const rest = system.slice(1)
  system.length = 0
  system.push(header, rest.join("\n"))
}
```

关键逻辑：所有内容先 join 为一个字符串放入 `system[0]`。如果 agent 有 `prompt` 字段，provider prompt 被完全跳过。Plugin 可以向 system 数组 push 额外元素。

### 2.2 Provider Prompt 选择

`packages/opencode/src/session/system.ts` → `SystemPrompt.provider(model)`

根据 `model.api.id`（模型 ID 字符串）按顺序匹配，首个命中即返回：

```text
匹配条件（按顺序，首个命中即返回）    → 使用的 prompt 文件
────────────────────────────────────────────────────────────
model.api.id 包含 "gpt-5"            → codex_header.txt
model.api.id 包含 "gpt-" 或 "o1" 或 "o3" → beast.txt
model.api.id 包含 "gemini-"          → gemini.txt
model.api.id 包含 "claude"           → anthropic.txt
model.api.id（小写）包含 "trinity"   → trinity.txt
以上均不匹配                         → qwen.txt（PROMPT_ANTHROPIC_WITHOUT_TODO）
```

注意匹配顺序的影响：`gpt-5` 在 `gpt-` 之前检查，所以 GPT-5 使用 codex 而非 beast。`o1`/`o3` 与 `gpt-` 在同一条件中，使用 beast。

特殊情况：当 provider 为 OpenAI 且认证类型为 OAuth（Codex 会话）时，`SystemPrompt.provider()` 被完全跳过，改为通过 `options.instructions` 传递 `codex_header.txt` 的内容。

### 2.3 环境信息注入

`SystemPrompt.environment(model)` 生成环境上下文块（返回一个包含单个字符串的数组）：

```xml
You are powered by the model named claude-sonnet-4-5-20250929. The exact model ID is anthropic/claude-sonnet-4-5-20250929
Here is some useful information about the environment you are running in:
<env>
  Working directory: /path/to/project
  Workspace root folder: /path/to/project
  Is directory a git repo: yes
  Platform: darwin
  Today's date: Wed Mar 18 2026
</env>
<directories>
</directories>
```

注意：`<directories>` 块当前被硬编码跳过（`project.vcs === "git" && false`），始终为空。

### 2.4 技能列表注入

`SystemPrompt.skills(agent)` 生成可用技能列表（仅当 agent 有 skill 工具权限时）：

```text
Skills provide specialized instructions and workflows for specific tasks.
Use the skill tool to load a skill when a task matches its description.

<skills>
  <skill name="data-cleaning">数据清洗标准流程</skill>
  ...
</skills>
```

### 2.5 指令文件注入

`InstructionPrompt.system()` 加载以下位置的指令文件：

```text
搜索顺序（项目级，使用 findUp 从当前目录向上搜索到 worktree 根）：
  按 AGENTS.md → CLAUDE.md → CONTEXT.md 顺序尝试
  找到第一种类型即停止（break），不会同时加载 AGENTS.md 和 CLAUDE.md
  findUp 可能在多层目录中找到同类型的多个文件，全部加载

搜索顺序（全局级，找到第一个即停止）：
  1. OPENCODE_CONFIG_DIR/AGENTS.md（如果设置了环境变量）
  2. ~/.config/opencode/AGENTS.md
  3. ~/.claude/CLAUDE.md（除非 OPENCODE_DISABLE_CLAUDE_CODE_PROMPT）

额外指令（config.instructions 配置，全部加载，不互斥）：
  4. 本地文件路径（支持 glob 和 ~/ 展开）
  5. HTTP/HTTPS URL（5 秒超时，失败静默跳过）
```

指令文件的内容以 `"Instructions from: <path>"` 前缀注入到 system 中。每个文件作为 system 数组的一个独立字符串元素。

### 2.6 目录级指令动态加载

`InstructionPrompt.resolve()` 在 read 工具读取文件时触发：

当 agent 读取 `src/api/handler.ts` 时，系统会自动查找从 `src/api/` 到项目根之间每一层目录的 AGENTS.md，并将未加载过的指令注入到当前消息的上下文中。这实现了"越靠近工作文件的指令越优先"的效果。

### 2.7 消息流中的动态提醒

`insertReminders()` 在主循环中向消息历史（而非 system prompt）注入动态提醒。这些提醒作为 synthetic text part 附加到最后一条用户消息上：

```text
场景                              → 注入内容                        → 注入位置
──────────────────────────────────────────────────────────────────────────────
当前 agent 是 plan               → plan.txt（只读模式约束）        → 用户消息的 parts
从 plan 切换到 build             → build-switch.txt（解除只读）    → 用户消息的 parts
达到最大步数限制                 → max-steps.txt（禁止工具调用）   → 作为 assistant 消息追加
实验性 plan 模式开启时           → 动态生成的 plan 提醒（含计划文件路径）→ 用户消息的 parts
```

注意：这些提醒标记为 `synthetic: true`，不会持久化到数据库，仅在当前 LLM 调用中可见。

### 2.8 Plugin 钩子修改

`Plugin.trigger("experimental.chat.system.transform")` 允许插件在最终发送前修改 system 数组。此时 system 数组通常只有一个元素（所有内容已 join 为 `system[0]`）：

```typescript
// 插件可以 push 额外元素、修改现有元素、或替换整个数组
await Plugin.trigger(
  "experimental.chat.system.transform",
  { sessionID, model },
  { system },  // 可变引用，插件直接修改 system 数组
)

// Plugin 修改后的重组逻辑：
// 如果 Plugin 向 system push 了新元素（length > 2）且 system[0] 未被修改，
// 则将 system[1:] 合并为一个字符串，保持 2 元素结构以利用 Anthropic 的缓存机制
if (system.length > 2 && system[0] === header) {
  const rest = system.slice(1)
  system.length = 0
  system.push(header, rest.join("\n"))
}
```

最终 system 数组中的每个元素会被转换为一条 `{ role: "system", content: ... }` 消息发送给 LLM。

## 三、所有 Prompt 文件清单

### 3.1 Provider 级别提示词

位于 `packages/opencode/src/session/prompt/`：

| 文件 | 适用模型 | 核心特点 |
|------|----------|----------|
| `anthropic.txt` | Claude 系列 | 完整的 TodoWrite 任务管理、Task 工具委派策略、代码引用格式 |
| `beast.txt` | GPT-4o / o1 / o3 | 强调自主性、互联网研究、结构化工作流、Memory 文件 |
| `gemini.txt` | Gemini 系列 | 详细的工作流程（理解→计划→实现→验证）、绝对路径要求 |
| `codex_header.txt` | GPT-5 / Codex OAuth | 精简版，GPT-5 通过 provider 选择，Codex 通过 `options.instructions` 传递 |
| `trinity.txt` | Trinity 模型 | 极简风格、单工具调用、严格字数限制 |
| `qwen.txt` | 其他所有模型（兜底） | 变量名为 `PROMPT_ANTHROPIC_WITHOUT_TODO`，本质是无 TodoWrite 的 Anthropic 变体 |

未被 `SystemPrompt.provider()` 引用的存档文件：

| 文件 | 说明 |
|------|------|
| `copilot-gpt-5.txt` | GitHub Copilot GPT-5 的提示词参考，包含详细的结构化工作流和代码搜索指令 |
| `anthropic-20250930.txt` | Claude Code 原版提示词存档，包含 Sonnet 4.5 的环境信息硬编码，用于对比参考 |

### 3.2 Agent 级别提示词

位于 `packages/opencode/src/agent/prompt/`：

| 文件 | Agent | 用途 |
|------|-------|------|
| `explore.txt` | explore | 文件搜索专家，只读操作，返回绝对路径 |
| `compaction.txt` | compaction | 上下文压缩，生成对话摘要 |
| `title.txt` | title | 会话标题生成，≤50 字符，匹配用户语言 |
| `summary.txt` | summary | 会话摘要生成，类似 PR 描述 |

### 3.3 动态注入提示词

位于 `packages/opencode/src/session/prompt/`：

| 文件 | 触发条件 | 注入方式 |
|------|----------|----------|
| `plan.txt` | agent 为 plan 且实验性 plan 模式关闭 | 附加到用户消息 parts（synthetic） |
| `plan-reminder-anthropic.txt` | 实验性 plan 模式开启时进入 plan | 动态生成类似内容（含计划文件路径），附加到用户消息 parts |
| `build-switch.txt` | 从 plan 切换到 build | 附加到用户消息 parts（synthetic） |
| `max-steps.txt` | 达到 agent 步数上限 | 作为 assistant 消息内容追加到消息历史末尾 |

## 四、Agent 定义与 Prompt 的关系

### 4.1 内置 Agent 的 Prompt 配置

```text
Agent        prompt 字段     实际使用的 prompt
─────────────────────────────────────────────────────────
build        未设置          → 回退到 SystemPrompt.provider(model)
plan         未设置          → 回退到 provider prompt + plan.txt 动态注入
general      未设置          → 回退到 provider prompt
explore      PROMPT_EXPLORE  → 使用 agent/prompt/explore.txt
compaction   PROMPT_COMPACTION → 使用 agent/prompt/compaction.txt
title        PROMPT_TITLE    → 使用 agent/prompt/title.txt
summary      PROMPT_SUMMARY  → 使用 agent/prompt/summary.txt
```

关键发现：`build`、`plan`、`general` 三个最重要的 agent 都没有设置 `prompt` 字段，它们回退到 provider 级别的提示词。这意味着通过配置设置 `prompt` 字段就能完全替换它们的身份。

### 4.2 配置覆盖优先级

```text
最高优先级
    ↑  .opencode/agent/build.md 的正文内容
    │  config.jsonc 中 agent.build.prompt 字段
    │  Config.loadAgent() 从 .opencode/agents/*.md 加载
    │  ─── 以上任一存在，provider prompt 被完全跳过 ───
    │  SystemPrompt.provider(model) 根据模型选择
    ↓  内置 .txt 文件（anthropic.txt / beast.txt 等）
最低优先级
```

## 五、配置和修改 Prompt 的方法

### 方法一：Markdown Agent 文件（推荐）

在 `.opencode/agent/` 或 `.opencode/agents/` 下创建 `.md` 文件：

```text
.opencode/agent/
├── build.md        ← 覆盖主 agent 的 prompt
├── plan.md         ← 覆盖规划 agent
├── explore.md      ← 覆盖探索子 agent
├── general.md      ← 覆盖通用子 agent
└── reviewer.md     ← 定义全新的子 agent
```

文件格式：

```markdown
---
description: agent 的描述（显示在 @ 菜单中）
mode: primary          # primary | subagent | all
model: anthropic/claude-sonnet-4-5-20250929  # 可选，指定专用模型
temperature: 0.7       # 可选
steps: 50              # 可选，最大步数
hidden: false          # 可选，是否隐藏
color: "#FF5733"       # 可选，UI 颜色
permission:            # 可选，权限覆盖
  edit: deny
  bash: allow
---

这里是 prompt 正文。
文件名（不含 .md）即为 agent 名称。
正文内容会设置为 agent 的 prompt 字段。
```

优势：
- 纯文本，版本控制友好
- frontmatter 支持所有 Agent 配置字段
- 正文就是 prompt，直观易编辑
- 文件名即 agent 名，零配置关联
- 支持嵌套目录（如 `agents/team/reviewer.md`，名称为 `team/reviewer`）

### 方法二：config.jsonc 中定义

在 `.opencode/config.jsonc`（项目级）或 `~/.config/opencode/config.jsonc`（全局级）中：

```jsonc
{
  "agent": {
    "build": {
      "prompt": "你是一个通用 AI 执行引擎...",
      "description": "通用任务执行"
    },
    "explore": {
      "prompt": "你是一个文件探索专家...",
      "description": "文件搜索与探索"
    },
    // 定义全新 agent
    "reviewer": {
      "mode": "subagent",
      "prompt": "你是一个审核专家...",
      "description": "审核报告质量"
    }
  }
}
```

适合简短的 prompt 覆盖。长文本在 JSON 中不够友好，推荐用 Markdown 文件。

### 方法三：AGENTS.md 指令注入

AGENTS.md 不替换 system prompt，而是作为额外指令追加到 system 消息中。适合在保留默认 prompt 的基础上补充领域知识：

```markdown
你工作在一个数据分析项目中。

## 工作方式
- 使用 bash 执行 Python 分析脚本
- 数据文件在 data/ 目录下，不要修改原始数据
- 报告输出到 reports/ 目录

## 项目约定
- 所有分析脚本使用 pandas
- 图表使用 matplotlib，保存为 PNG
```

AGENTS.md 支持分层：项目根目录的全局生效，子目录的在 read 工具读取该目录文件时动态加载。

### 方法四：Plugin 钩子

通过 `.opencode/plugin/` 下的 TypeScript 文件实现最灵活的修改：

```typescript
// .opencode/plugin/prompt.ts
export default async function (input) {
  return {
    async "experimental.chat.system.transform"(_input, output) {
      // 在 system 末尾追加领域上下文
      output.system.push("当前项目的数据字典：\n" + await Bun.file("docs/dict.md").text())
    },
  }
}
```

适合需要动态生成 prompt 内容的场景（如从数据库读取、根据时间变化等）。

### 方法五：config.instructions 引入外部指令

```jsonc
{
  "instructions": [
    "docs/coding-standards.md",        // 本地文件
    "~/.config/opencode/global.md",    // 全局文件
    "**/.agents.md",                   // glob 模式
    "https://example.com/rules.md"     // 远程 URL
  ]
}
```

### 方法对比

```text
方法              替换 prompt?   动态?   适合场景
──────────────────────────────────────────────────────────────
Markdown Agent    是             否      完整重定义 agent 身份
config.jsonc      是             否      简短 prompt 覆盖
AGENTS.md         否（追加）     否      补充领域知识和约束
Plugin 钩子       可替换可追加   是      动态生成、条件注入
config.instructions 否（追加）   否      引入外部标准文档
```

## 六、Provider Prompt 的设计模式分析

通过对比所有 provider prompt，可以提炼出以下共性结构：

### 6.1 共性模块

所有 provider prompt 都包含以下模块（表述不同但语义一致）：

```text
1. 身份声明        "You are OpenCode / opencode, ..."
2. 语气风格        简洁、CLI 友好、不用 emoji、Markdown 格式
3. 安全约束        不猜测 URL、不暴露密钥
4. 任务执行流程    理解 → 计划 → 实现 → 验证
5. 工具使用策略    专用工具优先、并行调用、Task 委派
6. 代码引用格式    file_path:line_number
```

### 6.2 差异模块

不同 provider prompt 的差异主要在于：

```text
模块                anthropic    beast      gemini     trinity/qwen
──────────────────────────────────────────────────────────────────
TodoWrite 任务管理   ✓ 详细      ✓ 简略     ✗          ✗
互联网研究强调       ✗           ✓ 强烈     ✗          ✗
Memory 文件          ✗           ✓          ✗          ✗
结构化工作流         简略        ✓ 详细     ✓ 详细     ✗
输出长度限制         中等        宽松       中等       严格（<4行）
前端设计指引         ✗           ✓          ✗          ✗
新应用创建流程       ✗           ✗          ✓ 详细     ✗
```

### 6.3 设计启示

编写自定义 prompt 时，建议包含以下模块：

1. 身份声明 — 明确 agent 的角色和能力边界
2. 语气风格 — 适配 CLI 输出环境
3. 任务执行流程 — 提供清晰的工作步骤
4. 工具使用策略 — 指导工具选择和并行策略
5. 领域约束 — 特定于你的使用场景的规则

## 七、修改思路与最佳实践

### 7.1 通用化改造（去编码专用化）

核心思路：替换身份声明，保留工具使用策略。

```text
需要改的：
  ✗ "You are the best coding agent"  → "你是一个通用 AI 执行引擎"
  ✗ "software engineering tasks"     → "各类任务"
  ✗ 代码相关的具体指引               → 领域相关的指引

不需要改的：
  ✓ 工具使用策略（并行、Task 委派）— 这些是通用的
  ✓ 语气风格（CLI 简洁输出）— 这是界面约束
  ✓ 安全约束 — 通用适用
  ✓ TodoWrite 任务管理 — 通用适用
```

### 7.2 中文化改造

注意事项：
- title agent 的 prompt 中有规则 "use the same language as the user message"，会自动适配语言
- compaction 和 summary agent 的 prompt 是内部使用的，中文化优先级低
- 主 agent 和 explore 子 agent 的 prompt 中文化价值最高

### 7.3 多 Agent 协作优化

通过配置文件定义专门的子 agent，实现领域分工：

```text
.opencode/agent/
├── build.md        ← 主 agent：任务调度和执行
├── explore.md      ← 探索 agent：文件搜索和结构分析
├── analyst.md      ← 分析 agent：数据分析和统计
├── writer.md       ← 写作 agent：文档和报告生成
└── reviewer.md     ← 审核 agent：质量检查和验证
```

每个子 agent 通过 frontmatter 配置权限：

```markdown
---
mode: subagent
description: 数据分析专家，执行统计分析和数据可视化
permission:
  edit:
    "data/raw/*": deny    # 禁止修改原始数据
    "*": allow
  bash: allow              # 允许运行分析脚本
---
```

主 agent 通过 Task 工具自动委派任务给匹配的子 agent。

### 7.4 渐进式改造路径

```text
第一步（5 分钟）：
  创建 AGENTS.md，补充领域知识
  → 效果：默认 prompt 不变，但 agent 获得领域上下文

第二步（15 分钟）：
  创建 .opencode/agent/build.md，替换主 agent 身份
  → 效果：完全替换 provider prompt，使用自定义身份

第三步（30 分钟）：
  创建 explore.md、general.md 等子 agent 定义
  → 效果：所有 agent 使用统一的领域化 prompt

第四步（按需）：
  定义新的领域子 agent（analyst.md、writer.md 等）
  → 效果：实现领域分工的多 agent 协作

第五步（按需）：
  编写 Plugin 实现动态 prompt 注入
  → 效果：根据运行时状态动态调整 prompt
```

## 八、源码文件索引

| 文件 | 职责 |
|------|------|
| `src/session/llm.ts` | LLM 调用入口，system 数组组装 |
| `src/session/system.ts` | Provider prompt 选择、环境信息、技能列表 |
| `src/session/instruction.ts` | AGENTS.md 等指令文件加载 |
| `src/session/prompt.ts` | 主循环、动态提醒注入（plan/build-switch/max-steps） |
| `src/session/prompt/*.txt` | Provider 级别提示词文件 |
| `src/agent/agent.ts` | Agent 定义、配置合并 |
| `src/agent/prompt/*.txt` | Agent 级别提示词文件 |
| `src/config/config.ts` | 配置加载，loadAgent() 从 .md 文件加载 agent |
| `src/plugin/index.ts` | Plugin 钩子系统 |

所有路径相对于 `packages/opencode/`。
