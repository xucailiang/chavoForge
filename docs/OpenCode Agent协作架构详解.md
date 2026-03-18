# OpenCode Agent 协作架构详解

## 一、整体架构概览

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│                              客户端层                                        │
│   TUI (terminal)  │  Desktop (electron)  │  Web  │  SDK (programmatic)      │
└────────┬───────────────────┬──────────────────┬──────────┬───────────────────┘
         └───────────────────┴────────┬─────────┴──────────┘
                                      │ HTTP / WebSocket (Hono)
┌─────────────────────────────────────┼────────────────────────────────────────┐
│                              服务器层                                        │
│  ┌──────────────────────────────────┴─────────────────────────────────────┐  │
│  │                         Instance (项目实例)                            │  │
│  │  ┌─────────┐ ┌─────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐  │  │
│  │  │ Config  │ │ Provider│ │ Agent    │ │ Tool     │ │ Permission   │  │  │
│  │  │         │ │         │ │          │ │ Registry │ │              │  │  │
│  │  └─────────┘ └─────────┘ └──────────┘ └──────────┘ └──────────────┘  │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │                    Session 会话管理                              │  │  │
│  │  │  ┌───────────────────────────────────────────────────────────┐  │  │  │
│  │  │  │              SessionPrompt.loop() 主循环                  │  │  │  │
│  │  │  │                                                           │  │  │  │
│  │  │  │  User ──▶ Processor ──▶ LLM.stream ──▶ Tool Execute     │  │  │  │
│  │  │  │    │                         │                            │  │  │  │
│  │  │  │    │    SubtaskPart ─▶ TaskTool ─▶ 子会话 loop()         │  │  │  │
│  │  │  │    │    CompactionPart ─▶ Compaction ─▶ 摘要压缩         │  │  │  │
│  │  │  │    │                                                      │  │  │  │
│  │  │  │    └──────────────── 继续循环 ◀──────────────────────────│  │  │  │
│  │  │  └───────────────────────────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │  │
│  │  │ Snapshot │ │ MCP      │ │ Plugin   │ │ Skill    │ │ Command  │   │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘   │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │  Bus (事件总线)  │  Scheduler (定时任务)  │  Storage (SQLite/文件)     │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

## 二、职责模块详解

### 2.1 Instance — 项目实例管理

`src/project/instance.ts`

Instance 是整个运行时的根上下文。每个打开的项目目录对应一个 Instance，通过 Node.js `AsyncLocalStorage` 实现上下文隔离。所有模块通过 `Instance.state()` 创建实例级别的单例状态，Instance 销毁时自动清理。

核心职责：

- 通过 `AsyncLocalStorage` 提供 `directory`、`worktree`、`project` 三个上下文属性
- `Instance.state(init, dispose?)` 工厂方法：为每个模块创建与项目目录绑定的惰性单例，支持自定义销毁逻辑
- `Instance.provide()` 启动或复用实例上下文，内部维护 `Map<string, Promise<Context>>` 缓存
- `Instance.bind(fn)` 捕获当前 ALS 上下文，返回可在任意异步边界恢复上下文的包装函数
- `Instance.dispose()` / `Instance.disposeAll()` 清理状态和 Effect 层资源
- `Instance.reload()` 热重载：先销毁再重建，用于配置变更后刷新

设计要点：几乎所有有状态模块（Config、Agent、ToolRegistry、Bus、MCP、Skill 等）都通过 `Instance.state()` 注册，形成以项目目录为 key 的单例树。这使得多项目并行运行时彼此完全隔离。

### 2.2 Config — 分层配置系统

`src/config/config.ts`

Config 实现多层级配置合并，支持项目级、全局级、以及用户自定义目录的配置文件。

核心职责：

- 从多个目录加载 `.opencode/config.jsonc`（或 `config.json`），按优先级合并（项目级 > 全局级）
- 支持 JSONC 格式（带注释的 JSON），通过 `parseConfig()` 解析
- `Config.directories()` 返回所有配置搜索路径（项目 `.opencode/`、全局 `~/.config/opencode/` 等）
- `loadAgent(dir)` / `loadCommand(dir)` / `loadPlugin(dir)` / `loadMode(dir)` 从配置目录扫描自定义 agent、command、plugin、mode 定义
- `Config.get()` 返回合并后的完整配置对象，包含 provider、agent、command、permission、plugin、experimental 等字段
- `Config.update()` / `Config.updateGlobal()` 写回配置文件，使用 `patchJsonc()` 保留注释和格式
- `Config.waitForDependencies()` 确保 `bun install` 完成后再加载需要依赖的模块（plugin、custom tool 等）

设计要点：配置系统是声明式的，agent、command、permission 等均可通过配置文件定义或覆盖内置默认值。`mergeConfigConcatArrays()` 对数组字段采用拼接而非覆盖策略。

### 2.3 Provider — 多模型供应商抽象

`src/provider/provider.ts`

Provider 封装了对多个 LLM 供应商的统一访问接口，支持 Anthropic、OpenAI、Google、AWS Bedrock、Azure、Groq、xAI、Ollama、LM Studio、GitHub Copilot、GitLab 等十余种后端。

