# OpenCode Agent 框架：综合分析

## (a) 项目概览

OpenCode 是由 AnomalyCo 开发的开源 AI 编程智能体（仓库：`github.com/anomalyco/opencode`）。它以 npm 包（`opencode-ai`）和独立 CLI 两种形式发布，可通过 npm、brew、scoop、choco、pacman 和 nix 安装。在版本 `1.14.41` 时，它已累积超过 200,000 次总下载量（GitHub + npm），并且其文档支持 20 多种语言。

该项目是一个大型 TypeScript 单仓库（monorepo），使用 **Bun**（v1.3.13）作为包管理器和运行时，**Turborepo** 用于构建编排，以及 **Effect**（v4.0.0-beta.59）作为核心函数式效应库。它包含 **18 个包**，涵盖全栈：

| 包 | 用途 |
|---------|---------|
| `packages/opencode` | 核心智能体引擎：工具注册表、会话管理、LLM 集成、LSP、总线、权限系统 |
| `packages/core` | 共享工具：文件系统抽象、ripgrep 集成、日志、标志位、npm 配置 |
| `packages/console` | 基于终端的 TUI（包含多个子包，用于 app 等） |
| `packages/app` | Web 应用前端 |
| `packages/web` | Web 前端资源 |
| `packages/desktop` | 桌面应用（Electron/Tauri 封装） |
| `packages/ui` | 共享 UI 组件库 |
| `packages/storybook` | UI 组件开发环境 |
| `packages/plugin` | 第三方扩展的插件 SDK 接口 |
| `packages/sdk` | 用于编程式控制智能体的 JavaScript/TypeScript SDK |
| `packages/containers` | 容器/Docker 部署支持 |
| `packages/script` | 构建和维护脚本 |
| `packages/slack` | Slack 集成 |
| `packages/identity` | 认证和身份管理 |
| `packages/enterprise` | 企业功能 |
| `packages/function` | 无服务器函数支持 |
| `packages/docs` | 文档 |
| `packages/extensions` | 编辑器扩展（VS Code 等） |

代码库规模相当庞大——仅 `packages/opencode/src` 就包含 **43 个目录模块**，涵盖智能体编排、工具系统、LLM 提供者抽象、会话管理、LSP 集成、发布-订阅总线、权限评估、基于 Effect 的运行时管理、Shell 执行、文件系统操作等。

## (b) 架构概览

### 单仓库结构

```
opencode/
├── packages/
│   ├── opencode/       # 核心智能体引擎（43 个 src/ 模块）
│   │   └── src/
│   │       ├── agent/          # 智能体定义和生成
│   │       ├── tool/           # 工具注册表和实现（39 个文件）
│   │       ├── provider/       # LLM 提供者抽象层
│   │       ├── session/        # 会话状态、压缩、LLM 管道
│   │       ├── effect/         # Effect 运行时、桥接、实例状态
│   │       ├── bus/            # 发布-订阅事件总线
│   │       ├── lsp/            # 语言服务器协议集成
│   │       ├── permission/     # 基于规则的权限评估
│   │       ├── config/         # 配置管理（22 个模块）
│   │       ├── shell/          # Shell 执行和进程管理
│   │       ├── file/           # 文件系统和 ripgrep 集成
│   │       ├── v2/             # V2 架构（事件溯源、模式）
│   │       └── ...             # mcp, sync, storage, auth, plugin 等
│   ├── core/           # 共享运行时工具
│   ├── plugin/         # 扩展 SDK
│   ├── sdk/            # 客户端 SDK
│   ├── ui/             # 组件库（SolidJS）
│   ├── console/        # 终端 UI（基于 Hono）
│   └── ...             # app, web, desktop, containers 等
```

### 依赖关系图（核心引擎）

```
                  ┌──────────┐
                  │  Config  │ （用户/工作区设置）
                  └─────┬────┘
                        │
  ┌──────────┐    ┌─────▼─────┐    ┌────────────┐
  │ Provider ├───►│  Session  │◄───┤ Permission │
  │ (LLM)    │    │  (状态)   │    │  (规则)    │
  └────┬─────┘    └─────┬─────┘    └────────────┘
       │                │
  ┌────▼─────┐    ┌─────▼─────┐    ┌────────────┐
  │   LLM    │    │  Session   │    │    Bus     │
  │ (stream) │◄───┤ Processor  │───►│  (PubSub)  │
  └──────────┘    └─────┬─────┘    └────────────┘
                        │
                  ┌─────▼─────┐    ┌────────────┐
                  │  Effect   │    │   Agent    │
                  │  Runner   │    │   Defs     │
                  └─────┬─────┘    └────────────┘
                        │
                  ┌─────▼─────┐    ┌────────────┐
                  │   Tool    │◄───│   Shell    │
                  │ Registry  │    │  (bash)    │
                  └─────┬─────┘    └────────────┘
                        │
       ┌────────────────┼────────────────┐
       │                │                │
  ┌────▼─────┐    ┌─────▼─────┐    ┌─────▼─────┐
  │   LSP    │    │ FileSystem│    │  Ripgrep  │
  │ (智能)   │    │ (AppFS)   │    │  (搜索)   │
  └──────────┘    └───────────┘    └───────────┘
```

