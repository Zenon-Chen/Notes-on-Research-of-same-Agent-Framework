# 两两对比：free-code vs OpenCode

## 两款终端原生 AI 编码智能体如何应对非马尔可夫状态分解

**分析日期：** 2026年5月  
**对比版本：** free-code 2.1.87 / OpenCode 1.14.41  
**分析依据：** 各项目独立架构报告（01-free-code-analysis.md、03-opencode-analysis.md）

---

## 目录

1. [执行摘要](#a-执行摘要)
2. [架构对比](#b-架构对比)
3. [智能体编排模型](#c-智能体编排模型)
4. [工具系统对比](#d-工具系统对比)
5. [状态管理](#e-状态管理)
6. [LLM 集成](#f-llm-集成)
7. [上下文管理](#g-上下文管理)
8. [通信模型](#h-通信模型)
9. [马尔可夫分解分析](#i-马尔可夫分解分析)
10. [评分表](#j-评分表)
11. [使用场景建议](#k-使用场景建议)

---

## (a) 执行摘要

free-code 和 OpenCode 代表了构建终端原生 AI 编码智能体的两种根本不同的哲学路线。两者都是运行在 Bun 上的生产级 TypeScript 系统，都向 LLM 提供约 50 个工具类别，并且都汇聚于同一个架构洞见——LLM 不需要将整个项目载入上下文，只需要一个马尔可夫式的工具接口，可以按需查询和变更状态。然而，它们在代码组织方式、状态管理、模型集成以及组件间通信处理方面存在显著分歧。

**free-code** 是 Anthropic Claude Code 的剥离遥测分支——一个由构建其所编排模型的公司打造的生产级单体应用。约 1,934 个源文件，体现了一种务实的、单仓库哲学：`QueryEngine` 异步生成器是其中枢神经系统，Redux 风格的 store 管理应用状态，Anthropic SDK 是第一优先级的集成路径。它追求深度而非广度：五个 LLM 提供商，各自带有细致的 API 特定处理；三级压缩系统，使用 Haiku 驱动的摘要；以及一个与无头 SDK 共享流式路径的 Ink/React 终端 UI。

**OpenCode** 是 AnomalyCo 构建的开源 monorepo，包含 18 个包，基于函数式 effect 库 Effect-TS。free-code 的单体强调内聚性，而 OpenCode 的 monorepo 强调模块化——核心引擎（`packages/opencode`）只是众多包之一，与 Web 应用、桌面封装器、插件 SDK、Slack 集成和企业功能并列。其架构采用分层六边形模式：服务作为 Effect 层注入，Bus 提供发布-订阅解耦，V2 事件溯源架构已在开发中。通过 Vercel AI SDK 支持 15+ 个 LLM 提供商，它追求广度而非深度。

**关键相似点**：两款智能体都使用相同的三层分解策略来处理"理解和修改大型代码库"这一非马尔可夫问题——Bash 作为通用接口 → 工具作为按需状态查询 → 压缩作为历史压缩。两者得出相同的结论：上下文窗口就是马尔可夫状态，智能体的任务就是保持其有界。

**关键差异**：free-code 是一个产品；OpenCode 是一个平台。free-code 的架构优先考虑单人终端体验，具有深度集成的权限门控和针对 Anthropic 优化的工具语义。OpenCode 的架构优先考虑可扩展性——插件 SDK、V2 事件溯源、多终端部署以及 Effect 的依赖注入使其更容易嵌入、扩展和定制，但代价是贡献者面临更陡峭的学习曲线。

本报告剩余部分将从八个架构维度对这些相似点和差异进行分解，根据七个质量维度对两个项目进行评估，并提供具体的使用场景建议。

---

## (b) 架构对比

### free-code：务实的单体应用

free-code 是一个**单一的单体应用**，包含约 1,934 个 TypeScript/TSX 文件，组织成扁平但有纪律的目录层级：

```
src/
├── entrypoints/          # CLI (cli.tsx)、SDK、MCP 服务器
├── QueryEngine.ts        # 中央编排类（异步生成器）
├── tools/                # ~50 个工具实现（BashTool、ReadTool 等）
├── state/                # Redux 风格的 AppStateStore + store.ts
├── services/api/         # 3,419 行的 API 客户端（claude.ts）
├── services/compact/     # 压缩子系统（micro、full、snip）
├── services/mcp/         # Model Context Protocol 集成
├── utils/                # ~120 个共享工具函数
├── hooks/                # 用于 Ink UI 的 React hooks
├── components/           # 终端 UI 组件（Ink/React）
├── skills/               # 技能定义和加载
├── plugins/              # 社区插件系统
└── tasks/                # 后台任务管理
```

`ToolUseContext` 类型聚合了约 60 个字段——这是单体应用集成深度的体现。每个工具、hook 和 UI 组件都可以直接访问完整的应用上下文。这形成了紧耦合，但也实现了快速的功能开发：添加一个新工具只需要继承 `Tool` 类，在 `getAllBaseTools()` 中注册，并编写 Ink 渲染器——这些都在相邻文件中。

**单体应用优势**：快速内部迭代，没有跨包版本管理开销，每个组件都可以直接访问其他组件。**单体应用劣势**：约 60 字段的 `ToolUseContext` 使得对单个工具进行测试变得困难，导入图深度互联，当前快照中有 88 个功能标志中的 34 个编译失败——这表明单体应用的耦合使功能标志变得脆弱。

### OpenCode：模块化的 Monorepo

OpenCode 是一个由 Turborepo 管理的**18 包 monorepo**，具有清晰的关注点分离：

```
packages/
├── opencode/         # 核心智能体引擎（43 个模块）
├── core/             # 共享工具函数（文件系统、ripgrep、日志）
├── console/          # 终端 TUI
├── app/              # Web 应用前端
├── web/              # Web 前端资源
├── desktop/          # 桌面封装器（Electron/Tauri）
├── ui/               # 共享 UI 组件库
├── storybook/        # UI 开发环境
├── plugin/           # 第三方扩展的插件 SDK
├── sdk/              # JS/TS SDK，用于编程控制
├── containers/       # Docker 部署
├── script/           # 构建脚本
├── slack/            # Slack 集成
├── identity/         # 身份认证
├── enterprise/       # 企业功能
├── function/         # 无服务器函数
├── docs/             # 文档
└── extensions/       # 编辑器扩展（VS Code）
```

在 `packages/opencode/src` 内部，架构遵循**分层六边形模式**：外部能力（LSP、文件系统、shell、ripgrep）通过基于 Effect 的服务进行抽象，Bus 提供组件间的发布-订阅解耦，工具注册中心在 LLM 工具调用和具体实现之间进行中介。`AppFileSystem` 抽象位于工具层之下，使得相同的工具可以在本地文件系统、内存文件系统（用于沙盒测试）或远程文件系统（用于容器开发）上运行。

**Monorepo 优势**：清晰的依赖边界（插件 SDK 依赖 core，而非反向），独立版本控制，从单一代码库进行多终端部署，SDK 包使得可以嵌入到非 OpenCode 应用中。**Monorepo 劣势**：跨包依赖管理增加了构建复杂性（Turborepo 开销），Effect-TS 基础带来了陡峭的学习曲线，贡献者必须穿越 18 个包边界才能追踪一个功能的端到端实现。

### 对比评估

free-code 的单体应用更适合紧密集成的终端智能体，其中每个组件都需要感知完整的权限上下文和模型状态。OpenCode 的 monorepo 更适合构建一个必须可嵌入、可扩展、可跨多个终端（终端、Web、桌面、Slack）部署的智能体平台。两种方法没有绝对的优劣——选择取决于目标是追求产品深度（free-code）还是平台广度（OpenCode）。

---

## (c) 智能体编排模型

### free-code：QueryEngine 异步生成器

free-code 的核心是 `QueryEngine` 类——一个独立的、会话级生命周期的编排器，拥有完整对话生命周期。其编排模式是一个**单一的异步生成器函数**（`submitMessage`），产出 `StreamEvent | Message` 对象：

```
QueryEngine.submitMessage(userMessage)
  ├── 查询前钩子（技能加载、记忆注入）
  ├── 上下文构造（系统提示 + 消息历史 + 工具）
  ├── queryModel() → 产出 StreamEvent 的异步生成器
  │     ├── 产出 text deltas → UI 实时渲染
  │     ├── 产出 tool_use blocks → 执行工具 → 产出结果
  │     └── 产出 message deltas → 部分递送推理内容
  ├── 查询后钩子（清理、日志记录）
  ├── 预算检查（最大美元数、最大轮次、最大 token 数）
  └── 会话持久化
```

这种基于生成器的设计具有若干架构意义：

- **流式传输是默认路径**——非流式传输是回退方案，而非独立的代码路径
- **内置中止安全性**——`AbortController` 通过所有 `yield*` 委托传播取消操作
- **REPL 和 SDK 共享同一 API**——两者都消费生成器输出，只是消费者不同（Ink 组件 vs 编程回调）
- **LLM 是唯一的决策者**——QueryEngine 提供基础设施（上下文、工具、流式传输），但从不决定做什么。这完全是 LLM 的领域。

子智能体通过 `AgentTool` 生成，创建拥有自己上下文窗口的子 `QueryEngine` 实例。父智能体委派子任务，接收结构化结果，且永不看到子智能体的内部工具调用历史。

### OpenCode：基于 Effect 的智能体系统及 V2 架构

OpenCode 的编排模型分布在多个互联系统中：

1. **`Agent.Info`**：声明式智能体配置（名称、模式、权限、模型、系统提示）
2. **`Agent.generate()`**：LLM 驱动的智能体架构师，从自然语言创建智能体规范
3. **`SessionPrompt`**：构建上下文、调用 `LLM.stream()` 并处理结果的主管道
4. **`SessionProcessor`**：解析流式响应，提取工具调用，处理错误，检测 doom-loop
5. **`Runner`**：具有四种状态的有限状态机——`Idle`、`Running`、`Shell`、`ShellThenRun`

编排流程如下：

```
用户消息 → SessionPrompt.prompt()
  ├── 压缩检查（overflow.ts）
  ├── 上下文构造（系统提示 + 历史 + 工具 + 权限）
  ├── LLM.stream() → 由 SessionProcessor 处理
  │     ├── 解析流式响应
  │     ├── 提取工具调用 → ToolRegistry.execute()
  │     ├── 处理错误（重试、doom-loop 检测）
  │     └── 返回 "continue" | "compact" | "stop"
  └── 后处理（摘要更新、todo 持久化、Bus 事件）
```

**V2 架构**代表了一次重大演进。`v2/` 目录包含一个新兴的事件溯源架构，将会话、消息和提示重新定义为带版本的事件流。`EventV2.define()` 函数创建带有同步支持的强类型事件——这是向 CQRS/事件溯源迁移的一步，以实现审计追踪、离线同步和多实例协调。

### 关键差异

| 维度 | free-code | OpenCode |
|-----------|-----------|----------|
| **编排模式** | 单一异步生成器 | 分层管道 + 有限状态机 |
| **子智能体模型** | 子 QueryEngine 实例 | 带有父子数据库关系的子会话 |
| **决策架构** | LLM 作为唯一决策引擎 | LLM 作为主引擎，配以系统级护栏（doom-loop 检测） |
| **可扩展性** | 技能 + 插件 + MCP 服务器 | 插件 SDK + Agent.generate() 用于动态智能体创建 |
| **未来轨迹** | 稳定，增量优化 | 积极向事件溯源（V2）迁移 |

free-code 基于生成器的模型以其简洁优雅——整个对话是一个单一的异步迭代器，由 REPL 消费。OpenCode 的分层模型更复杂但也更可扩展——每一层（Prompt、Processor、Runner）都可以独立测试、替换或增强。

---

## (d) 工具系统对比

### 工具类型定义

两个项目都通过共同的接口模式定义工具，但使用了不同的类型系统：

**free-code** 使用基于类的 `Tool` 接口：

```typescript
type Tool = {
  name: string
  description: string
  inputSchema: ToolInputJSONSchema   // Zod schema → JSON Schema
  isEnabled(): boolean               // 功能门控检查
  isReadOnly(): boolean              // 可变更性分类
  call(input, context, options): Promise<ToolResultBlockParam | Message>
  renderResult?(...): React.ReactNode    // Ink/React 渲染
  renderToolUse?(...): React.ReactNode   // 终端 UI
  isDangerous?(input): boolean           // 权限提示
  needsApproval?(...): boolean           // 异步确认门控
}
```

**OpenCode** 使用惰性、Effect 原生的 `Tool.Def` 接口：

```typescript
interface Def<P extends Schema.Decoder, M extends Metadata> {
  id: string
  description: string
  parameters: P                       // Effect Schema decoder
  execute(args, ctx): Effect.Effect<ExecuteResult<M>>
  formatValidationError?(error): string
}
```

### 工具注册与发现

**free-code** 使用**扁平注册表**：`getAllBaseTools()` 返回一个单一的 `Tool` 实例数组，基于功能标志进行条件包含。工具在启动时急切加载。

**OpenCode** 使用**惰性初始化**：`Tool.Info` 包装一个 `() => Effect.Effect<Def>`，在执行时解析依赖。直到 LLM 实际调用时工具才会被解析，从而减少启动开销并支持动态工具注册。

### 工具数量与精细度

| 方面 | free-code | OpenCode |
|--------|-----------|----------|
| **总工具数** | ~50（30 核心 + 20 条件） | 17+ 内置 |
| **文件系统工具** | Read、Write、Edit、Grep、Glob | read、write、edit、apply_patch、grep、glob |
| **Bash 工具功能** | 基于 AST 的命令解析，通过 spawnShellTask 后台执行，沙盒路由，文件历史追踪 | Tree-sitter AST 解析，PTY 支持，权限扫描（分类命令集：FILES、CWD、CMD_FILES），SIGTERM→SIGKILL 升级 |
| **LSP 操作** | go-to-def、find-references、diagnostics | go-to-def、find-references、symbols（文档+工作区）、hover、call hierarchy |
| **独特工具** | Notebook edit、web browser、REPL tool、agent swarms、teams、verify plan execution | apply_patch（多块）、plan（进入/退出）、truncate（溢出存储）、invalid（哨兵） |
| **编排** | AgentTool（生成子智能体）、TaskCreateTool（后台任务） | TaskTool（生成子智能体作为子会话） |

### 执行模式

**free-code** 工具返回 `Promise<ToolResultBlockParam>`，在标准的基于 Promise 的 async 中运行。工具执行与 Anthropic API 的 `tool_use` 块格式集成——结果被结构化为 API 原生理解的内容块。

**OpenCode** 工具返回 `Effect.Effect<ExecuteResult>`，在 Effect 运行时内运行。这提供了：
- **类型化错误通道**——工具可以声明预期的错误类型（例如 `FileNotFoundError`、`PermissionDeniedError`）
- **资源作用域**——工具可以在保证清理的情况下获取/释放资源（文件句柄、子进程）
- **并发**——多个工具调用可以在 Effect 运行时内作为并发纤程分叉
- **可观测性**——OpenTelemetry 追踪自动贯穿工具执行过程

此外，OpenCode 的 `edit` 工具使用**基于信号量的锁定**来防止对同一文件的并发编辑，`apply_patch` 工具使用**文件级锁定**处理多块补丁——这些并发控制机制是 free-code 工具系统中所没有的。

---

## (e) 状态管理

### free-code：类 Redux 的可观察 Store

free-code 的状态管理遵循极简的 Redux 模式。`AppState` 是一个 `DeepImmutable` 类型，包含设置、模型选择、权限上下文、UI 状态、功能标志和远程桥接状态的切片。store 是一个简单的发布-订阅容器：

```typescript
function createStore(initialState: AppState, onChange?: fn) {
  let state = initialState
  let listeners = []
  
  return {
    getState: () => state,
    setState: (updater: (prev: AppState) => AppState) => {
      const old = state
      state = updater(state)
      if (state !== old) {
        listeners.forEach(fn => fn(state, old))
        onChange?.({ newState: state, oldState: old })
      }
    },
    subscribe: (fn) => { listeners.push(fn); return () => /* unsubscribe */ }
  }
}
```

store 通过 React context（`AppStoreContext`）消费，并使用 `useSyncExternalStore` 进行并发模式安全的订阅。每个 `QueryEngine` 实例内的可变消息历史与此不可变 store 并存——对话记录是每个会话临时的，而 AppState 是全局的。

**优势**：简单、可预测、易于调试。每次状态变更都是从先前状态到新状态的纯函数。**劣势**：没有类型安全的错误处理，没有资源管理，没有结构化并发，没有持久化层——store 完全位于内存中。

### OpenCode：Effect 层依赖注入

OpenCode 的状态管理是一个根本不同的范式。没有全局 store——相反，状态通过以下方式管理：

1. **Effect 层**：服务定义为 `Context.Service<T>`，配以基于 `Layer` 的依赖注入。每个服务在 Layer 上声明其依赖作为类型参数，实现编译时依赖解析。

2. **SQLite/Drizzle 持久化**：会话、消息和部分内容通过 Drizzle ORM（`SessionTable`、`MessageTable`、`PartTable`）持久化到 SQLite。这实现了跨进程重启的会话连续性，并通过 revert 能力实现会话级撤销。

3. **InstanceState**：通过 `ScopedCache` 实现的按目录/按项目服务缓存，确保服务（Bus、LSP、权限）作用域限定于项目目录并在销毁时清理。

4. **V2 事件溯源**：新兴的 V2 架构将状态变更存储为版本化事件，支持重放、审计追踪和多实例同步。

5. **V2 消息格式**：消息由类型化的部分内容（`TextPart`、`ToolPart`、`ReasoningPart`、`FilePart`、`ImagePart`、`ErrorPart`）组成，而非扁平 JSON——实现了更丰富的结构化表示。

**优势**：类型安全的错误处理，结构化资源管理（获取/释放），默认持久化，事件溯源就绪。**劣势**：显著更复杂，需要理解单子效应系统，Effect-TS 学习曲线陡峭。

### 关键对比

差异不仅是技术性的，更是哲学性的。free-code 将状态视为**应用的可变快照**——可检查、可修改，但临时的。OpenCode 将状态视为**持久的、事件驱动的流**——持久的、可追踪的，并设计用于多实例协调。对于单人终端智能体，free-code 更简单的模型通常已足够。对于必须支持并发会话、离线同步和审计需求的平台，OpenCode 的模型在架构上是必要的。

---

## (f) LLM 集成

### free-code：深度 Anthropic，多提供商适应能力

free-code 的 LLM 集成集中在一个单一的 3,419 行文件（`src/services/api/claude.ts`）中，通过统一的 `queryModel()` 接口处理五个 API 提供商：

| 提供商 | 选择方法 | 集成方式 |
|----------|-----------------|-------------|
| Anthropic | 默认（`ANTHROPIC_API_KEY`） | `beta.messages.create()`——原生 SDK |
| OpenAI Codex | `CLAUDE_CODE_USE_OPENAI=1` | 自定义适配器，转换 Anthropic `tool_use` 格式 |
| AWS Bedrock | `CLAUDE_CODE_USE_BEDROCK=1` | `BedrockRuntime` + 代理路由 |
| Google Vertex | `CLAUDE_CODE_USE_VERTEX=1` | Vertex AI 端点 |
| Anthropic Foundry | `CLAUDE_CODE_USE_FOUNDRY=1` | 企业部署 |

提供商选择是二元的（每次激活一个提供商，通过环境变量选择），集成是**以 Anthropic 为首要**的：工具定义、内容块和流式事件都符合 Anthropic API 格式。非 Anthropic 提供商需要转换层，可能会丢失语义保真度，从而产生微妙但真实存在的供应商锁定。

流式路径（`queryModelWithStreaming`，一个异步生成器）是默认方式；非流式是在流式传输存在问题的情况下的回退方案（Bedrock 使用大型工具集时、API 错误）。流式事件包括文本增量、工具使用块、消息增量和结构化输出——实时递送到 Ink 终端 UI。

### OpenCode：通过 Vercel AI SDK 实现广泛的多提供商支持

OpenCode 通过 Vercel AI SDK 集成了**15+ 个 LLM 提供商**，由 `Provider` 服务管理：

```
支持的提供商：
├── Anthropic (@ai-sdk/anthropic)
├── OpenAI (@ai-sdk/openai)
├── Google (@ai-sdk/google、@ai-sdk/google-vertex)
├── Azure (@ai-sdk/azure)
├── AWS Bedrock (@ai-sdk/amazon-bedrock)
├── xAI (@ai-sdk/xai)
├── DeepInfra (@ai-sdk/deepinfra)
├── Cerebras (@ai-sdk/cerebras)
├── Cohere (@ai-sdk/cohere)
├── Perplexity (@ai-sdk/perplexity)
├── TogetherAI (@ai-sdk/togetherai)
├── Alibaba (@ai-sdk/alibaba)
├── OpenRouter（通过 @openrouter/ai-sdk-provider）
├── GitHub Copilot（通过 gitlab-ai-provider，为 GPT-5 定制 Responses API 封装器）
└── OpenAI 兼容网关
```

模型发现结合了静态定义（`models.ts`）和从 npm 包的动态加载。API 密钥解析流经 `Auth` 服务，支持环境变量、配置文件和云密钥存储。每个提供商系列维护有模型特定的系统提示：`anthropic.txt`、`gpt.txt`、`beast.txt`、`gemini.txt`、`codex.txt`、`kimi.txt`、`trinity.txt`。

流式接口返回 `Stream.Stream<Event>`——一个 Effect 原生的流，通过 `SessionProcessor` 处理，并带有 SSE 超时包装来检测过期的服务器发送事件流。

### 对比评估

| 维度 | free-code | OpenCode |
|-----------|-----------|----------|
| **提供商数量** | 5 | 15+ |
| **提供商策略** | 以 Anthropic 为首要，其他适配 | 通过统一 SDK 实现多提供商平等地位 |
| **API 格式依赖** | 耦合于 Anthropic 块格式 | 通过 Vercel AI SDK 实现提供商抽象 |
| **模型发现** | 静态（环境变量切换） | 静态定义 + 动态 npm 包加载 |
| **系统提示** | 单一提示，包含条件部分 | 按提供商系列划分的提示（7+ 文件） |
| **流式传输** | 异步生成器（`for await`） | Effect 原生 Stream |

free-code 深度 Anthropic 集成使得可以针对 Claude 模型进行更精细的优化（例如，理解 `tool_use` 块的细微差别，可感知缓存的提示构造）。OpenCode 的广泛集成支持模型选择灵活性和对提供商中断的弹性，但每提供商提示的维护负担更高。

---

## (g) 上下文管理

### 共同洞见

两个项目都汇聚于同一个根本洞见：智能体的上下文窗口就是马尔可夫状态，而主要的架构挑战是保持其有界。两者都实现了三级上下文管理策略，但侧重点不同。

### free-code：三级压缩，Haiku 摘要

**第一级——微压缩**：按消息、按结果应用。超过可配置 TTL 的旧工具结果被替换为 `[Old tool result content cleared]`。图像结果被压缩到 2000 token 限制。大型文本结果被摘要化。九种工具类型获得特殊的微压缩处理（Bash、Read、Grep、Glob、WebSearch、WebFetch、Edit、Write，再加上通用处理）。策略是**基于时间的**（TTL）和**基于大小的**（截断）。

**第二级——全量压缩**：当微压缩不足且 token 数量超过阈值时，使用摘要提示分叉一个子智能体（Claude Haiku，出于速度和成本考虑）："摘要对话到目前为止的内容。包括：关键决策、已修改的文件、当前任务、下一步。"预边界消息被替换为摘要，并由 `compact_boundary` 系统消息标记过渡。`getMessagesAfterCompactBoundary()` 函数确保 LLM 只看到压缩后的上下文。

**第三级——片段压缩**：对于具有极长对话的 SDK/无头调用者，片段压缩（`snipCompact.ts` / `snipProjection.ts`）通过识别"片段边界"并投射压缩视图来提供激进的历史修剪。

`/context` 斜杠命令为用户提供透明性：应用 API 所见的相同变换，运行微压缩以显示确切的消息形状，并渲染上下文分配的彩色条形可视化。

### OpenCode：结构化摘要与修剪

**第一级——修剪**：工具输出被截断为 2000 字符（`TOOL_OUTPUT_MAX_CHARS`），完整输出存储在基于文件的溢出中，保留 7 天。这是基于内容长度的，而非基于时间的。

**第二级——压缩**：当总 token 数超过可用空间（上下文限制减去预留空间）时，生成具有特定部分的结构化压缩摘要：
```
## Goal（目标）              —— 我们要达成什么？
## Constraints（约束）       —— 什么规则支配我们的行动？
## Progress（进展）          —— 我们现在在哪里？
## Key Decisions（关键决策）  —— 我们决定了什么，为什么？
## Next Steps（下一步）       —— 接下来做什么？
## Critical Context（关键上下文）—— 什么技术事实重要？
## Relevant Files（相关文件） —— 哪些文件在发挥作用？
```

压缩会保留：系统提示（始终保留）、最近的尾部轮次（默认最近 2 个）、受保护的 `skill` 工具调用，以及至少 2000 token 的最近上下文。`PRUNE_MINIMUM`（20,000 token）和 `PRUNE_PROTECT`（40,000 token）常量控制策略。

**第三级——截断**：针对极端溢出场景的最终安全保障。

### 关键差异

| 维度 | free-code | OpenCode |
|-----------|-----------|----------|
| **微级策略** | 基于时间的 TTL + 基于大小的截断（9 种工具类型） | 基于长度的截断（2000 字符）+ 溢出存储 |
| **压缩摘要器** | Haiku 子智能体（LLM 生成的散文式摘要） | 具有预定义部分的结构化模板 |
| **压缩触发** | Token 阈值（自动） | Token 阈值（自动）+ 手动触发 |
| **用户可见性** | `/context` 命令，带条形可视化 | 隐式（没有专用的可视化工具） |
| **子智能体隔离** | 子智能体拥有独立的上下文窗口 | 带有父子数据库关系的子会话 |

free-code 的微压缩（9 种工具类型，每种类型的策略不同）比 OpenCode 的统一 2000 字符修剪更精细。然而，OpenCode 的结构化压缩模板比 free-code 的 Haiku 生成散文更可靠——结构化格式确保关键部分（目标、约束、下一步）始终存在，而 Haiku 摘要可能不可预测地遗漏细节。这是 OpenCode 的"平台"哲学（结构化数据、机器可读输出）与 free-code 的"产品"哲学（LLM 生成的自然语言、人类可读但质量可变）对比的最佳范例。

---

## (h) 通信模型

### free-code：MCP 集成作为外部扩展

free-code 的组件间通信主要是**直接且同步的**。QueryEngine 直接调用工具，hooks 直接观察状态变化，REPL 直接消费异步生成器。没有内部事件总线——组件通过共享状态（AppState store）和直接函数调用进行通信。

外部通信使用 **Model Context Protocol（MCP）**。智能体可以：
- **将自身暴露为 MCP 服务器**（`src/entrypoints/mcp.ts`），允许其他工具将其作为工具提供者调用
- **消费外部 MCP 服务器**（`src/services/mcp/`），在不修改源代码的情况下扩展其工具生态系统

MCP 作为 free-code 跨进程通信的扩展机制，但在进程内部，通信是状态驱动的（AppState 变更 + 订阅者通知）。

### OpenCode：Bus 作为内部解耦层

OpenCode 的通信模型围绕一个**发布-订阅 Bus**（`bus/` 模块），解耦内部组件：

```typescript
// Effect 服务模式（来自 bus/index.ts）
export class Service extends Context.Service<Service, Interface>()("@opencode/Bus") {}
```

Bus 中介了以下内容：
- **会话变更**：当会话更新时，Bus 发出 `session.updated` 事件，由 UI 组件消费
- **权限事件**：工具执行权限请求通过 Bus 流转以进行异步用户审批
- **工具完成**：当工具执行完成时，Bus 发出完成事件
- **压缩事件**：`session.compacted` 事件通知观察者

这种解耦实现了：
- **多终端 UI**：同一 Bus 事件同时驱动终端 UI、Web UI 和桌面 UI
- **插件可扩展性**：插件可以订阅 Bus 事件而无需修改核心代码
- **异步权限流程**：用户审批请求可以由任何注册的处理程序处理，不仅仅是终端 REPL

V2 架构通过**事件溯源同步**对此进行了扩展——事件不仅被发布，而且被持久化，从而实现跨实例协调。

### 对比评估

| 维度 | free-code | OpenCode |
|-----------|-----------|----------|
| **内部通信** | 直接函数调用 + 共享 AppState store | 发布-订阅 Bus |
| **外部通信** | MCP 服务器/客户端 | 插件 SDK |
| **解耦级别** | 低（单体，直接访问） | 高（服务通过 Bus 通信） |
| **多终端支持** | 仅 Ink 终端（无头 SDK 共享生成器 API） | Bus 支持同步的终端、Web、桌面 UI |
| **异步权限流程** | Tool 接口中的同步 `needsApproval()` 钩子 | 基于 Bus 的异步 Deferred 模式 |

free-code 的直接通信模型更简单、更快——组件之间的抽象更少。OpenCode 基于 Bus 的模型更加解耦和可扩展——可以添加、移除或替换组件而不触及消费者。Bus 对 OpenCode 的多终端架构至关重要；free-code 的模型要支持 Web UI 与终端并行将需要大量重构。

---

## (i) 马尔可夫分解分析

### 共享的架构模式

free-code 和 OpenCode 都独立地汇聚于相同的三层模式，将非马尔可夫的软件工程分解为马尔可夫的 LLM 轮次：

```
非马尔可夫世界（完整代码库，数百万 token）
         │
         ▼ 第 1 层：类 Bash 通用接口
         │ ── 查询接口：提取相关状态切片
         │ ── 有界输出：确定性、截断
         │ ── 状态变更：write、edit、apply_patch
         │
         ▼ 第 2 层：工具系统作为按需状态查询
         │ ── 每个工具结果是一个自包含的快照
         │ ── 工具将无界状态转换为有界观察
         │ ── 工具结果成为消息历史中的不可变事实
         │
         ▼ 第 3 层：压缩作为历史压缩
         │ ── 旧轮次被替换为结构化摘要
         │ ── 摘要捕获语义状态，丢弃操作细节
         │ ── 上下文窗口成为马尔可夫状态边界
         │
         ▼ 马尔可夫上下文（每轮次，有界 token）
         ├── 系统提示（每个会话不可变）
         ├── Todo 列表（显式任务状态）
         ├── 工具调用 → 结果对（最近的情节）
         ├── 压缩摘要（历史压缩）
         └── 环境（工作区、git、平台）
         │
         ▼ LLM 决策（无状态：上下文 → 行动）
```

### 评估：逐层分析

#### 第 1 层：Bash 接口作为不可约简的核心

两个项目都独立地将 Bash 工具确定为**不可约简的通用接口**。它充当逃生舱、组合层以及外部现实与内部上下文之间的桥梁。这不仅方便——它在架构上是基础性的：

- **完备性**：任何没有专用工具的操作都可以通过 Bash 完成
- **LLM 原生**：LLM 在 shell 命令上进行了预训练；Bash 直接映射到训练分布
- **可组合性**：Shell 管道组合了原本需要多次专用工具调用的操作
- **无状态重查询**：每个 bash 命令从文件系统重新推导所需状态——LLM 永远不需要追踪完整状态

free-code 的 Bash 工具（3,123 行）是该生态系统中最复杂的工具，具有基于 AST 的安全解析、后台执行、沙盒路由和文件历史追踪。OpenCode 的 ShellTool 同样复杂，具有 tree-sitter AST 解析、PTY 支持、分类命令集（FILES、CWD、CMD_FILES）和 SIGTERM→SIGKILL 升级。

**此层在两个项目中本质上是相同的**——这是架构收敛的标志：面对相同的问题（LLM + 代码库），相同的解决方案（Bash 作为通用接口）自然涌现。

#### 第 2 层：工具系统作为状态机接口

这里项目开始出现分歧：

**free-code** 将工具视为**被动执行器**：LLM 决策，工具执行，结果回流。`ToolUseContext` 聚合所有状态（权限、会话、文件缓存）供工具按需访问。工具结果是 Anthropic 内容块——结构化但特定于 API 格式。

**OpenCode** 将工具视为**由 Effect 中介的状态转换**：每个工具在 Effect 运行时内运行，具有类型化错误通道、资源作用域和结构化并发。`plan` 工具提供显式的"进入计划模式"/"退出计划模式"状态转换——使智能体的操作模式成为马尔可夫状态的一部分。`todowrite` 工具为 `Tool.Def` 贡献元数据追踪，使 todo 状态成为上下文的一等部分。

OpenCode 还引入了**工具级并发控制**：`edit` 工具使用信号量锁定，`apply_patch` 工具使用文件级锁定——防止 LLM 在发出并发工具调用时意外创建竞态条件。free-code 没有等价机制；它依赖 LLM 正确排序工具调用。

#### 第 3 层：压缩作为马尔可夫属性执行器

两个项目的压缩系统都体现了马尔可夫属性：丢弃完整历史，替换为捕获"发生了什么"而不包含"如何发生"的紧凑摘要。关键差异在于**摘要格式**：

- **free-code**：Haiku 生成的自然语言散文 → 灵活、类人，但依赖于质量且可能丢失信息
- **OpenCode**：具有 7 个固定部分的结构化模板 → 刚性、机器可解析，可靠地保留关键维度

对于马尔可夫分解目标，OpenCode 的结构化方法客观上更可靠：每次压缩都保证目标、约束、进展、决策、下一步、关键上下文和相关文件被保留。free-code 的 Haiku 摘要可能在细微之处更丰富，但可能不可预测地遗漏关键细节。

### 哪个项目实现了更好的分解？

两个项目都实现了**实用的马尔可夫分解**，但失败模式不同：

| 失败模式 | free-code | OpenCode |
|--------------|-----------|----------|
| **压缩质量差异** | 高：Haiku 摘要质量各异 | 低：结构化模板保证覆盖率 |
| **压缩边界的信息丢失** | 可能错过细微的决策或失败的实验 | 模板部分约束了被保留的内容 |
| **工具竞态条件** | 可能发生（无并发控制） | 被阻止（信号量锁定） |
| **子智能体状态隔离** | 干净（独立上下文窗口） | 干净（父子会话关系） |
| **长运行会话退化** | 积累过时的微压缩结果的风险 | 模板疲劳的风险（重复压缩丢失细微之处） |

**结论**：OpenCode 通过结构化压缩和工具级并发控制实现了更可靠的马尔可夫分解，但两个项目都不完美——两者都在压缩边界丢失信息，并且退化在非常长的会话中会累积。

### 未解决的张力

两个项目都面临相同的未解决张力：**压缩本质上是丢失信息的，而某些形式的信息丢失比其他形式更严重**。一个压缩摘要可能包含"重构了 auth 模块"，但丢失了"重构引入了一个微妙的 JWT 过期时区 bug"。两个项目都将受益于：

1. **压缩边界的验证**：压缩后运行测试套件以验证状态一致性
2. **可逆压缩**：能够"展开"压缩摘要以恢复完整历史（事件溯源，正如 OpenCode 的 V2 正在构建的方向）
3. **关键上下文标记**：明确标记关键观察为"压缩时不要丢失此项"

OpenCode 的 V2 事件溯源架构是更有前景的长期方向——它支持完整的审计追踪和重放能力，最终可能通过允许从事件重新推导历史上下文而不是丢弃它，使压缩变为非丢失性的。

---

## (j) 评分表

每个维度评分 1–10，10 代表在构建通用 AI 编码智能体这一目标上该类别表现卓越。

| 维度 | free-code | OpenCode | 评注 |
|-----------|:---------:|:--------:|------------|
| **架构复杂性** | 7 | 9 | free-code 的单体应用因约 1,934 文件的耦合而复杂，但可理解。OpenCode 的 18 包 monorepo + Effect-TS + V2 事件溯源更复杂，但组织得更好。 |
| **工具精细度** | 8 | 8 | free-code 拥有更多工具（~50 vs 17+），更深入的每工具功能（9 类微压缩）。OpenCode 拥有更少但设计更精心的工具，具有基于 Effect 的错误处理、并发控制（信号量锁定）和惰性加载。 |
| **状态管理** | 6 | 9 | free-code 的类 Redux store 简单但临时的，没有持久化或类型安全的错误处理。OpenCode 的 Effect DI + SQLite 持久化 + V2 事件溯源显著更健壮，代价是复杂性。 |
| **上下文工程** | 8 | 8 | free-code 的具有每工具微压缩的三级压缩很精细。OpenCode 的结构化压缩模板在保留关键上下文方面更可靠。不同优势：free-code 在增量修剪方面更优，OpenCode 在摘要可靠性方面更优。 |
| **可扩展性** | 7 | 9 | free-code 支持技能、插件和 MCP——扎实的扩展点。OpenCode 的插件 SDK + 18 包 + Bus 解耦 + Agent.generate() 使其作为平台从根本上更可扩展。 |
| **代码质量** | 7 | 8 | free-code 是生产级加固的（由 Anthropic 构建），但存在单体耦合、34 个损坏的功能标志和临时依赖注入。OpenCode 强制执行严格的 TypeScript、无 `any`、函数式风格和 Effect 的类型化错误处理——但 Effect 的学习曲线造成可访问性障碍。 |
| **马尔可夫分解有效性** | 7 | 8 | 两者都实现了实用的马尔可夫分解。free-code 的子智能体隔离干净，微压缩粒度细，但基于 Haiku 的摘要可能丢失关键细节。OpenCode 的结构化摘要更可靠，V2 事件溯源承诺无损重放，工具级并发控制防止竞态条件。 |

**综合得分**：free-code：**50/70**（71.4%）| OpenCode：**59/70**（84.3%）

OpenCode 整体得分更高，源自更优越的状态管理、可扩展性和更可靠的马尔可夫分解。free-code 的优势集中在其深厚的工具生态系统和流式架构上，但在状态持久化、代码模块化和系统验证方面落后。

---

## (k) 使用场景建议

### 何时选择 free-code

**1. 单人、以终端为中心的开发**
如果你是一个在终端中工作的独立开发者，需要最精致、针对 Anthropic 优化的编码智能体体验，free-code 的 Ink/React 终端 UI、深度 Claude 集成和全面的工具生态系统提供最佳的开箱即用体验。

**2. Anthropic 生态系统的用户**
已在 Anthropic API（API 密钥、Bedrock 路由或 Foundry 企业部署）上投入的组织会发现 free-code 的提供商集成更自然——工具语义原生匹配 Anthropic API 格式，无转换层开销。

**3. 使用功能标志进行快速原型开发**
free-code 的 88 个功能标志支持激进的实验。如果你需要快速原型化实验性智能体功能（语音模式、token 预算、智能体集群），并接受并非所有标志都能编译的事实，free-code 的 bun:bundle 死代码消除是实验的优雅载体。

**4. 状态模型简单性很重要时**
如果你更喜欢简单的类 Redux 状态模型而不是学习单子效应系统，free-code 的 `AppState` + `createStore` 模式可以立即理解和调试。

### 何时选择 OpenCode

**1. 多终端部署**
如果你需要同一个智能体引擎同时驱动终端 UI、Web 应用、桌面封装器和 Slack 机器人，OpenCode 的 monorepo 和基于 Bus 的解耦是为此目的而构建的。18 包架构明确针对多终端部署。

**2. 以平台和可扩展性为重点**
如果你在编码智能体之上构建产品，而不是使用编码智能体，OpenCode 的插件 SDK、JavaScript/TypeScript SDK 和 `Agent.generate()` 动态智能体创建使其成为更优的基础。包边界强制执行清晰的扩展 API。

**3. 多提供商弹性**
需要切换 15+ 个 LLM 提供商、根据成本/性能/能力路由流量或避免供应商锁定的组织应选择 OpenCode——Vercel AI SDK 抽象提供最广泛的提供商覆盖。

**4. 长时间运行或分布式会话**
如果会话必须跨进程重启持久化、支持并发用户或需要审计追踪，OpenCode 的 SQLite 持久化、V2 事件溯源和父子会话关系提供了必要的基础设施。free-code 的内存状态模型不足以满足这些需求。

**5. 函数式编程团队**
熟悉 Effect-TS（或愿意投入学习曲线）的团队将受益于 OpenCode 的类型化错误通道、结构化并发、资源作用域和 OpenTelemetry 可观测性——这些功能在类型层面防止整类错误。

### 混合建议

对许多组织来说，理想的方法可能是**以 OpenCode 作为平台基础，以 free-code 级别工具精细度作为目标**。OpenCode 的架构（monorepo、Effect 服务、Bus 解耦、V2 事件溯源、多提供商支持）提供了更好的长期基础，但 free-code 更深入的工具生态系统（50 个工具、每工具微压缩、后台任务管理、Notebook 编辑）代表了更丰富的即时功能集。一条务实的迁移路径是部署 OpenCode，并逐步将 free-code 最有价值的工具（高级 Bash 安全解析、文件历史追踪、后台任务输出管理）移植为 OpenCode 插件。

---

*报告生成自对 free-code（版本 2.1.87）和 OpenCode（版本 1.14.41）各自独立架构报告的直接对比分析。所有论断均与源分析交叉验证。*