核心职责：

- 每个 Provider 定义 `id`、`name`、`getModel(sdk, modelID, options?)` 方法，返回 Vercel AI SDK 的 `LanguageModel`
- `Provider.list()` 聚合内置 + models.dev 远程模型列表 + 用户配置的自定义 provider
- `Provider.getModel(providerID, modelID)` 解析并返回完整的 `Model` 对象（含 capabilities、limits、pricing）
- `Provider.getLanguage(model)` 获取底层 AI SDK 语言模型实例
- `Provider.getSmallModel(providerID)` 获取同供应商的小模型（用于 title 生成等轻量任务）
- `Provider.defaultModel()` 根据配置和可用性选择默认模型
- `Provider.parseModel(str)` 解析 `"providerID/modelID"` 格式字符串

设计要点：Provider 层完全屏蔽了不同 SDK 的差异。`ProviderTransform` 负责处理各供应商特有的参数格式（如 Anthropic 的 cache control、OpenAI 的 Codex 模式、Gemini 的 thinking config 等）。

### 2.4 Agent — Agent 定义与注册

`src/agent/agent.ts`

Agent 是执行任务的角色抽象。每个 Agent 有独立的名称、模式、权限规则集、可选的专属模型和系统提示。

内置 Agent：

| 名称 | 模式 | 说明 |
| --- | --- | --- |
| `build` | primary | 默认主 agent，拥有完整工具权限 |
| `plan` | primary | 规划模式，禁止编辑工具，只能读取和写入计划文件 |
| `general` | subagent | 通用子 agent，用于并行执行多个独立任务 |
| `explore` | subagent | 代码探索专用，仅允许 grep/glob/read/bash/webfetch/websearch/codesearch 等只读工具 |
| `compaction` | primary (hidden) | 上下文压缩专用，无工具权限 |
| `title` | primary (hidden) | 会话标题生成，temperature=0.5 |
| `summary` | primary (hidden) | 会话摘要生成 |

核心职责：

- `Agent.Info` schema 定义：name、mode（primary/subagent/all）、permission（Ruleset）、model、prompt、steps、temperature 等
- `Agent.list()` 返回所有可用 agent（内置 + 用户配置），按 default_agent 排序
- `Agent.get(name)` 获取指定 agent
- `Agent.defaultAgent()` 返回默认主 agent 名称
- `Agent.generate()` 通过 LLM 根据自然语言描述自动生成 agent 配置
- 用户可通过 `config.agent` 覆盖内置 agent 的任意属性，或定义全新 agent

设计要点：Agent 的 `permission` 字段是一个 `Ruleset`（规则数组），由默认规则、用户全局规则、agent 特定规则三层合并而成。`mode` 字段决定 agent 是否可作为主 agent 或子 agent 使用。

### 2.5 ToolRegistry — 工具注册中心

`src/tool/registry.ts`

ToolRegistry 管理所有可用工具的注册、过滤和实例化。工具来源包括内置工具、用户自定义工具（JS/TS 文件）、Plugin 提供的工具，以及 MCP 服务器暴露的工具。

内置工具：bash、read、write、edit、glob、grep、task、webfetch、websearch、codesearch、skill、batch、apply_patch、question、todo 等。

核心职责：

- `ToolRegistry.state()` 在初始化时扫描配置目录下的 `tool/*.ts` 和 `tools/*.ts`，动态 import 并注册为自定义工具
- 同时加载所有 Plugin 导出的 `tool` 定义
- `ToolRegistry.tools(model, agent?)` 返回当前模型和 agent 可用的工具列表，内部执行过滤逻辑：
  - 根据模型 ID 决定使用 `edit`/`write` 还是 `apply_patch`（GPT 系列使用 patch 格式）
  - `websearch`/`codesearch` 仅对 opencode provider 或启用了 Exa 的用户开放
  - `batch` 工具需要 `experimental.batch_tool` 配置开启
- `ToolRegistry.register(tool)` 运行时动态注册工具
- 每个工具通过 `Tool.define(id, init)` 定义，`init` 返回 `{ description, parameters, execute }` 三元组
- 工具执行结果自动经过 `Truncate.output()` 截断处理，防止超长输出

设计要点：工具注册采用延迟初始化模式——`init` 函数接收 `agent` 上下文，可根据调用者动态调整工具描述和行为。MCP 工具在 `resolveTools()` 阶段与内置工具统一合并到同一个 tools map 中。

### 2.6 Permission — 权限控制系统

`src/permission/next.ts`

Permission 实现基于规则的权限控制，决定工具调用是否需要用户确认。

核心概念：

- `Rule`：单条规则，包含 `permission`（工具名或权限类别）、`pattern`（通配符匹配）、`action`（allow/deny/ask）
- `Ruleset`：规则数组，后面的规则覆盖前面的（last-match-wins）
- `Request`：权限请求，包含 permission、patterns、sessionID 等
- `Reply`：用户对权限请求的回复（allow/deny/always）

