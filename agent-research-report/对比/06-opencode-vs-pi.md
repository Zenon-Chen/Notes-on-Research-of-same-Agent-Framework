# 成对比较：OpenCode vs pi 编程智能体

> **分析日期**：2026年5月
> **来源报告**：`03-opencode-analysis.md`、`04-pi-analysis.md`
> **方法论**：跨11个架构维度的结构化成对比较，以马尔可夫分解理论为基础。评分反映框架的架构设计质量，而非原始功能数量。

---

## (a) 执行摘要

OpenCode 和 pi 代表了构建 AI 编程智能体框架的两种不同哲学。两者均是基于 TypeScript、以 MIT 许可证发布的 monorepo，两者都实现了复杂的上下文管理和压缩机制，都可以被理解为**马尔可夫核**——将软件开发这一本质上非马尔可夫的问题分解为一系列有界、自包含的 LLM 决策。但它们实现这一目标的路径截然不同。

OpenCode（由 AnomalyCo 开发）是一个基于函数式编程原则构建的**企业级全栈平台**。其由 18 个包组成的 monorepo，以 Effect-TS 依赖注入、Turborepo 构建和 Bun 运行时为核心，提供从终端 TUI 到桌面应用，再到 Slack 集成和无服务器函数支持的完整功能。仅核心智能体引擎就跨越了 43 个目录模块。它将编程智能体不是视为一个单体程序，而是一个**可组合的代码操作系统**——配备 LSP 智能、tree-sitter AST 解析、ripgrep 搜索、发布-订阅事件总线，以及支持通配符匹配的基于规则的权限引擎。其通过 Vercel AI SDK 与 15 个以上的 LLM 提供商深度集成，每个提供商都有针对模型的系统提示。

pi（由 Mario Zechner / "badlogic" 开发）是一个**极简的自扩展架构**，基于"智能体不需要复杂；它们需要将非马尔可夫问题分解为马尔可夫状态"这一原则。其 5 个包的 monorepo 故意跳过了子智能体、计划模式和 LSP——仅搭载 7 个内置工具，秉持用户应该让 pi 自行构建任何所需功能的理念。压缩系统是**架构的核心**——一个基于模板的迭代摘要引擎，使用增量更新（`state_{t+1} = f(state_t, new_observations)`）而非从头重新摘要。JSONL 会话文件将智能体的完整状态轨迹外化，使会话变得可移植、可检查、可共享且可分支。

**核心张力**：OpenCode 追求的是**通过深度实现架构完整性**——LSP、事件溯源、权限扫描、子智能体和插件 SDK。pi 追求的是**通过极简实现架构清晰性**——仅压缩系统就有 740 行精心结构化的状态管理代码，整个智能体循环是一条清晰的 `transform → convert → LLM → execute → emit` 管道。OpenCode 给你一个即产品化的平台，学习曲线陡峭（Effect-TS、Turborepo、Drizzle ORM）；pi 给你一个干净的内核，任何有能力的 TypeScript 开发者都能在一个周末内理解。

**结论**：OpenCode 在**工具深度、提供商广度、代码智能（LSP）、权限复杂度和企业部署覆盖面**方面胜出。pi 在**压缩复杂度、会话可移植性、架构清晰度、扩展人机工程学和哲学一致性**方面胜出。两者是互补而非竞争关系——OpenCode 是你构建产品的基础框架；pi 是你嵌入自己智能体研究的内核。

---

## (b) 架构比较：18 包平台 vs 5 包内核

### OpenCode：全栈 Monorepo

OpenCode 的 18 包 monorepo 使用 Bun（运行时 + 包管理器）和 Turborepo（构建编排）进行管理：

| 层级 | 包 | 角色 |
|-------|----------|------|
| **核心引擎** | `opencode`（43 个 src/ 模块） | 智能体引擎：工具、会话、LLM、LSP、总线、权限、压缩、Effect 运行时 |
| **共享基础** | `core` | 文件系统抽象、ripgrep、日志、标志、npm 配置 |
| **UI 层** | `console`、`app`、`web`、`ui`、`storybook`、`desktop` | 终端 TUI（Hono）、Web 前端（SolidJS）、桌面包装器、组件库 |
| **可扩展性** | `plugin`、`sdk`、`extensions` | 插件 SDK、JS/TS 客户端 SDK、VS Code 扩展 |
| **企业级** | `enterprise`、`identity`、`containers`、`function`、`slack` | 认证、Docker 支持、无服务器、Slack 集成 |

依赖图遵循**分层六边形模式**：外部能力（LSP、文件系统、shell）在基于 Effect 的服务后面被抽象。事件总线提供会话变更、权限事件和 UI 更新之间的发布-订阅解耦。工具注册表在 LLM 工具调用和具体实现之间进行中介。

### pi：极简内核

pi 的 5 包 monorepo 使用 npm 工作空间，依赖图严格无环：

```
ai           （零内部依赖——基础层）
 ↑
agent        （依赖 ai）
 ↑
coding-agent （依赖 ai、agent、tui）
 ↑                ↑
web-ui             tui （无 pi 依赖——独立终端库）
```

每个包都有单一职责。`ai` 是多提供商 LLM 层（22+ 个提供商，17 个提供商实现）。`agent` 是纯粹的智能体运行时（`agent.ts` + `agent-loop.ts`）。`coding-agent` 负责 CLI、会话管理（约 2750 行的 `agent-session.ts`）、压缩（约 740 行）和工具。没有构建编排器，没有服务定位器库，没有 ORM——只有干净的 TypeScript 与 npm 工作空间。

### 对比分析

| 维度 | OpenCode | pi |
|-----------|----------|----|
| **包数量** | 18 | 5 |
| **构建工具** | Turborepo + Bun | npm 工作空间（tsc） |
| **运行时** | Effect-TS（Monadic） | 纯 TypeScript（命令式） |
| **状态层** | Drizzle ORM + SQLite | JSONL 文件（仅追加） |
| **服务模型** | Effect Layer DI | 带事件发射器的类 |
| **代码库感受** | 企业平台 | 研究级内核 |