该架构遵循**分层六边形模式**：外部能力（LSP、文件系统、Shell）通过基于 Effect 的服务进行抽象，总线提供发布-订阅解耦，工具注册表在 LLM 工具调用和具体实现之间进行中介。

## (c) 智能体编排模型

### 智能体生命周期

OpenCode 中的智能体由 `Agent.Info` 定义——这是一种声明式配置，包含名称、描述、模式（`primary`、`subagent`、`all`）、权限规则、模型绑定、系统提示词以及可选的 temperature/topP 设置：

```typescript
// 来自 agent/agent.ts 的模式（简化）
export const Info = Schema.Struct({
  name: Schema.String,
  description: Schema.optional(Schema.String),
  mode: Schema.Literals(["subagent", "primary", "all"]),
  native: Schema.optional(Schema.Boolean),
  hidden: Schema.optional(Schema.Boolean),
  topP: Schema.optional(Schema.Finite),
  temperature: Schema.optional(Schema.Finite),
  color: Schema.optional(Schema.String),
  permission: Permission.Ruleset,
  model: Schema.optional(Schema.Struct({
    modelID: ModelID,
    providerID: ProviderID,
  })),
  variant: Schema.optional(Schema.String),
  prompt: Schema.optional(Schema.String),
  options: Schema.Record(Schema.String, Schema.Unknown),
  steps: Schema.optional(Schema.Finite),
})
```

智能体生命周期如下：

1. **定义**：智能体通过 YAML/JSON 配置定义，或通过 `Agent.generate()` 动态生成——这是一个由 LLM 驱动的智能体架构师，可以根据自然语言描述创建智能体规范
2. **路由**：`TaskTool` 生成子智能体，创建具有各自权限范围的子会话
3. **执行**：每次智能体调用都通过 `SessionPrompt` 运行——这是主管道，用于构建上下文、通过 `LLM.stream()` 调用 LLM、通过 `SessionProcessor` 处理工具调用，并循环直到完成或触发停止条件
4. **终止**：自然完成、达到步骤限制或触发压缩

### 基于 Effect 的架构

OpenCode 最独特的架构选择是使用 **Effect-TS** 库作为基础运行时层。所有服务都定义为 Effect 层：

```typescript
// Effect 服务模式（来自 bus/index.ts）
export class Service extends Context.Service<Service, Interface>()("@opencode/Bus") {}

export const layer = Layer.effect(Service, Effect.gen(function* () {
  const state = yield* InstanceState.make<State>(...)
  // ... 服务实现
}))
```

这提供了：
- **依赖注入**：服务将依赖声明为 `Layer` 类型参数
- **错误处理**：通过 `Schema.TaggedError` 实现类型化错误
- **资源管理**：通过 `Effect.acquireRelease` 实现作用域资源
- **并发**：通过 `Effect.fork`/`Fiber` 实现结构化纤程管理
- **可观测性**：通过 `@effect/opentelemetry` 集成 OpenTelemetry

### 实例状态模型

`InstanceState` 抽象通过 `ScopedCache` 提供按目录/按项目的服务缓存：

```typescript
export const make = <A, E, R>(
  init: (ctx: InstanceContext) => Effect.Effect<A, E, R | Scope.Scope>,
): Effect.Effect<InstanceState<A, E, Exclude<R, Scope.Scope>>> =>
  Effect.gen(function* () {
    const cache = yield* ScopedCache.make<string, A, E, R>({
      capacity: Number.POSITIVE_INFINITY,
      lookup: () => init(yield* context),
    })
    // ... 注册清理器，返回状态
  })
```

这确保了 Bus、LSP 和权限规则等服务的作用域限定在一个项目目录中，并在实例被销毁时进行清理。

### 运行器系统

`Runner` 抽象建模了智能体的执行状态机：

```
Idle ──► Running ──► Shell ──► ShellThenRun ──► Running
  ▲                                                    │
  └────────────────────────────────────────────────────┘
```

四种状态：`Idle`、`Running`、`Shell`、`ShellThenRun`——使智能体能够运行 LLM 回合、执行 Shell 命令、排队 shell-then-run 序列，并干净地处理取消。

### V2 架构

`v2/` 目录包含一个新兴的事件溯源架构，将会话、消息和提示词重新定义为版本化的事件流。`EventV2.define()` 函数创建具有同步支持的类型化事件：

```typescript
export function define<const Type extends string, Fields extends Schema.Struct.Fields>(input: {
  type: Type
  schema: Fields
  aggregate: string
  version?: number
}) { /* ... 返回带有 Sync 集成的 Payload */ }
```

这表明正在向 **CQRS/事件溯源** 迁移，以实现更好的审计追踪、离线同步和多实例协调。

