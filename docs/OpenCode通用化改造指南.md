# OpenCode 通用化改造指南

## 核心认知：它已经是一个通用执行引擎

OpenCode 表面上是一个编码 agent，但其底层架构是领域无关的：

```text
┌─────────────────────────────────────────────────────────────┐
│                    看起来是编码专用的                         │
│                                                              │
│  bash / read / write / edit / grep / glob / snapshot         │
│                                                              │
│                    实际上是通用能力                           │
│                                                              │
│  shell 执行 / 文件读取 / 文件写入 / 文本编辑                │
│  内容搜索 / 文件发现 / 操作回滚                              │
└─────────────────────────────────────────────────────────────┘
```

- 文件系统是通用工作空间——写代码是操作文件，写报告、做分析、管理知识库也是操作文件
- Snapshot（git 快照）是通用回滚机制——任何 agent 都可能犯错，需要撤销
- Project/Instance 是工作目录上下文——任何任务都需要一个隔离的工作边界
- 子 agent 协作（TaskTool）是通用的任务分解模式——不限于编码

唯一"编码专用"的部分是默认的系统提示词（"you are a coding assistant"）。而这恰好是最容易替换的一层。

## 零代码改造路径

所有改造都通过配置、Skill、MCP、Plugin 完成，不需要修改源码。

### 第一层：替换 Agent 身份（必做）

通过 `.opencode/config.jsonc` 重新定义 agent 的角色：

```jsonc
{
  "agent": {
    // 覆盖默认 build agent 的提示词
    "build": {
      "prompt": "你是一个专业的数据分析助手。你可以读取数据文件、执行分析脚本、生成报告。使用 bash 工具运行 Python/R 脚本，使用 read/write 工具操作数据文件和报告文档。",
      "description": "数据分析与报告生成"
    },
    // 覆盖 plan agent
    "plan": {
      "prompt": "你是一个分析规划师。在执行分析前，先制定详细的分析计划，明确数据源、分析方法、预期产出。",
      "description": "分析计划制定"
    },
    // 覆盖 explore 子 agent
    "explore": {
      "prompt": "你是一个数据探索专家。快速浏览数据文件结构、检查数据质量、发现数据模式。",
      "description": "数据探索与发现"
    },
    // 定义全新的领域子 agent
    "reviewer": {
      "mode": "subagent",
      "prompt": "你是一个报告审核专家。检查报告的数据准确性、逻辑一致性、格式规范性。",
      "description": "审核报告质量和准确性"
    }
  }
}
```

或者更简单的方式——通过项目根目录的 `AGENTS.md`：

```markdown
你是一个数据分析助手，工作在一个数据项目中。

## 工作方式
- 使用 bash 执行 Python/R 分析脚本
- 使用 read 工具查看数据文件和已有报告
- 使用 write/edit 工具生成和修改报告
- 使用 grep 在数据文件中搜索特定模式
- 使用 glob 发现项目中的数据文件

## 注意事项
- 生成的报告放在 reports/ 目录下
- 数据文件在 data/ 目录下，不要修改原始数据
- 分析脚本放在 scripts/ 目录下
```

### 第二层：注入领域知识（Skill）

在 `.opencode/skills/` 下创建领域技能：

```text
.opencode/
└── skills/
    ├── data-cleaning/
    │   └── SKILL.md          # 数据清洗的标准流程和最佳实践
    ├── visualization/
    │   └── SKILL.md          # 可视化图表的生成规范
    ├── statistical-analysis/
    │   └── SKILL.md          # 统计分析方法选择指南
    └── report-template/
        └── SKILL.md          # 报告模板和格式规范
```

每个 SKILL.md 的格式：

```markdown
---
name: data-cleaning
description: 数据清洗标准流程，处理缺失值、异常值、格式统一
---

## 数据清洗流程

1. 使用 bash 运行 `python scripts/profile.py data/raw/` 生成数据概况
2. 检查缺失值比例，决定填充或删除策略
3. ...

## 常用脚本

- `scripts/clean.py` — 通用清洗脚本
- `scripts/validate.py` — 数据验证脚本
```