**关键洞察**：OpenCode 的 18 个包编码了一种产品策略——多表面部署（CLI、Web、桌面）、企业需求（身份认证、容器、Slack）和第三方可扩展性（插件、SDK、扩展）。pi 的 5 个包编码了一种研究策略——每个包都是干净的抽象层，零循环依赖，整个系统可以装进一个开发者的脑子里。没有哪个"更好"——它们服务于不同的目标。OpenCode 优化的是产品表面积；pi 优化的是认知表面积。

---

## (c) 智能体编排：基于 Effect 的系统 vs 带事件发射器的智能体循环

### OpenCode：声明式智能体 + Effect 运行时

OpenCode 将智能体建模为**声明式配置**（`Agent.Info`）——名称、描述、模式（`primary` | `subagent` | `all`）、权限规则集、模型绑定、系统提示、温度：

```typescript
// 智能体是定义的，而非编程的
export const Info = Schema.Struct({
  name: Schema.String,
  mode: Schema.Literals(["subagent", "primary", "all"]),
  permission: Permission.Ruleset,
  model: Schema.optional(Schema.Struct({ modelID, providerID })),
  prompt: Schema.optional(Schema.String),
  steps: Schema.optional(Schema.Finite),
})
```

智能体还可以通过 `Agent.generate()` 进行**动态生成**——一个由 LLM 驱动的架构师，从自然语言描述中创建智能体规格。`Task` 工具将子智能体实例化为**子会话**，拥有自己的权限范围、SQLite 持久化消息和压缩策略。

执行流程经过 `SessionPrompt` → `LLM.stream()` → `SessionProcessor` → 工具执行 → 循环。`Runner` 抽象将执行建模为状态机：`Idle → Running → Shell → ShellThenRun → Running → Idle`。

Effect-TS 运行时提供类型化依赖注入、作用域资源管理（通过 `Effect.acquireRelease`）、结构化纤程并发和 OpenTelemetry 可观测性。一个关键的架构组件是**Bridge**——一种在基于 Promise 的异步代码和 Effect 纤程之间桥接的机制，同时保留 Effect 纤程本地存储无法跨 async/await 边界传播的每个实例的上下文。

### pi：带双层队列的事件驱动智能体循环

pi 将智能体建模为一个**事件发送类**，具有清晰的 `transform → convert → LLM → execute` 管道：

```
用户输入 → AgentMessage[] → transformContext() → convertToLlm() → LLM
                                 （可选裁剪）         （AgentMsg → LLM Msg）
                                                        ↓
                                                  LLM 响应（流式）
                                                        ↓
                                            从响应中解析工具调用
                                                        ↓
                                     ┌→ executeToolCalls() → 工具结果
                                     │         ↓
                                     │   发出生命周期事件
                                     │         ↓
                                     │   hasMoreToolCalls? → 循环
                                     │         │否
                                     │         ↓
                                     │   getSteeringMessages? ← 用户在运行中注入
                                     │         │
                                     └─ 有待处理？→ 继续循环
                                               │否：agent_end
```

**双层队列系统**尤其优雅：

1. **引导队列**（`steer()`）：在当前轮次完成工具执行后注入消息。用户在智能体运行时输入 `/model claude-sonnet-4`，消息会在不中止当前工作的情况下被注入会话。

2. **后续队列**（`followUp()`）：等待智能体自然停止的消息。用于跨会话链式工作。

每个队列可独立配置为 `"all"`（批量清空）或 `"one-at-a-time"`（顺序执行）。这使得用户可以在不破坏马尔可夫分解的情况下进行实时干预——新约束作为新的状态观测被注入。

事件系统发出丰富的生命周期流：`agent_start → turn_start → message_start → message_update* → message_end → tool_execution_start → tool_execution_end → turn_end → agent_end`。订阅者接收事件以及活动的 `AbortSignal`，实现跨 UI、扩展和核心循环的协调取消。

### 对比分析

| 维度 | OpenCode | pi |
|-----------|----------|----|
| **智能体定义** | 声明式 Schema + LLM 生成 | 内置 Agent 类，可扩展 |
| **子智能体** | 原生（`Task` 工具 + 子会话） | 设计上无（可扩展） |
| **队列** | 简单的轮次循环 | 双层：引导 + 后续 |
| **运行时** | Effect-TS（Monadic，类型化 DI） | EventEmitter（命令式，回调） |
| **状态机** | Runner（Idle/Running/Shell/ShellThenRun） | 隐式循环状态 |
| **取消操作** | Effect 纤程中断 | AbortSignal 传播 |
| **可观测性** | 通过 `@effect/opentelemetry` 的 OpenTelemetry | 类型化事件流 |

**关键洞察**：OpenCode 的 Effect-TS 基础提供了类型安全的错误处理、资源管理和结构化并发——但代价是陡峭的学习曲线，会过滤掉贡献者。pi 的事件驱动类模型对任何 JavaScript 开发者都立即可理解——但缺乏 Effect 提供的可组合性保证。OpenCode 是构建健壮产品的更好基础；pi 是快速实验和嵌入的更好基础。

---

## (d) 工具系统：丰富的注册表 + LSP vs 类型化操作接口

### OpenCode：带有深度代码智能的延迟初始化工具注册表

OpenCode 搭载 **17+ 个工具**，每个都实现为延迟的 `Tool.Info`，解析为强类型的 `Tool.Def`：

```
工具：bash、read、write、edit、apply_patch、grep、glob、lsp、
      task、todowrite、question、webfetch、websearch、skill、
      plan、truncate、invalid
```

工具执行流程受权限控制：`LLM 发出工具调用 → SessionProcessor 解析 Tool.Def → Permission.evaluate(rules, tool, patterns) → 如果 "ask"，等待用户 → Tool.execute(params, ctx) 在 Effect 运行时中 → 结果 → Bus 发出完成事件 → LLM 接收结果`。