## (d) 工具系统设计

### 工具注册表架构

工具系统是 LLM 与外部世界交互的主要接口。它包括：

1. **`Tool.Info`**：一个延迟初始化的工具定义，包含 `id` 和 `init()`，返回参数、描述和执行函数
2. **`Tool.Def`**：解析后的工具，具有强类型参数和元数据
3. **`Tool.Context`**：执行上下文，提供会话状态、中止信号、消息历史、元数据跟踪和权限询问

```typescript
// 工具定义模式
export interface Def<P extends Schema.Decoder, M extends Metadata> {
  id: string
  description: string
  parameters: P
  execute(args: Schema.Schema.Type<P>, ctx: Context): Effect.Effect<ExecuteResult<M>>
  formatValidationError?(error: unknown): string
}
```

### 核心工具（39 个文件）

OpenCode 定义了 **17 个以上的内置工具**，每个都作为延迟的 `Tool.Info`，解析为 `Tool.Def`：

| 工具 | 类型 | 描述 |
|------|------|-------------|
| `bash` / `shell` | 执行 | 运行 Shell 命令，具有进程管理、超时和扫描功能 |
| `read` | 输入输出 | 读取文件/目录/图像/PDF，支持偏移量和限制 |
| `write` | 变更 | 创建/覆写文件，具有差异预览 |
| `edit` | 变更 | 精确字符串替换，使用信号量锁 |
| `apply_patch` | 变更 | 多块（multi-hunk）补丁应用，具有文件级锁定 |
| `grep` | 搜索 | 通过 ripgrep 进行正则内容搜索 |
| `glob` | 搜索 | 快速文件模式匹配，按修改时间排序 |
| `lsp` | 智能 | LSP 操作：跳转到定义、查找引用、文档/工作区符号、悬停、调用层次结构 |
| `task` | 编排 | 将子智能体生成为子会话（任务委托） |
| `todowrite` | 状态 | 结构化待办事项列表管理 |
| `question` | 交互 | 向用户提问并收集答案 |
| `webfetch` | 外部 | HTTP 抓取，将 HTML 转换为 Markdown |
| `websearch` | 外部 | 通过 Exa API 进行网络搜索 |
| `skill` | 扩展 | 加载和调用技能定义 |
| `plan` | 控制 | 计划进入 / 计划退出，用于结构化规划阶段 |
| `truncate` | 工具 | 输出截断，基于文件的溢出存储 |
| `invalid` | 哨兵 | 无效/未知工具调用的占位符 |

### 工具执行流程

```
1. LLM 发起工具调用 → 2. SessionProcessor 解析 Tool.Def
  → 3. Permission.evaluate(rules, tool, patterns)
    → 4. 如果为 "ask"：Deferred.await → 用户提示
    → 5. Tool.execute(params, ctx) 在 Effect 运行时中执行
    → 6. 结果：字符串输出 + 元数据 + 附件
    → 7. Session processor 将 ToolPart 追加到消息中
    → 8. Bus 发出工具完成事件
    → 9. LLM 在下一个上下文中接收工具结果
```

### Bash 工具：核心接口

`ShellTool` 是最复杂的工具实现。它封装了系统 Shell 执行，具有：

- **进程生成**：通过 `effect/unstable/process` 使用 Bun 的 `node-pty` 实现 PTY 支持
- **超时处理**：默认 2 分钟超时，可通过 `OPENCODE_EXPERIMENTAL_BASH_DEFAULT_TIMEOUT_MS` 配置
- **权限扫描**：解析 Shell 命令中影响文件系统的操作（`rm`、`cp`、`mv`、`mkdir`、`chmod` 等），并将其映射到权限模式
- **输出截断**：截断到 2000 行 / 50KB，将完整输出存储在截断目录中，保留 7 天
- **信号处理**：SIGTERM → SIGKILL 升级机制，Windows 上使用 tree-kill
- **Tree-sitter 解析**：基于 AST 的命令分析用于安全扫描

该工具定义了分类命令集：
- `FILES`：文件修改命令（`rm`、`cp`、`mv`、`mkdir`、`touch`、`chmod`、`chown`、`cat`）
- `CWD`：目录更改命令（`cd`、`chdir`、`pushd`、`popd`）
- `CMD_FILES`：Windows CMD 等效命令（`copy`、`del`、`move`、`md`、`rename`、`rd`）

这种主动命令解析使权限系统能够在执行前拦截危险操作。

## (e) 状态管理

### 会话状态架构

会话是状态的主要单元。每个会话都是数据库持久化的记录，具有：
- **SQLite 存储**：通过 Drizzle ORM（`SessionTable`、`MessageTable`、`PartTable`）
- **父子关系**：子智能体会话是父会话的子级
- **消息版本控制**：V2 消息格式，具有类型化的部分（文本、工具调用、工具结果、推理、文件）
- **Token 跟踪**：输入、输出、缓存读取、缓存写入、推理 Token
- **摘要**：每个会话的文件更改 Git-diff 风格摘要
- **回滚能力**：会话级撤销