核心职责：

- `PermissionNext.evaluate(permission, pattern, ...rulesets)` 评估权限，返回匹配的规则
- `PermissionNext.ask(input)` 发起权限请求，阻塞等待用户回复
- `PermissionNext.fromConfig(permission)` 将配置格式转换为 Ruleset
- `PermissionNext.merge(...rulesets)` 合并多个规则集（简单 flat 拼接）
- `PermissionNext.disabled(tools, ruleset)` 返回被全局 deny 的工具集合
- 支持 `~` 和 `$HOME` 路径展开
- 特殊权限类别：`edit`（覆盖 edit/write/patch/multiedit）、`external_directory`、`doom_loop`、`question`、`plan_enter`/`plan_exit`

设计要点：权限规则从三个层级合并——Agent 默认规则、用户全局配置、Session 级别覆盖。`ask` 操作通过 Effect 服务层实现，底层对接 Question 模块弹出交互式确认。

### 2.7 Bus — 事件总线

`src/bus/index.ts`

Bus 是实例级别的发布/订阅事件系统，所有模块间的异步通信都通过 Bus 完成。

核心职责：

- `BusEvent.define(type, schema)` 定义类型安全的事件，schema 用 zod 描述
- `Bus.publish(def, properties)` 发布事件，同时通过 `GlobalBus.emit()` 广播到跨实例层
- `Bus.subscribe(def, callback)` 订阅特定事件，返回取消订阅函数
- `Bus.subscribeAll(callback)` 通配订阅所有事件
- `Bus.once(def, callback)` 一次性订阅，callback 返回 `"done"` 时自动取消
- 事件通过 `GlobalBus` 跨实例传播，服务器层通过 SSE/WebSocket 推送到客户端

设计要点：Bus 的 state 通过 `Instance.state()` 注册，实例销毁时自动触发 `InstanceDisposed` 事件并清理所有订阅。事件是 fire-and-forget 模式，publish 返回 `Promise.all(pending)` 但调用方通常不 await。

### 2.8 Plugin — 插件系统

`src/plugin/index.ts`

Plugin 提供生命周期钩子机制，允许第三方代码介入 agent 执行的各个阶段。

核心职责：

- 插件来源：内置插件（CodexAuthPlugin、CopilotAuthPlugin、GitlabAuthPlugin）、npm 包（通过 `BunProc.install()` 安装）、本地文件
- 每个插件导出一个 `(input: PluginInput) => Promise<Hooks>` 函数
- `PluginInput` 提供 `client`（SDK client）、`project`、`directory`、`worktree`、`serverUrl`、`$`（Bun shell）
- `Plugin.trigger(name, input, output)` 按顺序调用所有插件的同名钩子，钩子可修改 output 对象

可用钩子：

- `tool.definition` — 修改工具描述和参数 schema
- `tool.execute.before` / `tool.execute.after` — 工具执行前后拦截
- `chat.params` — 修改 LLM 调用参数（temperature、topP 等）
- `chat.headers` — 注入自定义 HTTP 头
- `chat.message` — 修改用户消息
- `experimental.chat.system.transform` — 修改系统提示
- `experimental.chat.messages.transform` — 修改消息历史
- `experimental.text.complete` — 修改模型文本输出
- `experimental.session.compacting` — 注入压缩上下文
- `command.execute.before` — 命令执行前拦截
- `shell.env` — 注入 shell 环境变量
- `auth` — 自定义认证
- `config` — 配置加载后回调
- `event` — 接收所有 Bus 事件

设计要点：插件通过 mutation output 对象实现拦截，而非返回值替换。这使得多个插件可以链式修改同一个对象。内置的 `BUILTIN` 插件列表会自动安装（除非 `OPENCODE_DISABLE_DEFAULT_PLUGINS`）。

### 2.9 MCP — Model Context Protocol 集成

`src/mcp/index.ts`

MCP 模块管理与外部 MCP 服务器的连接，将远程工具、Prompt、Resource 统一暴露给 agent。

核心职责：

- `MCP.create(key, mcp)` 根据配置创建 MCP 客户端连接（支持 stdio 和 SSE 两种传输）
- `MCP.tools()` 收集所有已连接 MCP 服务器的工具，转换为内部 `Tool` 格式
- `MCP.prompts()` 收集所有 MCP Prompt，转换为 Command 格式
- `MCP.resources()` 收集所有 MCP Resource
- `MCP.connect(name)` / `MCP.disconnect(name)` 管理单个服务器连接
- `MCP.status()` 返回所有服务器的连接状态
- `MCP.add(name, mcp)` 动态添加新的 MCP 服务器配置
- OAuth 认证支持：`startAuth()`、`authenticate()`、`finishAuth()`、`removeAuth()`
- `convertMcpTool()` 将 MCP 工具定义转换为 Vercel AI SDK 的 `tool()` 格式，处理超时和错误