**LSP 工具**是 OpenCode 最具特色的工具——它提供九种 LSP 操作（goToDefinition、findReferences、hover、documentSymbol、workspaceSymbol、goToImplementation、prepareCallHierarchy、incomingCalls、outgoingCalls）。这使 LLM 能够**符号化**地（什么调用了这个函数？什么实现了这个接口？）而不仅仅是文本化地（grep 函数名）推理代码。LSP 服务器按语言启动，配置在 `config/lsp.ts` 中，并作为带有重连逻辑的持久化进程进行管理。

**bash 工具**是最复杂的实现——超过 215 行的进程管理，包括通过 Bun 的 `node-pty` 支持 PTY，使用 web-tree-sitter 进行权限扫描，将命令解析为 AST 节点并将影响文件系统的操作映射到权限模式，以及 Windows 上的 SIGTERM → SIGKILL 升级与 tree-kill。

### pi：带有可插拔操作的类型化工具系统

pi 搭载 **7 个内置工具**，设计用于分层可扩展性：

```
AgentTool<TInput>：名称、描述、schema（TypeBox）、exec
     ↓
ToolDefinition<TI, TD>：渲染、创建、包装
     ↓
操作接口：BashOperations、ReadOperations、
          EditOperations、WriteOperations、
          FindOperations、GrepOperations、LsOperations
```

工具支持两种执行模式：**`"parallel"`**（默认——工具按顺序预检，然后并发执行）和 **`"sequential"`**（每个工具一次一个地准备、执行和完成）。这可以按工具配置。

**操作接口**是架构钩子——每个工具的执行逻辑抽象在一个可插拔接口后面，使得可以在不改变工具定义的情况下进行沙盒执行、远程执行和测试模拟。这比 OpenCode 中工具执行与 Effect 运行时紧密耦合的方式有更清晰的分离。

### 对比分析

| 维度 | OpenCode | pi |
|-----------|----------|----|
| **工具数量** | 17+ | 7 |
| **工具定义** | 延迟 `Tool.Info` → `Tool.Def` | `AgentTool<TInput>` + `ToolDefinition` |
| **执行模型** | 基于 Effect，顺序执行 | 顺序/并行，可按工具配置 |
| **代码智能** | LSP 集成（9 种操作） | 无（仅基于文本） |
| **操作抽象** | 与 Effect 运行时紧密耦合 | 可插拔操作接口 |
| **权限层** | 基于规则的通配符匹配 | 无（基于信任） |
| **命令扫描** | Tree-sitter AST 解析 | 无（纯字符串执行） |
| **类型安全** | Schema.Struct（Effect Schema） | TypeBox |

**关键洞察**：OpenCode 在工具深度上明显胜出——LSP 集成、tree-sitter 命令扫描和权限控制的执行为与代码库的关系创造了质的差异。pi 在工具执行模型上胜出——操作接口使得工具行为可以在不触及工具逻辑的情况下进行替换，并行执行配置更加明确。这些工具反映了项目的哲学：OpenCode 开箱即用提供功能丰富度；pi 提供干净的抽象，期望你去构建其余部分。

---

## (e) 状态管理：Effect Layer DI vs 可变 AgentState

### OpenCode：Effect 服务 + SQLite 持久化

OpenCode 的状态管理是多层的：

1. **Effect 服务模式**：所有能力都是带有基于 `Layer` 的 DI 的 `Context.Service<T>`。`InstanceState` 抽象通过 `ScopedCache` 提供按目录的服务缓存——像 Bus、LSP 和权限规则这样的服务被限定在项目目录范围内，并在销毁时清理。

2. **SQLite 持久化**：会话、消息、部件和摘要通过 Drizzle ORM 存储。消息使用类型化的 `Part` 系统（TextPart、ToolPart、ReasoningPart、FilePart、ImagePart、ErrorPart）。令牌追踪覆盖输入、输出、缓存读取、缓存写入和推理令牌。

3. **V2 事件溯源**：`v2/` 中的新兴架构使用 `EventV2.define()` 将会话和消息定义为版本化的事件流，暗示向 CQRS 迁移，以支持审计跟踪、离线同步和多实例协调。

4. **Effect Bridge**：Bridge 模块解决了跨 Promise-Effect 边界同时保留 `InstanceRef` 和 `WorkspaceRef` 上下文的关键问题：

```typescript
export const fromPromise = <T>(fn: () => Promise<T>): Effect.Effect<T> =>
  Effect.gen(function* () {
    const instance = yield* InstanceRef
    const workspace = yield* WorkspaceRef
    return yield* Effect.promise(() =>
      Promise.resolve(restore(instance, workspace, () => fn())))
  })
```

### pi：可变 AgentState + 仅追加 JSONL

pi 的状态管理更简单但更透明：

1. **可变 AgentState**：`AgentState` 类持有 `messages`、`tools`、`extensions`、`resources` 和 `abortController`——所有都是可变的，通过 getter/setter 对进行防御性复制。这是有意为之的简单：状态就是消息的集合。

2. **仅追加 JSONL**：会话文件是**外化的状态存储**。每条消息、工具调用、结果、压缩事件和分支点都作为 JSONL 行写入。文件可以任意大（无界历史），而 LLM 上下文是有界的。`buildSessionContext()` 从 JSONL 文件中确定性重建 LLM 可见的状态。

3. **树形结构历史**：`parentId` 链实现了分支。你可以在任何点分叉并探索替代方法。每个分支都是一个马尔可夫轨迹——一系列仅依赖于分支点状态加上后续轮次的状态。

4. **无数据库**：pi 没有 SQLite，没有 ORM，没有表迁移。状态持久化就是 JSONL 文件本身。这是一个根本性简化——没有模式迁移，没有查询优化，没有连接管理。代价是复杂查询（例如，"找到所有我编辑了 auth.ts 的会话"）需要扫描文件，而不是索引查询。

### 对比分析