Agent 会在系统提示中看到技能列表，按需通过 skill 工具加载完整内容。

### 第三层：接入领域工具（MCP）

通过 `.opencode/config.jsonc` 配置 MCP 服务器，接入领域特定的外部能力：

```jsonc
{
  "mcp": {
    // 数据库查询
    "postgres": {
      "command": "uvx",
      "args": ["mcp-server-postgres", "--connection-string", "postgresql://..."],
    },
    // 浏览器自动化
    "browser": {
      "command": "uvx",
      "args": ["mcp-server-playwright"]
    },
    // 知识库检索
    "knowledge": {
      "command": "node",
      "args": ["./tools/knowledge-server.js"]
    },
    // API 网关
    "api": {
      "command": "python",
      "args": ["-m", "custom_mcp_server"]
    }
  }
}
```

MCP 工具会自动出现在 agent 的工具列表中，与内置工具无差别。

### 第四层：自定义轻量工具

在 `.opencode/tool/` 下用 TypeScript 定义领域工具：

```typescript
// .opencode/tool/query.ts
import { z } from "zod"

export default {
  description: "执行 SQL 查询并返回结果",
  args: {
    sql: z.string().describe("SQL 查询语句"),
    database: z.string().describe("数据库名称").default("analytics"),
  },
  async execute(args: { sql: string; database: string }, ctx: any) {
    // 直接用 Bun API 或调用外部命令
    const result = await Bun.$`psql -d ${args.database} -c ${args.sql} --csv`.text()
    return result
  },
}
```

### 第五层：定义领域命令

通过配置创建快捷命令：

```jsonc
{
  "command": {
    "analyze": {
      "description": "分析指定数据集",
      "template": "请分析 data/$1 数据集，生成完整的分析报告，包含数据概况、关键发现、可视化图表。报告输出到 reports/$1-report.md",
      "agent": "build"
    },
    "clean": {
      "description": "清洗数据文件",
      "template": "请对 data/raw/$1 执行数据清洗流程，输出到 data/cleaned/",
      "subtask": true
    },
    "daily": {
      "description": "生成每日报告",
      "template": "生成今天的数据日报。检查 data/daily/ 下最新的数据文件，与昨天对比，生成 reports/daily/!`date +%Y-%m-%d`.md"
    }
  }
}
```

用户通过 `/analyze sales-2024` 即可触发完整的分析流程。

### 第六层：权限适配（可选）

根据领域需求调整权限：

```jsonc
{
  "permission": {
    // 禁止修改原始数据
    "edit": {
      "data/raw/*": "deny",
      "*": "allow"
    },
    // 外部目录访问需确认
    "external_directory": {
      "*": "ask"
    },
    // bash 命令自动允许（数据分析场景频繁使用脚本）
    "bash": "allow"
  }
}
```

### 第七层：Plugin 深度定制（可选）

当配置层不够用时，Plugin 可以拦截几乎所有行为：

```typescript
// .opencode/plugin/domain.ts
import type { PluginInput, Hooks } from "@opencode-ai/plugin"

export default async function (input: PluginInput): Promise<Hooks> {
  return {
    // 在系统提示中注入领域上下文
    async "experimental.chat.system.transform"(_input, output) {
      output.system.push("当前项目的数据字典：\n" + await Bun.file("docs/data-dict.md").text())
    },

    // 工具执行前注入环境变量
    async "shell.env"(_input, output) {
      output.env.PYTHONPATH = "./scripts"
      output.env.DATA_DIR = "./data"
    },

    // 拦截写操作，确保不修改原始数据
    async "tool.execute.before"(input, output) {
      if (input.tool === "write" && output.args.filePath?.startsWith("data/raw/")) {
        throw new Error("禁止修改原始数据文件")
      }
    },
  }
}
```

## 领域适配示例

### 示例 1：数据分析工作站