设计要点：MCP 工具在 `SessionPrompt.resolveTools()` 中与内置工具合并，对 LLM 来说无差别。MCP 工具的执行结果同样经过 `Truncate.output()` 截断和 Plugin 钩子处理。服务器进程的生命周期由 Instance 管理，实例销毁时自动断开。

### 2.10 Snapshot — 文件快照系统

`src/snapshot/index.ts`

Snapshot 基于独立的 git 仓库（`$DATA_DIR/snapshot/{projectID}`）跟踪工作目录的文件变更，支持细粒度的撤销和差异查看。

核心职责：

- `Snapshot.init()` 初始化独立的 git 仓库（与项目 git 分离）
- `Snapshot.track()` 对当前工作目录做一次 `git add + git commit`，返回 commit hash
- `Snapshot.patch(hash)` 获取指定 commit 相对于前一次 commit 的文件变更列表
- `Snapshot.restore(hash)` 将工作目录恢复到指定 commit 的状态（`git checkout`）
- `Snapshot.revert(patches)` 批量撤销一组 patch 的文件变更
- `Snapshot.diff(hash)` / `Snapshot.diffFull(from, to)` 获取差异详情
- `Snapshot.syncExclude()` 同步项目 `.gitignore` 到快照仓库的 exclude 文件
- `Snapshot.cleanup()` 清理快照仓库

设计要点：每次工具执行前后都会调用 `Snapshot.track()`，Processor 在 `start-step` 和 `finish-step` 事件中记录快照。这使得每个工具调用的文件变更都可以独立追踪和撤销。快照 hash 存储在消息的 `step-start` / `step-finish` / `patch` Part 中。

### 2.11 SessionRevert — 会话回退

`src/session/revert.ts`

SessionRevert 利用 Snapshot 系统实现会话级别的回退功能。

核心职责：

- `SessionRevert.revert(input)` 将会话回退到指定消息/Part 位置：收集该位置之后所有 patch，调用 `Snapshot.revert()` 撤销文件变更
- `SessionRevert.unrevert(input)` 取消回退，恢复到回退前的状态（`Snapshot.restore()`）
- `SessionRevert.cleanup(session)` 确认回退：删除回退点之后的所有消息和 Part 记录
- 回退信息存储在 `Session.revert` 字段中，包含 messageID、partID、snapshot hash、diff 摘要

设计要点：回退是两阶段操作——先 revert（文件恢复但消息保留），用户确认后 cleanup（删除消息）。这允许用户在回退后反悔（unrevert）。

### 2.12 SessionCompaction — 上下文压缩

`src/session/compaction.ts`

SessionCompaction 处理上下文窗口溢出问题，通过摘要压缩历史消息来释放 token 空间。

核心职责：

- `SessionCompaction.isOverflow(input)` 检测当前 token 用量是否超过模型上下文限制（减去预留缓冲）
- `SessionCompaction.prune(input)` 修剪旧工具输出：从后向前扫描，保留最近 40000 token 的工具输出，标记更早的为 compacted
- `SessionCompaction.process(input)` 执行压缩：使用 `compaction` agent 生成历史摘要，摘要消息标记 `summary: true`
- `SessionCompaction.create(input)` 创建压缩请求：插入一条带 `CompactionPart` 的用户消息，主循环下次迭代时处理
- 压缩后如果是自动触发（`auto: true`），会自动插入 continue 消息让 agent 继续工作
- 支持 overflow 模式：当 LLM 返回 ContextOverflowError 时，压缩会尝试回放最后一条用户消息

设计要点：压缩是异步触发的——Processor 检测到溢出后返回 `"compact"`，主循环创建 CompactionPart，下次迭代时执行。`prune()` 在每次循环结束时运行，是轻量级的输出截断，与完整压缩互补。

### 2.13 Skill — 技能系统

`src/skill/skill.ts`

Skill 是可加载的指令集，通过 `SKILL.md` 文件定义，为 agent 提供特定任务的专业知识。

核心职责：

- 从多个位置扫描 `SKILL.md` 文件：
  - 外部目录：`.claude/skills/`、`.agents/skills/`（兼容 Claude Code）
  - OpenCode 目录：`.opencode/skill/`、`.opencode/skills/`
  - 配置的额外路径：`config.skills.paths`
  - 远程 URL：`config.skills.urls`（通过 DiscoveryService 下载）
- 每个 SKILL.md 包含 frontmatter（name、description）和 markdown 正文
- `Skill.all()` 返回所有已加载的技能
- `Skill.available(agent?)` 根据 agent 权限过滤可用技能
- `Skill.get(name)` 获取指定技能
- `Skill.fmt(list, opts)` 格式化技能列表（verbose 模式用 XML 标签，简洁模式用 markdown）

设计要点：Skill 通过 `SkillTool` 工具按需加载——agent 在系统提示中看到技能列表和描述，需要时调用 skill 工具加载完整内容。这避免了将所有技能内容塞入系统提示。技能名称全局唯一，项目级覆盖全局级。

### 2.14 Command — 命令系统

`src/command/index.ts`

Command 是用户可通过 `/` 前缀触发的预定义操作模板。

核心职责：