```typescript
// 具有类型化部分的会话消息（来自 message-v2.ts）
// 部分包括：TextPart, ToolPart, ReasoningPart, FilePart,
//   ImagePart, ErrorPart 等
export interface WithParts {
  id: MessageID
  sessionID: SessionID
  role: "user" | "assistant"
  parts: Part[]
}
```

### 每回合的状态流转

```
用户消息
  │
  ▼
SessionPrompt.prompt()
  │
  ├── 1. 压缩检查 (overflow.ts)
  │     ├── usable = model.limit.context - reserved
  │     └── 如果 tokens >= usable → 压缩
  │
  ├── 2. 上下文构建
  │     ├── 系统提示词（模型特定 + 技能 + 环境）
  │     ├── 消息历史（修剪 + 压缩）
  │     ├── 工具定义（JSON Schema）
  │     └── 智能体权限
  │
  ├── 3. LLM.stream()
  │     ├── 提供者特定的模型封装
  │     ├── SSE 超时封装
  │     └── Token 计数
  │
  ├── 4. SessionProcessor.process()
  │     ├── 解析流式响应
  │     ├── 提取工具调用
  │     ├── 通过 ToolRegistry 执行工具
  │     ├── 处理错误（重试、末日循环检测）
  │     └── 返回 "continue" | "compact" | "stop"
  │
  └── 5. 后处理
        ├── 会话摘要更新（diff 统计）
        ├── 待办事项列表持久化
        └── Bus 事件（session.updated, session.compacted）
```

### 会话压缩

压缩是管理 LLM 上下文窗口的关键机制。`compaction.ts` 模块实现了：

```typescript
// 溢出检测 (overflow.ts)
export function isOverflow(input) {
  if (input.cfg.compaction?.auto === false) return false
  const count = input.tokens.total || input.tokens.input + input.tokens.output + ...
  return count >= usable(input)
}

// 压缩策略 (compaction.ts)
const PRUNE_MINIMUM = 20_000
const PRUNE_PROTECT = 40_000
const DEFAULT_TAIL_TURNS = 2
```

压缩保留：
1. **系统提示词**（始终保留）
2. **最近的尾部回合**（默认保留最后 2 个回合，可配置）
3. **受保护的工具**（`skill` 工具调用）
4. **至少 2000 个 Token** 的最近上下文
5. **较早回合的压缩摘要**，包含以下部分：目标、约束、进展、关键决策、下一步、关键上下文、相关文件

### Effect 系统状态

`effect/` 模块提供运行时基础设施：

- **`Bridge`**：支持从 JavaScript 回调调用 Effect，同时通过 AsyncLocalStorage 保留实例上下文
- **`RunService`**：创建 `ManagedRuntime` 实例，具有工作区和实例上下文的按实例引用
- **`InstanceRegistry`**：跟踪运行中的实例及其清理回调
- **`AppRuntime`**：引导应用级 Effect 运行时

桥接尤其有趣——它解决了在基于 Promise 的异步代码和 Effect 纤程之间跨越的问题，同时保留了 Effect 的纤程本地存储无法跨 async/await 边界传播的 `InstanceRef` 和 `WorkspaceRef` 上下文：

```typescript
// bridge.ts — 跨越 Effect ↔ Promise
export const fromPromise = <T>(fn: () => Promise<T> | T): Effect.Effect<T> =>
  Effect.gen(function* () {
    const instance = yield* InstanceRef
    const workspace = yield* WorkspaceRef
    return yield* Effect.promise(() =>
      Promise.resolve(restore(instance, workspace, () => fn())))
  })
```

## (f) LLM 集成

### 多提供者系统

OpenCode 通过 Vercel AI SDK（`ai` v6.0.168）集成了广泛的 LLM 提供者：

```typescript
// 来自 package.json 的依赖项
"@ai-sdk/openai": "3.0.53",
"@ai-sdk/anthropic": "3.0.71",
"@ai-sdk/google": "3.0.63",
"@ai-sdk/amazon-bedrock": "4.0.96",
"@ai-sdk/azure": "3.0.49",
"@ai-sdk/mistral": "3.0.27",
"@ai-sdk/groq": "3.0.31",
"@ai-sdk/xai": "3.0.82",
"@ai-sdk/deepinfra": "2.0.41",
"@ai-sdk/cerebras": "2.0.41",
"@ai-sdk/cohere": "3.0.27",
"@ai-sdk/perplexity": "3.0.26",
"@ai-sdk/togetherai": "2.0.41",
"@ai-sdk/alibaba": "1.0.17",
// + gateway, openai-compatible, vercel
"@openrouter/ai-sdk-provider": "2.8.1",
"gitlab-ai-provider": "6.6.0",
```