```text
项目结构：
├── .opencode/
│   ├── config.jsonc          # agent 提示词 + MCP(postgres) + 命令
│   ├── skills/
│   │   ├── pandas/SKILL.md   # pandas 使用指南
│   │   └── plotting/SKILL.md # matplotlib/plotly 图表规范
│   └── tool/
│       └── query.ts          # SQL 快捷查询工具
├── AGENTS.md                 # "你是数据分析师..."
├── data/
│   ├── raw/                  # 原始数据（permission deny edit）
│   └── cleaned/              # 清洗后数据
├── scripts/                  # Python 分析脚本
└── reports/                  # 输出报告
```

内置工具的复用：

- `bash` → 执行 Python/R 脚本、运行 jupyter nbconvert
- `read` → 查看 CSV/JSON 数据文件、已有报告
- `write` → 生成 markdown 报告、Python 脚本
- `edit` → 修改分析脚本、更新报告
- `grep` → 在数据文件中搜索特定值或模式
- `glob` → 发现 `data/**/*.csv` 等数据文件
- `explore` 子 agent → 并行探索多个数据源
- `Snapshot` → 报告写错了可以回退

### 示例 2：知识库管理系统

```text
├── .opencode/
│   ├── config.jsonc          # agent 提示词 + MCP(向量数据库)
│   └── skills/
│       ├── taxonomy/SKILL.md # 分类体系规范
│       └── style/SKILL.md    # 写作风格指南
├── AGENTS.md                 # "你是知识库管理员..."
├── docs/                     # 知识文档
│   ├── guides/
│   ├── references/
│   └── tutorials/
└── templates/                # 文档模板
```

内置工具的复用：

- `read` → 阅读现有文档
- `write` → 创建新文档
- `edit` → 更新和修订文档
- `grep` → 全文搜索、查找重复内容、检查术语一致性
- `glob` → 按目录结构浏览文档
- `bash` → 运行 markdown lint、生成目录索引、调用翻译 API
- `general` 子 agent → 并行处理多篇文档
- `Snapshot` → 文档修改可回退
- `plan` 模式 → 大规模重组前先制定计划

### 示例 3：DevOps 运维助手

```text
├── .opencode/
│   ├── config.jsonc          # MCP(k8s, aws, monitoring)
│   ├── skills/
│   │   ├── k8s/SKILL.md     # Kubernetes 运维手册
│   │   ├── incident/SKILL.md # 故障排查流程
│   │   └── deploy/SKILL.md  # 部署检查清单
│   └── tool/
│       └── metrics.ts        # 监控指标查询工具
├── AGENTS.md                 # "你是 SRE 工程师..."
├── playbooks/                # 运维剧本
├── configs/                  # 基础设施配置
└── scripts/                  # 运维脚本
```

内置工具的复用：

- `bash` → 执行 kubectl、aws cli、terraform、ansible
- `read` → 查看配置文件、日志片段
- `write` → 生成运维脚本、更新配置
- `grep` → 在日志和配置中搜索
- `explore` 子 agent → 并行排查多个服务
- `Snapshot` → 配置变更可回退

### 示例 4：学术研究助手

```text
├── .opencode/
│   ├── config.jsonc          # MCP(arxiv, semantic-scholar)
│   └── skills/
│       ├── literature/SKILL.md  # 文献综述方法
│       └── latex/SKILL.md       # LaTeX 写作规范
├── AGENTS.md                 # "你是学术研究助手..."
├── papers/                   # 论文草稿
├── notes/                    # 阅读笔记
├── references/               # 参考文献
└── figures/                  # 图表
```

## 内置能力的通用性映射