- 内置命令：`init`（创建/更新 AGENTS.md）、`review`（代码审查）
- 用户自定义命令：通过 `config.command` 定义，包含 template、agent、model、description、subtask 等字段
- MCP Prompt 自动转换为命令
- Skill 自动转换为命令（如果名称不冲突）
- 模板支持 `$1`、`$2`、`$ARGUMENTS` 占位符，以及 `` !`shell command` `` 内联 shell 执行
- `Command.hints(template)` 提取模板中的占位符列表
- subtask 命令会被包装为 `SubtaskPart`，在主循环中通过 TaskTool 执行

设计要点：命令的 `template` 字段使用 getter 实现惰性求值（MCP Prompt 需要异步获取）。subtask 模式允许命令在子 agent 中执行，不影响主会话的上下文。

### 2.15 InstructionPrompt — 指令加载

`src/session/instruction.ts`

InstructionPrompt 负责加载项目和全局级别的指令文件（AGENTS.md、CLAUDE.md 等），注入到系统提示中。

核心职责：

- 搜索指令文件：项目目录向上查找 `AGENTS.md` / `CLAUDE.md` / `CONTEXT.md`，全局目录查找 `~/.config/opencode/AGENTS.md` 和 `~/.claude/CLAUDE.md`
- 支持配置额外的指令路径（`config.instructions`），包括本地文件和 HTTP URL
- `InstructionPrompt.system()` 返回所有系统级指令内容
- `InstructionPrompt.resolve(messages, filepath, messageID)` 在读取文件时，自动查找该文件所在目录到项目根之间的 AGENTS.md 并加载（上下文感知指令）
- 通过 `claim` 机制避免同一消息重复加载同一指令文件

设计要点：指令加载是分层的——系统级指令在每次 LLM 调用时注入，目录级指令在 read 工具读取文件时动态发现并注入。这实现了"越靠近工作文件的指令越优先"的效果。

### 2.16 SystemPrompt — 系统提示构建

`src/session/system.ts`

SystemPrompt 组装发送给 LLM 的系统提示。

核心职责：

- `SystemPrompt.provider(model)` 根据模型类型选择基础提示模板（Anthropic、OpenAI/Beast、Gemini、Codex、Trinity 等）
- `SystemPrompt.environment(model)` 生成环境信息块：模型名称、工作目录、平台、日期等
- `SystemPrompt.skills(agent)` 生成可用技能列表（如果 agent 有 skill 权限）
- `SystemPrompt.instructions()` 返回 Codex 模式的 instructions 头

设计要点：系统提示由多个部分拼接而成：provider 基础提示 → 环境信息 → 技能列表 → InstructionPrompt 指令 → 用户自定义 system。Plugin 可通过 `experimental.chat.system.transform` 钩子修改最终结果。

### 2.17 SessionStatus — 会话状态

`src/session/status.ts`

SessionStatus 跟踪每个会话的运行状态。

- 三种状态：`idle`（空闲）、`busy`（执行中）、`retry`（重试等待，含 attempt、message、next 时间戳）
- `SessionStatus.set()` 更新状态并通过 Bus 发布 `session.status` 事件
- `SessionStatus.get()` / `SessionStatus.list()` 查询状态
- 状态存储在 Instance.state 中，不持久化

### 2.18 Question — 用户交互

`src/question/index.ts`

Question 模块处理 agent 执行过程中需要用户确认的场景。

- `Question.ask(input)` 发起问题请求，阻塞等待用户回复
- `Question.reply(input)` 用户提交回答
- `Question.reject(requestID)` 用户拒绝回答（抛出 RejectedError）
- `Question.list()` 列出所有待回答的问题
- 主要用于权限确认（Permission.ask 内部调用）和 agent 主动提问（QuestionTool）

### 2.19 Scheduler — 定时任务

`src/scheduler/index.ts`

Scheduler 提供简单的 `setInterval` 封装，支持实例级和全局级定时任务。

- `Scheduler.register(task)` 注册定时任务，包含 id、interval、run 函数、scope
- 实例级任务随 Instance 销毁自动清理
- 全局级任务（`scope: "global"`）跨实例共享，同 id 不重复注册
- timer 使用 `unref()` 不阻止进程退出

## 三、核心组件

### 3.1 Session 会话

Session 是一次完整对话的容器，存储在 SQLite 中。

```text
Session
├── id: SessionID (ULID)
├── parentID?: SessionID        ← 子会话指向父会话
├── title: string
├── permission: Ruleset         ← 会话级权限覆盖
├── revert?: { messageID, partID, snapshot, diff }
└── messages: MessageV2[]
    ├── User
    │   ├── agent: string       ← 使用的 agent 名称
    │   ├── model: { providerID, modelID }
    │   ├── tools?: Record<string, boolean>
    │   ├── format?: { type: "text" | "json_schema" }
    │   └── parts: (TextPart | FilePart | AgentPart | SubtaskPart)[]
    └── Assistant
        ├── agent: string
        ├── modelID / providerID
        ├── tokens: { input, output, reasoning, cache }
        ├── cost: number
        ├── finish: FinishReason
        ├── error?: NamedError
        ├── summary?: boolean   ← 压缩摘要消息标记
        └── parts: (TextPart | ReasoningPart | ToolPart | StepPart | PatchPart | CompactionPart | SubtaskPart)[]
```