| 维度 | OpenCode | pi |
|-----------|----------|----|
| **持久化** | Drizzle ORM + SQLite | 仅追加 JSONL |
| **状态粒度** | 类型化 Parts（Text、Tool、File、Image、Error） | AgentMessage（灵活类型） |
| **DI/服务模型** | Effect Layer DI | 无（纯类） |
| **会话分支** | Revert（完全撤销） | parentId 树（非破坏性） |
| **查询能力** | 通过 Drizzle 的 SQL 查询 | 顺序文件扫描 |
| **离线同步** | V2 事件溯源（新兴） | JSONL 文件传输 |
| **崩溃安全性** | SQLite WAL | 仅追加（最后一行可能损坏） |
| **审计跟踪** | 事件溯源（V2） | JSONL 文件就是审计跟踪 |
| **可移植性** | SQLite 文件（二进制） | JSONL 文件（文本，git 友好） |

**关键洞察**：OpenCode 的 SQLite + Effect 架构在复杂查询和生产部署方面更强大。pi 的 JSONL 方法更透明、可移植，且在外化状态方面更具哲学一致性——会话文件就是基本事实，任何工具都可以读取它（包括 git diff、grep、jq）。如果你需要构建一个具有用户账户、可搜索的会话历史和分析的产品，OpenCode 胜出。如果你希望将会话作为文本文件来检查、分享和分支，pi 胜出。

---

## (f) LLM 集成：Vercel AI SDK vs pi-ai 自定义包

### OpenCode：通过 Vercel AI SDK 利用生态系统

OpenCode 通过 **Vercel AI SDK**（`ai` v6.0.168）进行集成，配备 15+ 个 `@ai-sdk/*` 提供商包：OpenAI、Anthropic、Google、Amazon Bedrock、Azure、Mistral、Groq、xAI、DeepInfra、Cerebras、Cohere、Perplexity、TogetherAI、Alibaba——外加 OpenRouter（200+ 模型）和 GitLab AI 提供商。

`Provider` 服务管理模型发现（静态定义 + 动态 npm 加载）、API 密钥解析（环境变量、配置、云密钥存储）以及针对特定提供商的模型包装（例如，GitHub Copilot 的 Responses API for GPT-5）。`LLM.Service` 使用 SSE 超时检测和令牌计数包装 `ai.streamText()`。

一个独特的能力：**针对模型的系统提示**存储为文本文件（`anthropic.txt`、`gpt.txt`、`beast.txt`、`gemini.txt`、`codex.txt`、`kimi.txt`、`trinity.txt`）。每个提示都针对模型的特性进行了定制——Claude 得到与 GPT 不同的指令，GPT 得到与 Gemini 不同的指令。这反映了对不同模型在智能体循环中行为的深刻运营知识。

### pi：自定义多提供商层（`pi-ai`）

pi 的 `packages/ai` 是一个**自定义构建的多提供商 LLM 抽象**，支持 22+ 个提供商，对外部 AI SDK 零依赖。架构包括：

- **`api-registry.ts`**：一个中央注册表，提供商在其中注册 API 端点、认证和模型列表
- **`stream.ts`**：统一的流式接口，将提供商差异规范化为通用事件流
- **`types.ts`**：TypeBox 验证的 LLM 消息、工具调用和响应类型
- **17 个提供商实现**：每个都为其各自提供商实现流式 API（OpenAI、Anthropic、Google、Bedrock、DeepSeek、Mistral、Groq、Cerebras、Copilot 等）

关键优势：对 `@ai-sdk/*` 包零依赖。pi 的 LLM 层是自包含的，可以在不学习 Vercel AI SDK 抽象的情况下被理解、修改和审计。代价：pi 必须手动维护每个提供商集成，而 OpenCode 从 AI SDK 社区获得上游更新。

### 对比分析

| 维度 | OpenCode | pi |
|-----------|----------|----|
| **提供商数量** | 15+ 原生 + OpenRouter | 22+ 原生 |
| **SDK 依赖** | Vercel AI SDK（`ai` v6） | 无（自定义 `pi-ai`） |
| **针对模型的提示** | 7 个针对模型的文本文件 | 单一系统提示 |
| **提供商注册表** | 静态 + 动态 npm 加载 | 中央 `api-registry.ts` |
| **流式处理** | 包装的 `ai.streamText()` | 自定义 `stream.ts` 管道 |
| **认证解析** | 环境变量 + 配置 + 云密钥 | 环境变量 + 配置 |
| **OAuth 支持** | 通过 Vercel AI SDK | GitHub Copilot OAuth |

**关键洞察**：OpenCode 利用生态系统的力量——Vercel AI SDK 由一个团队维护，支持跨数十个提供商的流式处理、工具调用和结构化输出，并接收上游修复。pi 的方法提供完全控制，但带来维护负担。对于生产产品，OpenCode 的方法更具可持续性。对于需要理解 LLM 集成每一行的研究框架，pi 的方法更合适。

---

## (g) 上下文管理：两者都很复杂，方法截然不同

两个项目都实现了在架构上对其设计至关重要的复杂上下文管理。它们共享相同的核心洞察：LLM 上下文窗口是有界的，但软件开发对话是无界的。解决方案是**压缩**——将旧轮次压缩成保留基本状态的摘要。但它们的实现存在显著差异。

### OpenCode：三层裁剪 → 压缩 → 截断

OpenCode 的上下文策略有三层：

1. **裁剪**：工具输出截断为 2000 个字符（`TOOL_OUTPUT_MAX_CHARS`），完整输出存储在基于文件的溢出中，供截断工具按需访问。

2. **压缩**：当总令牌数超过 `usable = model.limit.input - reserved`（其中 `reserved = min(COMPACTION_BUFFER, maxOutputTokens)`，通常为 20K 令牌）时触发。算法保留：系统提示（始终）、最后 2 个尾部轮次（可配置）、`skill` 工具调用（受保护）、至少 2000 个令牌的近期上下文，以及一个**结构化摘要模板**：
   - 目标、约束与偏好、进展（已完成/进行中/受阻）
   - 关键决策、下一步、关键上下文、相关文件

3. **截断**：单个工具输出有硬限制（2000 行，50KB），带有基于文件的溢出存储和 7 天保留期。