```text
编码场景              →  通用能力              →  其他场景举例
─────────────────────────────────────────────────────────────
bash (执行命令)       →  shell 执行器          →  运行分析脚本、调用 API、执行部署
read (读文件)         →  内容读取器            →  查看数据、阅读文档、检查配置
write (写文件)        →  内容生成器            →  生成报告、创建文档、输出结果
edit (编辑文件)       →  内容修改器            →  修订文档、更新配置、调整数据
grep (搜索内容)       →  全文检索              →  搜索知识库、查找数据模式
glob (发现文件)       →  文件发现              →  浏览数据目录、列出文档
task (子 agent)       →  任务分解与并行        →  并行分析多个数据集
Snapshot (git 快照)   →  操作回滚              →  撤销错误修改
Compaction (压缩)     →  长对话管理            →  长时间分析任务的上下文管理
Permission (权限)     →  操作边界控制          →  保护原始数据不被修改
Skill (技能)          →  领域知识注入          →  分析方法、写作规范、运维手册
Command (命令)        →  快捷操作模板          →  /analyze、/report、/deploy
plan 模式             →  先规划后执行          →  大规模重组、复杂分析计划
explore 子 agent      →  先调研后行动          →  数据探索、文献调研
```

## 改造清单

按优先级排序：

```text
必做（10 分钟）：
  □ 创建 AGENTS.md，定义领域角色和工作方式

推荐（30 分钟）：
  □ 在 config.jsonc 中覆盖 agent prompt
  □ 定义 2-3 个常用命令
  □ 设置合适的 permission 规则

扩展（1-2 小时）：
  □ 编写领域 Skill（分析方法、写作规范等）
  □ 配置 MCP 服务器（数据库、API 等）
  □ 编写自定义工具（.opencode/tool/*.ts）

深度定制（按需）：
  □ 编写 Plugin 拦截特定行为
  □ 定义自定义子 agent
  □ 配置 skills.urls 引入远程技能库
```

## 提示词系统深度分析

### 当前架构：提示词的三层结构

通过源码分析，OpenCode 的系统提示词由三层拼接而成：

```text
┌─────────────────────────────────────────────────────────────┐
│  第一层：Agent Prompt（agent.prompt 字段）                   │
│  如果 agent 定义了 prompt → 直接使用                        │
│  如果没有 → 回退到 SystemPrompt.provider(model)             │
│            根据模型选择 anthropic.txt / beast.txt / gemini.txt│
├─────────────────────────────────────────────────────────────┤
│  第二层：自定义 system（input.system + user.system）         │
│  包括 InstructionPrompt（AGENTS.md）、环境信息、技能列表     │
├─────────────────────────────────────────────────────────────┤
│  第三层：Plugin 钩子（experimental.chat.system.transform）   │
│  插件可以修改最终的 system 数组                              │
└─────────────────────────────────────────────────────────────┘
```

关键发现：`build` agent 没有设置 `prompt` 字段，所以它回退到 provider 级别的提示词（如 anthropic.txt 中的 "You are OpenCode, the best coding agent on the planet"）。这意味着只要通过配置设置 `build.prompt`，就能完全替换这个"编码助手"身份。

### Main Agent 与 Sub Agent 的协作关系

```text
┌──────────────────────────────────────────────────────────────┐
│                    Main Agent (build)                         │
│                                                              │
│  身份：由 agent.prompt 或 provider prompt 定义               │
│  能力：完整工具权限（bash/read/write/edit/grep/glob/task）   │
│  职责：接收用户请求，执行或委派任务                          │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ TaskTool 调用 │  │ TaskTool 调用 │  │ TaskTool 调用 │       │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │
└─────────┼──────────────────┼──────────────────┼──────────────┘
          ▼                  ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐
│ general      │  │ explore      │  │ 自定义子 agent        │
│              │  │              │  │ (config 定义)         │
│ 独立子会话   │  │ 只读权限     │  │ 自定义权限和提示词    │
│ 完整工具权限 │  │ 搜索/阅读    │  │                       │
│ 结果摘要返回 │  │ 结果摘要返回 │  │ 结果摘要返回          │
└──────────────┘  └──────────────┘  └──────────────────────┘
```

协作特点：
- 父子之间通过 `<task_result>` 传递结果，父 agent 看不到子 agent 的中间步骤
- 子 agent 不继承父会话的消息历史，只接收 `prompt` 参数
- 子 agent 权限被收窄（deny todowrite/todoread/task），防止无限嵌套
- 多个子 agent 可并行执行（LLM 同时返回多个 task 工具调用时）

### 配置文件方式配置 System Prompt