### 3.2 Tool 工具

每个工具通过 `Tool.define(id, init)` 定义：

```typescript
Tool.define("example", async (initCtx) => ({
  description: "...",
  parameters: z.object({ ... }),
  async execute(args, ctx) {
    // ctx 提供: sessionID, messageID, abort, messages, metadata(), ask()
    return { title: "...", output: "...", metadata: {} }
  }
}))
```

`Tool.Context` 提供的能力：

- `ctx.sessionID` / `ctx.messageID` — 当前会话和消息 ID
- `ctx.abort` — AbortSignal，用于取消
- `ctx.messages` — 当前会话的完整消息历史
- `ctx.metadata(input)` — 更新工具执行状态（标题、元数据）
- `ctx.ask(req)` — 发起权限请求

## 四、Agent 协作执行流程

### 4.1 主循环 — SessionPrompt.loop()

```text
SessionPrompt.loop(sessionID)
│
├─ while (true)
│   ├─ 读取消息历史，找到 lastUser / lastAssistant / lastFinished
│   │
│   ├─ 检查是否有未完成的 finish（非 tool-calls）→ 退出循环
│   │
│   ├─ 扫描未处理的 SubtaskPart / CompactionPart
│   │
│   ├─ [SubtaskPart] ──────────────────────────────────────────┐
│   │   创建 assistant 消息                                     │
│   │   创建 ToolPart (tool="task", status="running")          │
│   │   调用 TaskTool.execute() ← 阻塞等待子会话完成           │
│   │   更新 ToolPart (status="completed", output=结果)        │
│   │   continue ← 回到循环顶部                                │
│   │                                                           │
│   ├─ [CompactionPart] ──────────────────────────────────────┐│
│   │   调用 SessionCompaction.process()                       ││
│   │   使用 compaction agent 生成摘要                         ││
│   │   如果 auto=true，插入 continue 消息                     ││
│   │   continue ← 回到循环顶部                                ││
│   │                                                           ││
│   ├─ [上下文溢出检测] ──────────────────────────────────────┐││
│   │   如果 lastFinished.tokens 超过模型限制                  │││
│   │   创建 CompactionPart → continue                         │││
│   │                                                           │││
│   └─ [正常处理] ─────────────────────────────────────────────┘││
│       创建 SessionProcessor                                    ││
│       解析工具列表 (resolveTools)                               ││
│       调用 processor.process() → LLM.stream()                  ││
│       返回 "continue" | "stop" | "compact"                     ││
│       continue / break / 创建 CompactionPart                   ││
│                                                                 ││
├─ 循环结束后调用 SessionCompaction.prune()                       ││
└─ 返回最后一条 assistant 消息                                    ││
```

### 4.2 Processor — SessionProcessor.process()

Processor 封装单次 LLM 调用的完整流处理：

1. 调用 `LLM.stream(input)` 获取流式响应
2. 遍历 `stream.fullStream`，按事件类型处理：
   - `reasoning-start/delta/end` → 创建/更新 ReasoningPart
   - `text-start/delta/end` → 创建/更新 TextPart
   - `tool-input-start` → 创建 ToolPart（status=pending）
   - `tool-call` → 更新 ToolPart（status=running），检测 doom loop（连续 3 次相同调用触发权限确认）
   - `tool-result` → 更新 ToolPart（status=completed）
   - `tool-error` → 更新 ToolPart（status=error），如果是权限拒绝则标记 blocked
   - `start-step` → 记录快照
   - `finish-step` → 计算 token 用量和费用，记录 patch，检测上下文溢出
3. 错误处理：
   - `ContextOverflowError` → 返回 `"compact"`
   - 可重试错误 → 指数退避重试（通过 SessionRetry）
   - 其他错误 → 记录到消息，返回 `"stop"`
4. 清理未完成的 ToolPart（标记为 error）
5. 返回 `"continue"` | `"stop"` | `"compact"`

### 4.3 LLM — LLM.stream()

LLM 层封装 Vercel AI SDK 的 `streamText` 调用：

1. 获取语言模型实例、配置、Provider 信息
2. 组装系统提示：agent prompt / provider prompt → 自定义 system → 用户 system
3. 通过 Plugin 钩子允许修改系统提示
4. 合并模型选项：base options → model options → agent options → variant options
5. 通过 Plugin 钩子允许修改 params 和 headers
6. 解析工具列表（根据权限过滤）
7. 调用 `streamText()` 并返回流

## 五、子 Agent 调用机制

### 5.1 TaskTool — 子 Agent 入口

`src/tool/task.ts`