`Provider` 服务管理：
- **模型发现**：`models.ts` 中的静态定义 + 从 npm 包动态加载
- **API 密钥解析**：通过 `Auth` 服务，支持环境变量、配置文件和云密钥存储
- **模型封装**：提供者特定的模型转换（例如，Github Copilot 对 GPT-5 的 Responses API）
- **SSE 超时封装**：检测停滞的服务器发送事件（SSE）流

### 流式架构

`LLM.Service` 封装了 `ai.streamText()`：

```typescript
// llm.ts — 流式接口
export type StreamInput = {
  user: MessageV2.User
  sessionID: string
  model: Provider.Model
  agent: Agent.Info
  permission?: Permission.Ruleset
  system: string[]
  messages: ModelMessage[]
  tools: Record<string, Tool>
  toolChoice?: "auto" | "required" | "none"
}
```

流返回 `Stream.Stream<Event>`——一个 Effect 原生的 LLM 事件流（文本增量、工具调用、完成原因等），通过 `SessionProcessor` 处理。

### 上下文构建

`SessionPrompt` 分层构建 LLM 上下文：

1. **模型特定的系统提示词**：为 Claude（`anthropic.txt`）、GPT（`gpt.txt`、`beast.txt`）、Gemini（`gemini.txt`）、Codex（`codex.txt`）、Kimi（`kimi.txt`）、Trinity（`trinity.txt`）量身定制的提示词
2. **环境上下文**：工作目录、工作区根目录、Git 状态、平台、日期
3. **技能**：从文件系统加载的可用技能
4. **智能体特定的提示词**：来自智能体定义的自定义系统提示词
5. **计划提醒**：在计划模式下，提供架构上下文
6. **构建开关**：条件性的构建/测试指令
7. **消息历史**：修剪和压缩后的助手/用户回合
8. **工具定义**：可用工具的 JSON Schema 表示

## (g) 上下文管理

OpenCode 的上下文管理是该框架在架构上最复杂的方面。它必须将任意大的编码会话放入固定大小的 LLM 上下文窗口（通常为 200K Token）。

### 三层上下文策略

1. **修剪**：工具输出被截断到 2000 个字符（`TOOL_OUTPUT_MAX_CHARS`），完整输出可选择性地存储在基于文件的溢出中
2. **压缩**：当总 Token 超过可用空间时（上下文限制减去为输出预留的空间），较早的回合通过 LLM 驱动的压缩过程进行摘要
3. **截断**：单个工具输出有硬性限制（2000 行，50KB），并使用基于文件的溢出存储

### 压缩算法

```typescript
// 来自 compaction.ts
const PRUNE_MINIMUM = 20_000    // 保留的最小 Token 数
const PRUNE_PROTECT = 40_000    // 受保护的最近 Token 数
const DEFAULT_TAIL_TURNS = 2    // 尾部保留的完整回合数
const MIN_PRESERVE_RECENT_TOKENS = 2_000
const MAX_PRESERVE_RECENT_TOKENS = 8_000
```

压缩摘要使用结构化模板，包含以下部分：目标、约束与偏好、进度（已完成/进行中/被阻塞）、关键决策、下一步、关键上下文以及相关文件。这种结构化格式确保压缩后的上下文对 LLM 仍然具有可操作性。

### 溢出检测

`overflow.ts` 模块将可用上下文计算为：

```
usable = model.limit.input 
  ? max(0, model.limit.input - reserved) 
  : max(0, context - maxOutputTokens)
```

其中 `reserved` 为 `min(COMPACTION_BUFFER, maxOutputTokens)`——通常为 20,000 个 Token。

## (h) LSP 与文件系统集成

### LSP 架构

LSP 模块同时作为以下两者提供语言服务器协议集成：
- 用于后台索引和诊断的**服务**
- 用于 LLM 驱动符号导航的**工具**（`lsp.ts`）

LSP 工具支持九种操作：
```
goToDefinition, findReferences, hover, documentSymbol,
workspaceSymbol, goToImplementation, prepareCallHierarchy,
incomingCalls, outgoingCalls
```

LSP 服务器通过 `launch.ts` 模块启动，在 `config/lsp.ts` 中按语言配置，并作为具有重连逻辑的持久进程进行管理。

### 文件系统抽象

`AppFileSystem`（来自 `@opencode-ai/core/filesystem`）提供了统一的文件系统接口，抽象了以下内容：
- 本地文件系统（Bun 的 `Bun.file()` 和 Node.js `fs`）
- 内存文件系统（用于沙盒化测试执行）
- 远程文件系统（用于容器/远程开发）

### Ripgrep 集成

文件搜索（grep、glob）由 `ripgrep` 二进制文件（通过 `@opencode-ai/core`）提供支持，能够在大型代码库上提供亚毫秒级的正则搜索。`Ripgrep.Service` 封装了 ripgrep 调用，具有：
- 基于 Glob 的文件过滤
- 上下文行提取
- 通过 `Stream.Stream` 流式传输结果
- 结果数量限制和截断

## (i) 权限与安全模型

### 基于规则的权限

权限系统使用基于规则的模式匹配引擎：