当前架构已经完整支持通过配置文件替换所有 agent 的 system prompt，有三种方式：

#### 方式一：Markdown Agent 文件（推荐）

在 `.opencode/agent/` 下创建与 agent 同名的 `.md` 文件，frontmatter 定义元数据，正文即为 prompt：

```text
.opencode/
└── agent/
    ├── build.md        ← 覆盖主 agent 的 prompt
    ├── plan.md         ← 覆盖规划 agent
    ├── explore.md      ← 覆盖探索子 agent
    ├── general.md      ← 覆盖通用子 agent
    └── reviewer.md     ← 定义全新的子 agent
```

示例 `.opencode/agent/build.md`：

```markdown
---
description: 默认主 agent，通用任务执行与文件操作
---

你是 OpenCode，一个强大的通用 AI 执行引擎。

你是一个交互式 CLI 工具，帮助用户在工作目录中完成各类任务。
你的核心能力是文件读写、命令执行、内容搜索和任务分解——
这些能力不限于编程，同样适用于数据分析、文档管理、
知识整理、运维操作等任何需要操作文件和执行命令的场景。

# 语气与风格
- 回复简短精炼，显示在命令行界面中
- 用文本与用户沟通，工具仅用于执行任务
- 除非绝对必要，不创建新文件

# 工具使用策略
- 无依赖关系的工具调用应并行执行
- 优先使用专用工具而非 bash 命令
- 探索项目结构时使用 Task 工具委派给子 agent
```

优势：
- 纯文本，版本控制友好
- frontmatter 支持所有 Agent 配置字段（mode、model、permission 等）
- 正文就是 prompt，直观易编辑
- 文件名即 agent 名，零配置关联

#### 方式二：config.jsonc 中定义

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
    }
  }
}
```

适合简短的 prompt 覆盖，但长文本在 JSON 中不够友好。

#### 方式三：AGENTS.md 指令注入

AGENTS.md 不替换 system prompt，而是作为额外指令注入。适合在保留默认 prompt 的基础上补充领域知识：

```markdown
你工作在一个数据分析项目中。

## 工作方式
- 使用 bash 执行 Python 分析脚本
- 数据文件在 data/ 目录下，不要修改原始数据
- 报告输出到 reports/ 目录
```

### 三种方式的对比

```text
方式              替换 prompt?   适合场景                 优先级
─────────────────────────────────────────────────────────────
Markdown Agent    是             完整重定义 agent 身份    最高（config 层）
config.jsonc      是             简短 prompt 覆盖         config 层
AGENTS.md         否（追加）     补充领域知识和约束       instruction 层
```

### 推荐的通用化配置方案

对于将 OpenCode 改造为非编码场景的通用执行引擎，推荐组合使用：

1. `.opencode/agent/build.md` — 用中文重新定义主 agent 身份，使用通用化表述
2. `.opencode/agent/explore.md` — 用中文重新定义探索子 agent
3. `AGENTS.md` — 补充项目特定的领域知识和工作规范
4. `.opencode/config.jsonc` — 配置权限、命令、MCP 等

这样做到了：
- 零源码修改
- 所有 prompt 可版本控制
- 主 agent 和子 agent 的身份独立可配置
- 领域知识与 agent 身份分离

## 总结

OpenCode 的架构本质是：

```text
通用 agent 循环 + 文件系统操作 + shell 执行 + 子 agent 协作 + 可扩展工具
```

这套组合在任何"需要 AI 在一个工作目录中读写文件、执行命令、完成任务"的场景下都成立。编码只是它的默认人格，不是它的能力边界。通过配置层的调整（Agent prompt + Skill + MCP + Command + Permission），可以在零代码改动的前提下适配到数据分析、知识管理、运维自动化、学术研究等任意领域。

提示词系统的三层结构（Agent Prompt → Instruction → Plugin）提供了从"完全替换身份"到"轻量补充知识"的灵活度。`.opencode/agent/*.md` 是最推荐的配置方式——纯文本、版本控制友好、支持完整的 agent 元数据定义。