```text
主 Agent (build)
│
├─ LLM 决定调用 task 工具
│   参数: { description, prompt, subagent_type, task_id? }
│
├─ TaskTool.execute()
│   ├─ 权限检查: PermissionNext.ask({ permission: "task", patterns: [subagent_type] })
│   ├─ 获取目标 Agent 定义
│   ├─ 创建子 Session (parentID = 父 sessionID)
│   │   └─ 注入权限限制:
│   │       - todowrite: deny
│   │       - todoread: deny
│   │       - task: deny (如果目标 agent 无 task 权限，防止无限嵌套)
│   │       - primary_tools: allow (实验性)
│   │
│   ├─ 调用 SessionPrompt.prompt() ← 触发子会话的完整 loop()
│   │   └─ 子 Agent 在独立 Session 中执行，拥有自己的消息历史
│   │
│   ├─ 提取子会话最后一条文本输出
│   │
│   └─ 返回结果:
│       output: "task_id: {sessionID}\n<task_result>\n{text}\n</task_result>"
│       ↑ task_id 用于后续恢复（传入 task_id 参数可复用子会话）
│
└─ 主 Agent 在 ToolPart.output 中看到 <task_result> 内容
    └─ 无法看到子 Agent 的中间工具调用和推理过程
```

### 5.2 关键特性

**同步阻塞**：TaskTool.execute() 内部 `await SessionPrompt.prompt()`，父循环在此处阻塞，直到子会话完成。

**结果传递**：只传递子会话最后一条 text 输出，包裹在 `<task_result>` 标签中。父 agent 看不到子 agent 的工具调用细节。

**会话恢复**：通过 `task_id` 参数可以恢复之前的子会话，子 agent 可以在之前的上下文基础上继续工作。

**权限隔离**：子会话注入额外的 deny 规则，防止子 agent 使用 todo 工具或无限嵌套创建子 agent。

## 六、并行 vs 串行执行

### 6.1 工具级并行（AI SDK 原生）

LLM 在单次响应中可以返回多个 tool_call，Vercel AI SDK 的 `streamText` 会并行执行这些工具调用。这是最常见的并行形式。

```text
LLM Response
├─ tool_call: read("file1.ts")  ─┐
├─ tool_call: read("file2.ts")  ─┼─ 并行执行
└─ tool_call: grep("pattern")   ─┘
```

### 6.2 BatchTool 显式并行

`src/tool/batch.ts`

BatchTool 是一个实验性工具，允许 agent 在单次工具调用中批量执行最多 25 个工具：

```text
batch({
  tool_calls: [
    { tool: "read", parameters: { filePath: "a.ts" } },
    { tool: "read", parameters: { filePath: "b.ts" } },
    { tool: "grep", parameters: { pattern: "TODO" } },
  ]
})
│
└─ Promise.all(toolCalls.map(executeCall))
   ├─ 每个调用独立创建 ToolPart
   ├─ 每个调用独立执行和记录结果
   └─ 不允许嵌套 batch
```

限制：不能批量执行 MCP 工具（外部工具需直接调用）、不能嵌套 batch、超过 25 个的调用会被丢弃并标记错误。

### 6.3 子 Agent 并行

当 LLM 在同一响应中返回多个 task 工具调用时，AI SDK 会并行执行它们：

```text
LLM Response
├─ tool_call: task({ subagent_type: "explore", prompt: "查找 API 路由" })  ─┐
├─ tool_call: task({ subagent_type: "explore", prompt: "查找数据模型" })  ─┼─ 并行
└─ tool_call: task({ subagent_type: "general", prompt: "分析依赖" })      ─┘
```

每个子 agent 内部是串行的（各自运行独立的 loop()），但多个子 agent 之间并行执行。

### 6.4 执行模型总结

```text
┌─────────────────────────────────────────────────────┐
│                    主循环 (串行)                      │
│                                                      │
│  step 1: Processor.process()                         │
│    └─ LLM.stream() → 工具调用 (并行)                │
│       ├─ tool_call_1 ──┐                             │
│       ├─ tool_call_2 ──┼── Promise.all (AI SDK)     │
│       └─ tool_call_3 ──┘                             │
│                                                      │
│  step 2: Processor.process()                         │
│    └─ LLM.stream() → task 调用 (并行)               │
│       ├─ task(explore) ──┐                           │
│       │   └─ 子 loop()  │                            │
│       └─ task(general) ──┼── Promise.all (AI SDK)   │
│           └─ 子 loop()  │                            │
│                          ↓                           │
│          各子 loop 内部串行                           │
│                                                      │
│  step 3: SubtaskPart 处理 (串行)                     │
│    └─ TaskTool.execute() ← 阻塞等待                 │
│                                                      │
│  step N: ...                                         │
└─────────────────────────────────────────────────────┘
```

## 七、上下文传递机制

### 7.1 父 → 子

- 子 agent 通过 `prompt` 参数接收任务描述（纯文本）
- 子 agent 不继承父会话的消息历史
- 子 agent 继承父 agent 的模型配置（除非目标 agent 有专属模型）
- 子 agent 的权限 = 目标 agent 权限 + 额外 deny 规则

### 7.2 子 → 父