```typescript
// permission/evaluate.ts
export function evaluate(permission: string, pattern: string,
  ...rulesets: Rule[][]): Rule {
  const rules = rulesets.flat()
  const match = rules.findLast(
    (rule) => Wildcard.match(permission, rule.permission)
      && Wildcard.match(pattern, rule.pattern),
  )
  return match ?? { action: "ask", permission, pattern: "*" }
}
```

关键特性：
- **最后匹配胜出**：规则按顺序评估，后面的规则覆盖前面的规则
- **通配符匹配**：权限和模式都支持 glob 风格的通配符（`*`、`**`）
- **三种操作**：`allow`（自动批准）、`deny`（阻止）、`ask`（提示用户）
- **多层规则集**：智能体级权限、工作区配置权限和会话特定的覆盖
- **`always` 模式**：某些模式可以标记为"始终允许"（例如 `["*"]`）

### 工具执行的权限流程

```
Tool.execute() 调用 ctx.ask({ permission: "edit", patterns: [filepath], ... })
  │
  ├── evaluate(tool, patterns, agent.rules, config.rules, session.rules)
  │     └── 返回 { action: "allow" | "deny" | "ask" }
  │
  ├── 如果为 "allow"：立即解析
  ├── 如果为 "deny"：以错误形式拒绝
  └── 如果为 "ask"：创建 Deferred，发布 Bus 事件，
       等待用户回复（一次/始终/拒绝）
```

### Shell 命令扫描

Shell 工具使用 `web-tree-sitter` 执行 AST 级别的命令解析，在执行前识别影响文件系统的操作。这种预执行扫描将 Shell 命令映射到权限模式，从而能够精细控制允许哪些 Shell 操作。

## (j) 非马尔可夫分解分析

### 马尔可夫分解论题

**马尔可夫分解论题**主张：*大型代码项目的智能体框架之所以成功，在于将固有的非马尔可夫系统（其中完整状态太大，无法放入任何 LLM 上下文窗口）分解为一系列马尔可夫决策问题，其中每个 LLM 回合只看到一个精心构建的、自包含的上下文，该上下文足以满足当前决策的需要。*

换句话说：完整的项目状态是非马尔可夫的（整个代码库无法放入上下文中），但每次单独的 LLM 调用应接收到一个马尔可夫状态——一个包含做出下一决策所需的一切信息、而不需要记住之前发生过什么的上下文。

### OpenCode 如何体现马尔可夫分解

OpenCode 通过**多层次分解策略**体现了这一论题：

#### 第一层：Bash 接口作为主要的状态访问器

`ShellTool`（`bash`）是将非马尔可夫的项目状态转换为马尔可夫 LLM 上下文的核心机制。当 LLM 需要关于代码库的信息时，它发出 bash 命令（`grep`、`find`、`cat`、`git log`、`npm test` 等），这些命令会：

1. **探测状态空间**：查询代码库中的特定信息
2. **返回确定性结果**：每个命令产生有界的、确定性的输出
3. **创建马尔可夫快照**：命令输出 + 系统状态 = 下一步决策所需的充分上下文

```typescript
// grep 工具体现了这种模式：
export const GrepTool = Tool.define("grep", Effect.gen(function* () {
  return {
    execute: (params, ctx) => Effect.gen(function* () {
      const result = yield* rg.search({
        cwd, pattern: params.pattern,
        glob: params.include ? [params.include] : undefined,
        signal: ctx.abort,
      })
      // 结果是代码库的一个有界的、确定性的快照
      return { title, output: formattedResult, metadata }
    }),
  }
}))
```

这**不仅仅是便利性**——它是状态分解的架构机制。每个 bash 命令都是一个查询，从完整的项目状态中提取出相关的子集，将无法管理的大状态空间转换为可管理的马尔可夫观测。

#### 第二层：工具系统作为状态机

工具系统将 LLM 与世界的交互转换为**有限状态机视图**：

- 每个工具都是一个**确定性转换**：给定输入参数，产生有界输出
- 工具不能超出其定义范围产生副作用（由权限扫描强制执行）
- 工具结果是添加到消息历史中的**不可变事实**——它们成为后续回合的马尔可夫状态的一部分
- `todowrite` 工具提供**显式状态跟踪**——待办事项列表成为上下文的一部分，使任务进度具有马尔可夫性

这意味着 LLM 永远不需要"记住"项目状态——它始终可以通过工具查询。上下文窗口包含：
- 系统提示词（常量）
- 待办事项列表（显式状态）
- 工具调用历史（状态访问的对话记录）
- 最近的输出（最相关的状态快照）

#### 第三层：压缩作为马尔可夫状态压缩

压缩是马尔可夫分解最直接的体现。当上下文窗口溢出时：

1. **旧回合被移除**（非马尔可夫的过去被丢弃）
2. **结构化摘要取代它们**——将历史状态有损压缩为马尔可夫表示
3. 摘要精确保留了未来决策所需的内容：目标、约束、进度、决策、下一步
4. LLM 继续在此压缩状态下运行，就如同它拥有完整的历史一样