关键常量：`PRUNE_MINIMUM = 20000`、`PRUNE_PROTECT = 40000`、`DEFAULT_TAIL_TURNS = 2`、`MIN_PRESERVE_RECENT_TOKENS = 2000`、`MAX_PRESERVE_RECENT_TOKENS = 8000`。

### pi：带有增量更新的迭代压缩

pi 的压缩系统是项目的**架构核心**——740 行精心结构化的状态管理：

1. **状态提取**：当上下文接近窗口限制时，**一次单独的 LLM 调用**从累积的消息中提取基本信息到结构化摘要中（从可能 450K 令牌的历史中提取约 2K 令牌）。

2. **状态注入**：摘要替换 LLM 输入中被丢弃的历史。压缩后，LLM 看到：`[compactionSummary] → [turn_prefix 如果轮次被分割] → [近期消息] → [近期工具结果]`。这是一个纯粹的马尔可夫状态——所有需要的信息都在摘要中，而非完整历史中。

3. **增量更新**（`UPDATE_SUMMARIZATION_PROMPT`）：当需要第二次压缩时，pi 不会从头重新摘要。它要求 LLM **将新信息合并到现有摘要中**：
   ```
   <previous-summary>## 目标：重构认证...</previous-summary>
   <new-messages>用户：现在添加刷新令牌...</new-messages>
   → 更新后的摘要，追加了刷新令牌的进展
   ```
   这是状态更新函数的直接类比：`state_{t+1} = f(state_t, new_observations)`。

4. **切断点逻辑**：该算法仔细选择切断位置，确保没有孤立的工具结果（在完整轮次单元后切断），保留 `keepRecentTokens`（20000）的近期上下文以保持连续性，并自动提取文件操作以供摘要使用。

5. **令牌估算**：pi 使用简单的 `chars/4` 启发式方法——比 OpenCode 的令牌计数更简单，但对非英语文本或代码密集型内容可能不够准确。

### 对比分析

| 维度 | OpenCode | pi |
|-----------|----------|----|
| **层级** | 3（裁剪、压缩、截断） | 1（压缩） |
| **压缩触发** | 令牌计数 vs 可用阈值 | chars/4 启发式 vs 窗口限制 |
| **摘要格式** | 结构化：目标、约束、进展、决策、下一步 | 结构化：目标、进展（已完成/进行中）、关键决策、关键上下文、文件 |
| **增量更新** | 否（从头重新压缩） | 是（UPDATE_SUMMARIZATION_PROMPT 将新内容合并到现有内容中） |
| **分割轮次处理** | 未提及 | 明确：在近期消息前保留轮次前缀 |
| **切断点逻辑** | 简单：最后 N 个尾部轮次 | 复杂：确保无孤立的工具结果，完整的轮次边界 |
| **溢出存储** | 基于文件，7 天保留期 | 不需要（JSONL 具有无界存储） |
| **令牌估算** | 准确（AI SDK 令牌计数） | 近似（chars/4 启发式） |
| **代码行数** | 约 500+ 行，分布在 compaction.ts + overflow.ts | 约 740 行，在专用 compaction/ 目录中 |

**关键洞察**：两个系统都很出色。OpenCode 的三层策略（在输出进入上下文之前裁剪、上下文溢出时压缩、带溢出存储的截断）更全面且经生产验证。pi 的**迭代更新机制**是真正优秀的创新——将新信息合并到现有摘要中而非从头重新摘要，保留了更多细微差别并降低了 LLM 成本。带有分割轮次处理的切断点逻辑也比 OpenCode 更简单的尾部轮次方法更小心。

---

## (h) 可扩展性：插件 SDK + 智能体生成 vs 扩展 + 技能

### OpenCode：三种扩展机制

1. **插件 SDK**（`packages/plugin`）：第三方扩展可以注册自定义工具、钩子和 UI 组件。SDK 提供类型化接口以集成到 Effect 运行时中。

2. **智能体生成**（`Agent.generate()`）：可以让 LLM 创建新的智能体定义——实际上，智能体可以从自然语言描述中架构自己的子智能体。这是元可扩展性：智能体扩展自身。

3. **技能**：带有 frontmatter 的 Markdown 文件，定义指令、上下文和工具绑定。可通过 `skill` 工具加载。技能是基于文件的、版本可控且可组合的。

4. **SDK**（`packages/sdk`）：一个 JS/TS 客户端 SDK，用于编程式智能体控制，使得可以将 OpenCode 嵌入其他应用。

### pi：深层生命周期钩子 + 基于文件的技能

1. **扩展**（TypeScript 模块）：扩展具有**深入到每个生命周期事件的钩子**。它们可以：
   - 修改工具执行（前后钩子）
   - 转换上下文（注入消息，裁剪历史）
   - 管理资源（文件附件、运行时提供商）
   - 渲染自定义 UI（Web UI 中的工具渲染器）
   - 添加自定义命令和主题
   每个扩展接收的上下文在会话替换时失效，防止陈旧状态错误。

2. **技能**（Markdown 文件）：带有 frontmatter 的技能，可通过 `/skill:name` 加载。这些与 OpenCode 的技能相似，但更紧密地集成到会话生命周期中。

3. **自扩展性**：pi 的核心哲学是用户应该让 pi **自身**来构建扩展。因为 pi 是一个编程智能体，它可以编写扩展自身行为的 TypeScript 模块——不需要单独的插件 SDK。

4. **可组合性**：扩展是 TypeScript 模块，技能是 markdown 文件，主题是 JSON。一切都是基于文件且版本可控的。

### 对比分析

| 维度 | OpenCode | pi |
|-----------|----------|----|
| **扩展模型** | 插件 SDK（类型化、基于 Effect） | TypeScript 模块（生命周期钩子） |
| **钩子深度** | 工具、权限、UI | 每个生命周期事件（消息、工具、上下文、资源、UI） |
| **智能体生成** | `Agent.generate()`（LLM 创建智能体） | 自扩展（pi 编写自己的扩展） |
| **技能** | Markdown + frontmatter | Markdown + frontmatter |
| **SDK** | JS/TS 客户端 SDK | RPC 协议（语言无关） |
| **UI 钩子** | 插件注册的组件 | 自定义工具渲染器（Lit Web Components） |
| **发现机制** | 无（手动） | 无（手动，通过 npm 或本地文件） |