- 只返回子会话最后一条 text 输出
- 包裹在 `<task_result>` 标签中
- 附带 `task_id`（子会话 ID），可用于后续恢复
- 父 agent 无法看到子 agent 的中间步骤

### 7.3 上下文压缩传递

当上下文溢出时，compaction agent 生成摘要，摘要消息标记 `summary: true`。后续 `MessageV2.toModelMessages()` 在遇到 summary 消息时会跳过之前的所有消息，只保留摘要 + 之后的消息。

## 八、权限隔离

```text
权限规则合并顺序（后者覆盖前者）：

1. Agent 默认规则 (PermissionNext.fromConfig({ "*": "allow", ... }))
2. 用户全局配置 (config.permission)
3. Agent 特定配置 (config.agent.{name}.permission)
4. Session 级别覆盖 (session.permission)

子 Agent 额外注入：
- todowrite: deny
- todoread: deny
- task: deny (如果目标 agent 无 task 权限)
- primary_tools: allow (实验性)
```

## 九、事件总线

```text
Bus 事件流：

模块 ──publish──▶ Bus (Instance 级)
                    ├──▶ 本地订阅者 (subscribe/subscribeAll)
                    └──▶ GlobalBus (跨实例)
                          └──▶ Server (HTTP/SSE/WebSocket)
                                └──▶ 客户端

关键事件：
- session.status          会话状态变更
- session.idle            会话空闲
- session.compacted       压缩完成
- message.updated         消息更新
- message.part.updated    Part 更新
- message.part.delta      Part 增量更新
- message.removed         消息删除
- command.executed        命令执行
- session.error           错误发生
- server.instance.disposed 实例销毁
```

## 十、设计模式总结

| 模式 | 应用位置 | 说明 |
| --- | --- | --- |
| AsyncLocalStorage 上下文 | Instance | 无需显式传参的项目级隔离 |
| 惰性单例 (Instance.state) | 所有有状态模块 | 按需初始化，自动清理 |
| 事件驱动 (Bus) | 模块间通信 | 解耦发布者和订阅者 |
| 规则链 (Permission) | 权限控制 | last-match-wins 的规则合并 |
| 流处理 (Processor) | LLM 交互 | 逐事件处理，支持中断和重试 |
| 两阶段回退 (Revert) | 文件变更 | revert → confirm/unrevert |
| 摘要压缩 (Compaction) | 上下文管理 | 自动检测溢出，异步压缩 |
| 插件钩子 (Plugin) | 扩展点 | mutation 模式，链式修改 |
| 子会话隔离 (TaskTool) | Agent 协作 | 独立会话 + 权限收窄 + 结果摘要 |
| 延迟初始化 (Tool.define) | 工具注册 | init 时注入 agent 上下文 |
| 声明式配置 (Config) | 全局配置 | 多层合并，JSONC 格式 |

## 十一、适合借鉴的场景

1. **多 Agent 协作系统**：TaskTool 的子会话模式提供了清晰的 agent 隔离和通信机制
2. **上下文窗口管理**：Compaction + Prune 双层策略值得参考
3. **权限控制**：基于通配符的规则链模式，灵活且可组合
4. **插件系统**：mutation-based 钩子模式，简单高效
5. **项目级隔离**：AsyncLocalStorage + Instance.state 模式，适用于多租户场景
6. **文件变更追踪**：独立 git 仓库做快照，支持细粒度撤销

## 十二、关键源码文件

| 文件 | 职责 |
| --- | --- |
| `src/project/instance.ts` | 项目实例管理，AsyncLocalStorage 上下文 |
| `src/config/config.ts` | 分层配置加载与合并 |
| `src/provider/provider.ts` | 多模型供应商抽象 |
| `src/agent/agent.ts` | Agent 定义、注册、配置合并 |
| `src/tool/tool.ts` | 工具基础抽象 |
| `src/tool/registry.ts` | 工具注册中心 |
| `src/tool/task.ts` | 子 Agent 调用（TaskTool） |
| `src/tool/batch.ts` | 批量并行工具（BatchTool） |
| `src/session/prompt.ts` | 主循环、工具解析、消息构建 |
| `src/session/processor.ts` | LLM 流处理、事件分发 |
| `src/session/llm.ts` | LLM 调用封装 |
| `src/session/compaction.ts` | 上下文压缩 |
| `src/session/revert.ts` | 会话回退 |
| `src/session/message-v2.ts` | 消息和 Part 数据结构 |
| `src/session/system.ts` | 系统提示构建 |
| `src/session/instruction.ts` | 指令文件加载 |
| `src/session/status.ts` | 会话状态跟踪 |
| `src/permission/next.ts` | 权限控制系统 |
| `src/bus/index.ts` | 事件总线 |
| `src/plugin/index.ts` | 插件系统 |
| `src/mcp/index.ts` | MCP 集成 |
| `src/snapshot/index.ts` | 文件快照 |
| `src/skill/skill.ts` | 技能系统 |
| `src/command/index.ts` | 命令系统 |
| `src/question/index.ts` | 用户交互 |
| `src/scheduler/index.ts` | 定时任务 |