这就是**实际运行中的马尔可夫性质**：未来仅依赖于当前（压缩后的）状态，而不依赖于完整的历史。

```typescript
// 压缩精确保留所需的内容：
const SUMMARY_TEMPLATE = `
## Goal           // ← 我们的目标是什么？
## Constraints     // ← 哪些规则约束我们的行动？
## Progress        // ← 我们目前在哪里？
## Key Decisions   // ← 我们做了什么决策？为什么？
## Next Steps      // ← 下一步做什么？
## Critical Context // ← 哪些技术事实是重要的？
## Relevant Files  // ← 涉及哪些文件？
`
```

#### 第四层：实例状态作为自包含的世界

`InstanceState` 模式为每个项目目录创建了一个**作用域限定、自包含的世界**：

```typescript
// 每个实例拥有自己的：
// - Bus（发布-订阅）
// - LSP 服务器
// - 权限规则
// - Shell 环境
// - 文件系统状态
export const make = <A, E, R>(init) =>
  Effect.gen(function* () {
    const cache = yield* ScopedCache.make(...)
    // 将所有服务绑定到一个目录
  })
```

这种作用域限定意味着跨实例状态不会泄露——每个目录/项目都是一个独立的马尔可夫域。

### OpenCode 偏离纯马尔可夫分解之处

1. **Skill 工具**引入了非确定性的外部状态（加载的文件、自定义指令），这些状态可以跨回合持续存在，而 LLM 无法仅通过工具来访问。

2. **`question` 工具**引入了人机交互（Human-in-the-loop）状态，该状态在工具-结果对话之外——人类的回答通过工具管道之外的机制成为上下文的一部分。

3. **插件扩展**可以引入任意状态，这些状态可能无法在工具-结果模型中完全捕获。

4. **会话回滚**允许跳回到先前状态，这本质上是非马尔可夫的——它需要记住一个已经被丢弃的状态。

### Bash 接口的核心角色

类似 Bash 的接口对 OpenCode 的架构**绝对至关重要**。证据：

1. `ShellTool` 是最复杂的工具实现之一（215 行以上的进程管理、tree-sitter 扫描、输出处理）
2. 即使是像 `grep` 和 `glob` 这样的高级工具，最终也通过类似 Shell 的执行模型调用 ripgrep
3. 文件系统工具（`read`、`write`、`edit`）在功能上等同于 bash 操作（`cat`、`echo >`、`sed -i`）
4. 权限系统的安全扫描是围绕解析 Shell 命令构建的
5. 截断系统将 Shell 输出存储在基于文件的目录结构中

**为什么是 bash？** 因为 bash 具有以下特性：
- **通用性**：每个开发者都了解它；每个系统都有它
- **可组合性**：`grep | sort | head` 链式操作天然具有表达力
- **确定性**：相同的输入 → 相同的输出（模文件系统状态）
- **自文档化**：命令字符串就是查询；输出就是结果
- **有界性**：输出可以被截断；错误是显式的

**已考虑的替代方案**：该框架不提供 REST API 工具、数据库工具或其他结构化的外部接口——bash 工具是通用的后备方案。`webfetch` 和 `websearch` 工具用于 Web 访问，但这些是补充性的，而非替代方案。

### 马尔可夫分解论题对 OpenCode 是否成立？

**是的，但有保留。** OpenCode 的架构强有力地支持这一论题：

1. **上下文始终有界**：压缩、截断和修剪强制执行硬性限制
2. **状态访问是显式的**：LLM 获取的每条信息都通过工具调用获得，结果是有界的、确定性的
3. **压缩创建马尔可夫摘要**：结构化摘要格式旨在足以满足未来决策的需要
4. **待办事项列表使任务进度具有马尔可夫性**：显式状态跟踪器消除了从消息历史推断进度

**然而**，这一论题并不完美：

- LLM 仍然在上下文中看到**工具调用历史**——这是一种情景记忆（episodic memory），而非纯粹的马尔可夫状态
- **系统提示词**是对每次决策产生持续性影响的非马尔可夫因素
- **压缩摘要**是一种有损表示——信息丢失可能导致次优决策，而这些决策在拥有完整历史时不会发生
- **权限系统的"始终"规则**创建了在 LLM 上下文中不可见的持久状态

### 马尔可夫分解架构模式

OpenCode 展示的通用模式：

```
非马尔可夫世界（完整代码库，数百万 Token）
         │
         ▼ 工具系统（bash, grep, read, LSP）
         │ ── 查询接口：提取相关切片
         │ ── 有界输出：确定性、截断
         │ ── 状态变更：write, edit, apply_patch
         │
         ▼ 马尔可夫上下文（每回合，有界 Token）
         ├── 系统提示词（每会话不可变）
         ├── 待办事项列表（显式任务状态）
         ├── 工具调用 → 结果对（情景上下文）
         ├── 压缩摘要（压缩后的历史）
         └── 环境（工作区、git、平台）
         │
         ▼ LLM 决策（无状态，上下文 → 行动）
         │
         └──► 工具调用 → 返回工具系统
```