**关键洞察**：两个项目都缺乏发现机制（无扩展市场）。OpenCode 的插件 SDK 提供更结构化的扩展面——类型化接口、Effect 集成——但需要理解 Effect-TS。pi 的生命周期钩子更深（钩入消息处理、上下文转换、工具执行）但形式化程度较低。pi 的自扩展哲学是最根本的区别：当智能体本身就是插件作者时，你不需要插件系统。

---

## (i) 马尔可夫分解分析：比较的最深层领域

### 理论重述

**马尔可夫分解理论**认为：针对大型代码项目的智能体框架通过将本质上非马尔可夫的系统（其中完整状态太大无法放入任何 LLM 上下文窗口）分解为一系列马尔可夫决策问题而成功，其中每个 LLM 轮次只看到一个精心构建的、自包含的、对当前决策足够的上下文。

OpenCode 和 pi 都在**架构层面体现了这一理论**——这不是偶然或优化，而是驱动其核心系统的明确设计哲学。这正是它们成为当前领域中两个最具理论一致性的智能体框架的原因。

### OpenCode：四层分解

OpenCode 在四个架构层实现马尔可夫分解：

**第一层：Bash 接口作为状态访问器**

bash 工具是状态分解的主要机制。每个 bash 命令都是一个查询，从完整项目状态中提取相关子集：
- `grep pattern ./src` → 仅从代码库中提取匹配行
- `git diff HEAD~1` → 仅提取更改的文件
- `ls -la src/` → 仅提取目录结构

每个查询将难以管理的巨大状态空间转换为可管理的马尔可夫观测：有界输出、确定性（相同输入 → 相同输出）、自描述。LLM 永远不需要"记住"项目状态——它通过工具查询状态。

**第二层：工具系统作为状态机**

每个工具都是一个确定性转换：给定输入参数，它产生有界输出。工具结果是添加到消息历史中的**不可变事实**——它们成为后续轮次的马尔可夫状态的一部分。`todowrite` 工具提供明确的状态追踪，使任务进展马尔可夫化。权限扫描确保工具不能在其定义范围之外产生副作用。

**第三层：压缩作为马尔可夫状态压缩**

当上下文窗口溢出时：旧轮次被丢弃（非马尔可夫的过去），结构化摘要替代它们（目标、约束、进展、决策、下一步、关键上下文、相关文件），LLM 在这个压缩状态上继续操作。这就是**马尔可夫性质在起作用**：未来仅依赖于当前（压缩的）状态，而非完整历史。

**第四层：实例状态作为自包含世界**

`InstanceState` 模式为每个项目目录创建一个限定范围、自包含的世界——它自己的 Bus、LSP 服务器、权限规则、shell 环境和文件系统状态。跨实例状态不能泄露，使每个项目成为一个独立的马尔可夫域。

**偏离纯马尔可夫分解的情况**：`skill` 工具引入了非确定性外部状态。`question` 工具引入了工具结果模型之外的人机循环状态。插件扩展可以引入任意持久状态。会话恢复允许跳回先前状态——本质上非马尔可夫。

### pi：三级分解（更明确、更具哲学性）

pi 的马尔可夫分解在文档中更明确，在设计上也更具哲学意图：

**第一级：轮次分解**

每次 LLM 调用：`f(system_prompt, messages, tools)` → `assistantMessage(text + toolCalls)`。LLM 从不推理"整个项目"——它推理当前轮次。工具调用创建观测：`read src/auth.ts` → `"文件内容..."` → 下一次 LLM 调用。这是一个**状态转换函数**：`S_{t+1} = LLM(S_t, tool_results)`。

**第二级：压缩分解**

压缩系统是 pi 的**标志性架构贡献**。它将无界对话历史（非马尔可夫，可能 450K 令牌）转化为结构化摘要（马尔可夫状态，约 2K 令牌）。迭代更新机制（`state_{t+1} = f(state_t, new_observations)`）是马尔可夫状态更新函数的直接数学类比。带有分割轮次处理的切断点逻辑确保干净的状态边界。

**第三级：会话树分解**

`parentId` 链实现分支：`fork → branch_summary → new trajectory`。每个分支都是一个马尔可夫轨迹——一系列仅依赖于分支点状态加上后续轮次的状态。这是 pi 最优雅的贡献：它使完整的**可能智能体轨迹空间**可作为 JSONL 文件树进行检查。

**pi 明确阐述而 OpenCode 隐含表达的内容**：

1. **外部状态存储**：JSONL 文件是智能体完整状态轨迹的外化、可检查和可共享的表示。OpenCode 的 SQLite 数据库服务于相同目的，但可检查性较差（二进制格式）且可移植性较差。

2. **状态重建为数学函数**：`buildSessionContext()` 从 JSONL 文件中确定性重建 LLM 可见的状态。给定相同的文件，你得到相同的上下文。OpenCode 的上下文构建更复杂，更难以推理。

3. **分支作为轨迹空间**：pi 明确将智能体的决策空间建模为马尔可夫轨迹树。OpenCode 的分支仅限于会话恢复（撤销），这是破坏性而非探索性的。

### 马尔可夫比较分析

| 维度 | OpenCode | pi |
|-----------|----------|----|
| **分解层级** | 4（Bash、工具、压缩、实例状态） | 3（轮次、压缩、会话树） |
| **主要分解机制** | Bash 接口（通用查询语言） | 压缩（显式状态提取） |
| **状态外化** | SQLite 数据库（二进制） | JSONL 文件（文本，git 友好） |
| **状态重建** | 隐式（Drizzle ORM 查询） | 显式（`buildSessionContext()` 作为纯函数） |
| **分支模型** | 会话恢复（破坏性撤销） | parentId 树（非破坏性分叉） |
| **迭代压缩** | 否（从头重新压缩） | 是（UPDATE_SUMMARIZATION_PROMPT） |
| **切断点逻辑** | 简单（最后 N 个尾部轮次） | 复杂（无孤立的工具结果，完整轮次边界） |
| **哲学明确性** | 由架构隐含 | 在设计文档中明确 |
| **偏离纯马尔可夫** | 技能、问题、插件、会话恢复 | 较少偏离（更简单的扩展模型） |

### 更深层的洞察

OpenCode 和 pi 通过不同的路径得出了相同的架构结论——智能体必须将非马尔可夫的软件开发分解为马尔可夫的 LLM 调用：

- **OpenCode** 通过**丰富的工具接口**来推广问题的解决方案：解决状态爆炸的方法不是更好地压缩状态，而是给 LLM 更好的工具来按需查询和修改状态。bash 接口、LSP 集成和 ripgrep 搜索就是答案。压缩是在即使是工具中介的上下文也溢出时的后备方案。

- **pi** 通过**状态压缩**来推广问题的解决方案：解决状态爆炸的方法是提取基本信息并丢弃其余部分，使用 LLM 本身作为压缩引擎。压缩系统不是后备方案——它是主要的架构机制。工具是将状态馈入压缩引擎的 I/O 边界。

两者都是正确的，也都是不完整的。OpenCode 无法处理 LLM 需要记住无法通过任何工具访问的东西的情况（例如，50 轮前做出的设计决策，未反映在任何文件或 git 提交中）。pi 无法处理 LLM 需要查询压缩摘要未捕获的状态的情况（例如，为什么在 30 轮前删除了某个特定的测试）。理想系统应将 pi 的迭代压缩与 OpenCode 的丰富工具访问结合起来——使用压缩作为语义记忆，使用工具作为程序性记忆。

---

## (j) 结论表

每个维度评分 1-10，其中 10 代表在所有已知智能体框架中为最佳。

| 维度 | OpenCode | pi | 胜者 | 评注 |
|-----------|:--------:|:--:|--------|------------|
| **架构清晰度** | 7 | 9 | **pi** | 18 个包加 Effect-TS DI vs 5 个包加严格无环依赖。pi 的架构可以在一个周末内理解；OpenCode 需要数周 |
| **工具广度** | 8 | 4 | **OpenCode** | 17+ 个工具，包括 LSP、网络搜索、网络获取、子智能体、规划 vs 7 个文本/代码工具 |
| **工具深度** | 9 | 5 | **OpenCode** | LSP 具有 9 种操作 + tree-sitter 命令扫描 + 权限控制。每个 OpenCode 工具都是一个子系统 |
| **代码智能** | 9 | 2 | **OpenCode** | LSP（转到定义、查找引用、调用层次）实现符号推理。pi 仅支持文本 |
| **LLM 提供商广度** | 8 | 9 | **pi** | 15+ 通过 AI SDK vs 22+ 自定义实现。pi 的提供商覆盖范围更广，但维护负担更高 |
| **LLM 集成深度** | 9 | 6 | **OpenCode** | 针对模型的系统提示、SSE 超时包装、AI SDK 生态系统杠杆 |
| **压缩复杂度** | 7 | 9 | **pi** | 三层（裁剪/压缩/截断）很全面，但 pi 的迭代更新 + 切断点逻辑 + 分割轮次处理真正更优秀 |
| **状态管理** | 8 | 7 | **OpenCode** | Effect Layer DI + SQLite + V2 事件溯源更强大；pi 的 JSONL 更透明 |
| **会话可移植性** | 4 | 9 | **pi** | SQLite 二进制 vs git 友好的 JSONL。pi 会话就是你分享和评估的产物 |
| **权限模型** | 9 | 1 | **OpenCode** | 基于规则的通配符匹配 + 多层 + shell 命令扫描。pi 没有权限层 |
| **可扩展性（易用性）** | 5 | 8 | **pi** | Effect-TS 插件 SDK 有陡峭的学习曲线。pi 扩展是带有生命周期钩子的纯 TypeScript 模块 |
| **可扩展性（能力）** | 8 | 7 | **OpenCode** | 插件 SDK + Agent.generate() + 技能 + SDK > 扩展 + 技能。但 pi 的钩子更深 |
| **智能体编排** | 8 | 7 | **OpenCode** | 子智能体 + 声明式智能体生成 + Task 工具。pi 的双层队列很优雅，但无子智能体 |
| **多表面部署** | 9 | 6 | **OpenCode** | CLI + Web + 桌面 + Slack + 容器 + 无服务器 vs CLI + Web UI |
| **企业就绪度** | 8 | 3 | **OpenCode** | 企业包 + 身份认证 + 容器 + Slack vs 无 |
| **代码质量** | 9 | 8 | **OpenCode** | 严格 TypeScript，无 `any`，Effect-TS 类型化错误，Drizzle 迁移，Turborepo 构建。pi 干净但更简单 |
| **马尔可夫分解** | 8 | 9 | **pi** | 两者都很出色。pi 的迭代压缩 + 会话树 + 状态外化更具哲学一致性 |
| **学习曲线** | 3 | 8 | **pi** | Effect-TS、Turborepo、Drizzle、Bun vs 普通 TypeScript。pi 显著更易接近 |
| **生产成熟度** | 8 | 5 | **OpenCode** | 200K+ 下载量、1.14.41 版本、多平台安装程序 vs 0.74.0 版本、仅 npm |
| **文档** | 7 | 5 | **OpenCode** | 20+ 语言文档 vs 内联代码注释 + README |
| **总体** | **7.4** | **6.1** | **OpenCode** | OpenCode 的广度在数字上胜出，但两个项目是互补而非竞争 |

**解读说明**：这些评分衡量的是架构质量和复杂度，而非"哪个对你的用例更好"。pi 在"工具广度"和"代码智能"上的较低分是**有意的设计选择**，而非缺陷。pi 明确选择省略 LSP、子智能体和计划模式；其哲学是用户应自行扩展智能体。OpenCode 较高的总分反映其更广的产品范围，而非更优越的架构。