这种模式使 OpenCode 能够有效处理任意大的代码库：它从不试图将整个项目放入上下文中。相反，它为 LLM 提供了一个**马尔可夫接口**——一组可以查询和变更世界的工具，结合上下文管理系统，确保每个回合拥有恰好做出下一决策所需的信息。

## (k) 设计模式与理念

### 关键设计模式

1. **Effect 服务模式**：所有能力都暴露为 Effect `Context.Service<T>`，具有基于 `Layer` 的依赖注入
2. **延迟工具初始化**：`Tool.Info` 封装了一个延迟的 `() => Effect.Effect<Def>`，在执行时解析依赖项
3. **事件驱动解耦**：`Bus`（PubSub）解耦了会话变更、权限事件和 UI 更新
4. **实例作用域**：`InstanceState` 通过 `ScopedCache` 提供按目录的服务缓存
5. **运行器状态机**：智能体执行循环被建模为有限状态机
6. **运行时适配器模式**：平台特定代码通过 Bun 的 `#imports` 映射有条件地导入（`#db.bun.ts` vs `#db.node.ts`）
7. **事件溯源（V2）**：新兴的 V2 架构使用具有同步支持的事件驱动状态

### 编码理念（来自 AGENTS.md 和代码风格）

- **函数式优于命令式**：优先使用 `const`、提前返回、三元运算符、函数式数组方法
- **不使用 try/catch**：使用 Effect 的类型化错误处理
- **不使用 `any` 类型**：始终追求完整的类型安全
- **单一用途函数**：避免过早抽象；在可能的情况下内联
- **避免在测试中使用 mock**：测试实际实现
- **并行工具**：在可能的情况下设计支持并发执行

## (l) 优势与劣势

### 优势

1. **Effect-TS 基础**：提供类型安全的错误处理、资源管理、并发和可观测性——在 TypeScript 智能体框架中实属罕见
2. **多提供者深度**：15 个以上的 LLM 提供者，具有模型特定的系统提示词
3. **压缩的成熟度**：结构化的、LLM 驱动的摘要保留了可操作的上下文
4. **权限粒度**：基于规则的、通配符匹配的、多层次的权限评估
5. **以 Bash 为中心的设计**：任何 LLM 都能有效使用的通用接口
6. **事件溯源迁移路径**：V2 架构支持审计追踪和多实例同步
7. **开源 + 插件 SDK**：开箱即用的第三方扩展性
8. **全栈单仓库**：终端 UI、Web 应用、桌面应用，共享同一个核心引擎

### 劣势

1. **Effect-TS 复杂性**：Effect 库学习曲线陡峭；贡献者必须理解单子效应系统
2. **Bun 依赖**：该项目与 Bun 紧密耦合，既是运行时也是包管理器，限制了可移植性
3. **Beta 版依赖**：Effect 和许多 AI SDK 包都在 Beta 版本上
4. **压缩信息丢失**：摘要不可避免地会丢失细微差别；重复压缩可能降低决策质量
5. **不支持原生视觉**：虽然提示词支持图像附件，但没有截图/画布工具用于视觉推理
6. **仅支持基于 Bash 的状态变更**：所有代码更改都通过基于文本的工具（write、edit、apply_patch）进行——没有基于 AST 的重构工具
7. **有限的结构化输出**：工具模式是纯 JSON Schema；除了 `plan` 工具之外，没有结构化的、受模式约束的输出机制
8. **实例隔离的限制**：虽然按目录的作用域限定防止了状态泄露，但也阻止了跨项目的上下文共享

## (m) 总结表

| 维度 | 评估 |
|-----------|------------|
| **架构** | 分层六边形架构、Effect-TS 服务、事件驱动总线 |
| **智能体模型** | 声明式配置 + 动态生成；主要/子智能体层次结构 |
| **工具系统** | 17 个以上工具；延迟初始化；强类型参数 |
| **LLM 集成** | 通过 Vercel AI SDK 支持 15 个以上提供者；模型特定的提示词 |
| **状态管理** | SQLite/Drizzle；压缩 + 截断；V2 事件溯源 |
| **上下文策略** | 修剪 → 压缩 → 截断；三层管理 |
| **权限模型** | 基于规则的通配符匹配；多层次；Shell 扫描 |
| **马尔可夫论题契合度** | 强：bash 接口 + 压缩创建马尔可夫回合 |
| **语言** | TypeScript（严格模式）；Bun 运行时 + Node.js 回退 |
| **依赖项** | Effect-TS、Vercel AI SDK、Drizzle ORM、ripgrep、tree-sitter |
| **规模** | 约 18 个包；核心 43 个模块；200K 以上下载量 |
| **许可证** | MIT |
| **版本** | 1.14.41（截至分析时） |