---

## (k) 用例建议

### 在以下情况使用 OpenCode：

1. **你正在构建编程助手产品**。OpenCode 的 18 包 monorepo、插件 SDK 和 MIT 许可证使其成为构建商业产品的天然基础。Fork 它，添加你的工具，注册为插件，你就拥有一个即产品化的智能体框架，具有内置的多表面部署（CLI、Web、桌面、Slack）。

2. **你需要深度代码智能**。LSP 集成（转到定义、查找引用、调用层次、悬停提示、工作区符号）给予 OpenCode 与代码库在质上的不同关系。LLM 可以**符号化地**而不仅仅是文本化地推理代码。如果你的用例涉及需要理解调用图的大型复杂代码库，OpenCode 的 LSP 工具不可或缺。

3. **你需要细粒度权限**。基于规则的通配符权限引擎，通过 tree-sitter 进行 shell 命令扫描，这在开源编程智能体中是独一无二的。如果你需要在智能体不得删除文件、修改生产配置或访问受保护目录的环境中部署智能体，OpenCode 的权限模型就是答案。

4. **你需要特定的 LLM 提供商或需要 AI SDK 生态系统**。OpenCode 与 Vercel AI SDK 的集成意味着你可以获得上游修复、新提供商支持和 API 演进，而无需自己维护提供商代码。

5. **你需要企业级部署**。容器支持、无服务器函数、身份管理和 Slack 集成都内置于 monorepo 中。你可以将 OpenCode 部署为服务、桌面应用或 Slack 机器人——所有这些都共享相同的核心引擎。

6. **你重视企业级的 TypeScript**。核心引擎中有 43 个模块，Effect-TS 用于类型化错误处理，Drizzle 用于迁移，Turborepo 用于构建——这是一个为团队而非个人设计的代码库。

### 在以下情况使用 pi：

1. **你正在研究编程智能体或构建新型智能体系统**。pi 的架构如此干净且文档齐全，你可以在一个周末内理解整个系统。仅压缩系统就值得作为一种马尔可夫状态管理的参考实现来研究。JSONL 会话格式实现了对智能体行为的定量评估——分享会话进行分析，比较轨迹，衡量压缩质量。

2. **你需要最大的提供商灵活性且无 SDK 锁定**。pi 的 22+ 个自定义提供商实现让你独立于 Vercel AI SDK 生态系统。如果你需要支持不在任何 SDK 中的提供商，或想实验特定提供商的优化，pi 的方法更灵活。

3. **你重视会话可移植性和可检查性**。pi 的 JSONL 会话文件是该项目的杀手级特性。会话是文本文件——你可以 git diff 它们，grep 它们，用 jq 分析它们，在 Hugging Face 上分享它们，以及分叉它们以探索替代方法。没有其他编程智能体框架能如此透明地外化状态。

4. **你想将智能体嵌入你自己的应用**。pi 的 5 包结构且零循环依赖使其易于嵌入。导入 `@earendil-works/pi-agent-core` 和 `@earendil-works/pi-ai`，提供你自己的工具和 UI，你就拥有了一个智能体。RPC 协议是语言无关的——你可以构建一个 Python 或 Rust 前端来控制 pi 智能体。

5. **你拥抱自扩展哲学**。pi 的核心思想——用户应该让智能体自身来构建扩展——与 OpenCode 的插件 SDK 方法有根本不同。如果你想要一个可以扩展自身的智能体，pi 的架构原生支持这一点。

6. **你重视简单性和可学习性**。pi 没有 Effect-TS，没有 Turborepo，没有 Drizzle ORM，没有 Bun 依赖。它就是带有 npm 工作空间的普通 TypeScript。任何有能力的 TypeScript 开发者都可以贡献。心理模型是 `AgentMessage[] → transformContext → convertToLlm → LLM → execute tools → emit events → loop`。在如此简化的同时保持最佳压缩能力，这是一项架构成就。

### 综合：每个项目缺乏的东西

**OpenCode 应向 pi 学习**：迭代压缩更新（将新内容合并到现有内容中而非重新压缩）、带有分割轮次处理的切断点逻辑，以及基于 JSONL 的会话外化。一个将 OpenCode 的 LSP 智能和权限模型与 pi 的迭代压缩和会话可移植性相结合的混合体将是强大的。

**pi 应向 OpenCode 学习**：LSP 集成（最大的缺失能力）、权限建模（目前无护栏）以及子智能体编排（`Task` 工具模式干净且强大）。pi 的极简主义是优势，但这三个能力对于生产使用来说是明显的缺口。

**两者应互相学习**：它们对马尔可夫分解问题的不同答案。OpenCode 的答案是"给 LLM 更好的工具"——查询世界，不要记住它。pi 的答案是"更好地压缩状态"——提取基本信息，丢弃其余部分。最优系统应两者兼得：使用工具对无界项目状态进行程序性访问，使用压缩对智能体的累积经验进行语义压缩。

---

## 结论

OpenCode 和 pi 是开源领域中两个架构意图最为明确的 TypeScript 编程智能体框架。它们都对马尔可夫分解理论有着深刻的承诺，并通过根本上不同的机制来实现——OpenCode 通过丰富的工具接口和结构化压缩模板，pi 通过迭代状态压缩和外化的会话树。

它们之间的选择不在于哪个"更好"——而在于哪种哲学与你的目标一致。如果你认为通往更好的编程智能体的道路是更深的代码智能、更细粒度的权限和更广泛的部署面，OpenCode 是你的基础。如果你认为这条路是更干净的状态管理、更透明的会话表示和自扩展架构，pi 是你的内核。

两个项目都通过证明编程智能体的核心问题不是 LLM 能力——而是**状态分解**——来推进技术水平。能在软件的非马尔可夫世界和 LLM 推理的马尔可夫世界之间最干净地管理边界的智能体胜出。OpenCode 和 pi 都胜出了，只是方式不同。
