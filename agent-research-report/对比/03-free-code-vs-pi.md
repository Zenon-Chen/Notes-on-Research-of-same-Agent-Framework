# 成对比较：free-code (Claude Code) vs Pi 编码代理

## 通过马尔可夫分解视角对两个终端原生 AI 编码代理的比较分析

**分析日期：** 2026年5月  
**来源：**
- 独立报告：`01-free-code-analysis.md`（free-code v2.1.87 — 无遥测的 Claude Code 分支）
- 独立报告：`04-pi-analysis.md`（Pi v0.74.0 — Mario Zechner 的自扩展代理框架）

---

## A) 执行摘要

free-code 和 Pi 代表了编码代理设计中两种根本不同的哲学理念。free-code（Anthropic 的 Claude Code 的一个分支）是一个**生产最大化**代理：约50种工具、约1,934个源文件、单体 Bun/TypeScript 代码库、子代理编排、多提供商 API 路由，以及源自 Anthropic 内部工程团队多年实战测试的三层压缩系统。Pi 是一个**极简但可扩展**的代理：7种内置工具、5个清晰分层的 monorepo 包、通过注册模式支持的22+ LLM 提供商、树形结构的 JSONL 会话，以及一个压缩系统，尽管规模仅为 free-code 的一小部分，但在架构上可以说更加严谨。

两个代理汇聚于同一个关键洞见——**马尔可夫分解命题**：编码代理必须将软件开发本质上是非马尔可夫的（正确下一步编辑依赖于对整个代码库和对话历史的理解）问题分解为一系列马尔可夫状态（LLM 的下一步输出仅依赖于当前上下文窗口）。两者都通过类 bash 接口作为外部文件系统状态与内部 LLM 上下文之间的桥梁来实现这一点，并通过压缩系统将无界的对话历史压缩为有界的摘要。

它们的分歧在于**各自优化的目标**。free-code 优化的是**能力的广度和威力**——你几乎可以做任何事情，子代理意味着你可以将任意复杂的任务进行分解。Pi 优化的是**架构的纯粹性和可扩展性**——核心是极简的，一切都可以扩展或替换而不需要分支代码，会话格式专为共享和分析而设计。free-code 对一项新需求的回答是"我们已经有一个工具来做这件事"。Pi 的回答是"你可以构建一个扩展来做这件事"。

本报告提供了一个结构化的、逐维度的比较，最终给出包含 1–10 评分的判决表和用例推荐。

---

## B) 架构比较：单体 vs 5包 Monorepo

### free-code：久经沙场的单体

free-code 位于单个 `src/` 目录中，大约有 1,934 个 TypeScript/TSX 文件。它**不是** monorepo——没有作用域包、没有工作区配置、没有独立版本管理。整个应用程序是单个 Bun 打包的二进制文件。关键模块通过直接导入和共享类型进行通信，依赖注入是临时性的而非系统性的。

架构按功能而非包边界组织：

```
src/
├── entrypoints/     # CLI、初始化、MCP 服务器、Agent SDK
├── screens/         # REPL.tsx — Ink/React 终端 UI 循环
├── QueryEngine.ts   # 核心代理生命周期（1,177行）
├── Tool.ts          # 基础工具接口（290+行）
├── tools/           # ~45 个独立工具实现
├── services/        # api (claude.ts)、compact、analytics、mcp
├── state/           # 类 Redux store + AppState 类型定义
├── utils/           # ~120 个共享工具文件
├── skills/          # Skill 定义和加载
├── plugins/         # 社区扩展系统
└── bridge/          # IDE 集成（VS Code/JetBrains）
```

单体架构能够实现**深度集成**——工具可以访问权限状态、压缩基础设施和会话钩子，而无需跨越包边界。`ToolUseContext` 类型大约有 60 个字段，因为每个子系统都可用。然而，这造成了紧耦合：更改权限模型需要触及 `state/`、`tools/` 和 `services/` 中的多个文件。

### Pi：清晰分层的 Monorepo

Pi 遵循严格分层的 5 包 npm workspaces 架构，具有清晰的依赖方向：

```
ai        （零内部依赖 — 基础层）
 ↑
agent     （依赖 ai — 代理运行时）
 ↑
coding-agent  （依赖 ai、agent、tui — CLI + 会话管理）
 ↑
web-ui    （依赖 agent — 浏览器聊天界面）

tui       （独立 — 独立的终端渲染库）
```

每个包具有单一职责：
- **`pi-ai`**（packages/ai）：多提供商 LLM 抽象，包含约 17 个提供商实现。定义流式原语、类型模式提供器注册表。对代理层零依赖。
- **`pi-agent-core`**（packages/agent）：`Agent` 类、`agent-loop.ts`、消息类型、工具接口和事件系统。仅依赖 `pi-ai`。
- **`pi-coding-agent`**（packages/coding-agent）：`AgentSession` 类（2,752行）、`SessionManager`（JSONL 会话）、压缩引擎（4个文件，约740行）和内置工具实现。依赖 `pi-ai` 和 `pi-agent-core`。
- **`pi-tui`**：自定义终端 UI 渲染库，独立于其他 pi 包。
- **`pi-web-ui`**：基于 Lit 的浏览器聊天界面，通过 RPC 协议连接。

### 比较判决

| 维度 | free-code | Pi |
|-----------|-----------|-----|
| **组织方式** | 按功能的单体 | 按层的 monorepo |
| **耦合度** | 紧 — 约60字段的 ToolUseContext | 松 — 清晰的依赖方向 |
| **可复用性** | 低 — 所有内容都是内部的 | 高 — `pi-ai` 和 `pi-agent-core` 是独立包 |
| **构建系统** | Bun 原生打包与死代码消除 | npm workspaces，每包使用 tsc |
| **学习曲线** | 陡峭 — 需要理解大量相互连接 | 中等 — 每个包可以独立理解 |
| **分支修改的适用性** | 难 — 必须分支整个代码库 | 易 — 通过包扩展，无需分支 |

**胜出者：Pi** 在架构清晰度方面。free-code 的单体在原始集成深度上胜出，但付出了显著的耦合代价。

---

## C) 代理编排模型：QueryEngine vs 带 agentLoop 模式的 Agent 类

### free-code：QueryEngine

`QueryEngine` 类（`src/QueryEngine.ts`，1,177行）是中枢神经系统。它是一个**会话作用域**、独立的类，拥有对话的完整生命周期。其签名方法是 `submitMessage(userInput)`，返回一个产生 `StreamEvent | Message` 对象的 `异步生成器`。

轮次生命周期遵循三个阶段：

1. **查询前阶段**：解析主循环模型，运行查询前钩子，检查上下文预算（必要时触发自动压缩），构建系统初始化消息，处理斜杠命令，加载动态 skills。

2. **查询阶段**：构建系统提示，规范化消息以供 API 使用，调用 `queryModelWithStreaming()`（基于 Anthropic API 的异步生成器），通过停止流式传输、执行工具、产出结果并继续来处理工具调用。

3. **查询后阶段**：运行查询后钩子，累积使用统计，检查预算耗尽，持久化会话转录，产出最终结果。

整个管道是一个 `async function*` 链 — `submitMessage` → `processUserInput` → `query()` → `queryModelWithStreaming()`。这使得能够实时流式传输到 Ink/React 终端 UI，通过 `AbortController` 优雅地传播中止信号，并在交互式（REPL）和无头（SDK）上下文中具有相同的行为。

### Pi：带 agentLoop 的 Agent 类

Pi 的编排分为两个文件：
- **`agent.ts`**：`Agent` 类，持有可变状态（`MutableAgentState`）并提供事件发射器接口。
- **`agent-loop.ts`**：核心循环：`agentLoop` → `streamAssistantResponse()` → `executeToolCalls()` → 重复。

循环是**模板驱动且事件发射的**：

```
用户输入 → transformContext() → convertToLlm() → LLM（流式）
    → 解析工具调用 → executeToolCalls() → 工具结果
    → hasMoreToolCalls? → 是：循环返回
    → 否：getSteeringMessages? → 注入用户干预
    → getFollowUpMessages? → 链接下一项工作
    → agent_end
```

关键差异化特点：

**双层队列系统**：Pi 实现了两个并发消息队列：
- **操控队列**（`steer()`）：在当前工具执行完成后注入消息。用于实时用户干预而不中止正在进行的工作。
- **后续队列**（`followUp()`）：等待代理自然停止后才发送的消息。用于链接任务。

这是比 free-code 的中止并重启模式或斜杠命令重写更优雅的中途干预方式。

**事件流式传输**：Pi 发出丰富的生命周期事件流（`agent_start` → `turn_start` → `message_start/update/end` → `tool_execution_start/end` → `turn_end` → `agent_end`）。扩展订阅这些事件，将 UI 渲染、持久化和工具钩子与核心循环解耦。

### 比较判决

| 维度 | free-code | Pi |
|-----------|-----------|-----|
| **核心模式** | 异步生成器（QueryEngine） | 事件驱动循环（Agent + agentLoop） |
| **用户干预** | 斜杠命令（修改消息） | 双层队列（steer + followUp） |
| **通过事件的可扩展性** | 钩子（查询前/后） | 丰富的生命周期事件流 |
| **SDK 兼容性** | 相同的异步生成器 | 通过 stdin/stdout 的 RPC 协议 |
| **多模式** | REPL + SDK（无头） | 交互式 TUI + Print + RPC + SDK |

**胜出者：Pi** 在中途干预的优雅性和事件驱动解耦方面。free-code 的异步生成器在流式 UX 方面更好，但 `steer()`/`followUp()` 模式在架构上对用户控制更为优越。

---

## D) 工具系统比较：两者都以 Bash 为核心，但抽象方式不同

### 共同基础：Bash 作为核心接口

两个代理汇聚于同一个关键设计洞见：**Bash 工具是通用的后备接口**。LLM 的原生训练分布与 shell 命令对齐。两个代理都将 Bash 用作确定性外部状态（文件系统、git、进程）与概率性内部状态（LLM 上下文窗口）之间的桥梁。

### free-code：全面的武器库（约50种工具）

free-code 默认提供大约 50 个注册工具，分布在 `src/tools/` 下的 45 个子目录中。`Tool` 基类定义了丰富的接口：模式验证（`Zod` → JSON Schema）、权限检查（`isDangerous`、`needsApproval`）、UI 渲染（`renderResult`、`renderCompactSummary`）和异步子代理控制（`userShouldConfirm`）。

**工具类别：**

| 类别 | 工具 |
|----------|-------|
| 文件系统 | Read、Write、Edit、Glob、Grep、NotebookEdit |
| Shell | Bash、PowerShell |
| 网络 | WebFetch、WebSearch |
| 代理编排 | AgentTool、SkillTool、TaskCreateTool、TaskOutputTool |
| 规划 | EnterPlanModeTool、ExitPlanModeV2Tool、TodoWriteTool |
| 自我管理 | ConfigTool、BriefTool、SnipTool、CtxInspectTool |
| IDE/LSP | LSPTool、REPLTool |
| MCP | ListMcpResourcesTool、ReadMcpResourceTool |
| 团队 | TeamCreateTool、TeamDeleteTool、SendMessageTool、ListPeersTool |
| 工具类 | AskUserQuestionTool、ToolSearchTool、SleepTool |

工具根据功能标志（总共88个，54个编译通过）、用户类型、环境变量和运行时能力有条件地包含。注册表（`getAllBaseTools()`）返回一个扁平数组。

**Bash 工具** 具有基于 AST 的命令解析用于安全分析、通过 `spawnShellTask` 的后台执行、沙箱路由、文件历史追踪，以及通过 `EndTruncatingAccumulator` 的输出截断。

### Pi：极简的七种工具（7个内置工具）

Pi 刻意只提供 7 种工具：`read`、`bash`、`edit`、`write`、`grep`、`find`、`ls`。设计理念是："强大的默认工具，但跳过子代理和计划模式等功能"。

工具架构是分层的：
- **`AgentTool<TInput>`**：注册到代理 — 名称、描述、TypeBox 模式、`exec` 函数。
- **`ToolDefinition<TI, TD>`**：编码代理级别 — 渲染、创建、包装。
- **`Operations` 接口**：可插拔的执行 — `BashOperations`、`ReadOperations`、`EditOperations` 等。

这种分层能够在**不分支代码的情况下替换工具执行**。`BashOperations` 接口允许扩展替换 SSH 执行、Docker 容器或沙箱环境：

```typescript
export interface BashOperations {
    exec(command: string, cwd: string, options: {
        onData: (data: Buffer) => void;
        signal?: AbortSignal;
        timeout?: number;
        env?: NodeJS.ProcessEnv;
    }): Promise<{ exitCode: number | null }>;
}
```

Pi 支持两种执行模式：
- **并行**（默认）：工具调用按顺序验证，然后并发执行。
- **顺序**：每个工具在下一个开始之前完成准备、执行和终结。单个工具可以覆盖此行为。

### 比较判决

| 维度 | free-code | Pi |
|-----------|-----------|-----|
| **工具数量** | ~50（全面） | 7（极简） |
| **扩展模型** | Plugins、MCP、Skills | 扩展注册工具、Operations 接口 |
| **可插拔执行** | 仅 MCP（外部服务器） | Operations 接口（本地、SSH、Docker、沙箱） |
| **权限系统** | 深度集成的 allow/deny/ask/bypass | 扩展钩子（`beforeToolCall`） |
| **子代理支持** | 内置（AgentTool、TaskCreateTool） | 设计上无（可扩展） |
| **文件追踪** | 文件历史（FileHistoryState） | 自动提取用于压缩 |
| **模式验证** | Zod → JSON Schema | TypeBox schemas |

**胜出者：free-code** 在原始能力广度方面。**Pi** 在工具执行的架构可扩展性方面（Operations 接口比单独的 MCP 更具可组合性）。

---

## E) 状态管理：Redux vs 可变 AgentState

### free-code：受 Redux 启发的全局 Store

free-code 的 `AppState` 是一个大型的、深度不可变的 TypeScript 接口，管理全局配置和持久化偏好。Store（`src/state/store.ts`）实现了一个简单的可观察模式，包含 `getState()`、`setState(updater)` 和 `subscribe(fn)`。通过 React 上下文（`AppStoreContext`）消费，使用 `useSyncExternalStore` 实现并发模式安全的订阅。

**进入 AppState 的内容**：设置、模型选择、权限规则、UI 状态（statusLineText、expandedView）、功能标志（kairosEnabled）、远程连接状态、任务管理、MCP 配置、文件历史、归属、会话钩子、待办列表、代理定义。

**不进入 AppState 的内容**：关键的是，**对话消息被排除**。`mutableMessages[]` 数组位于 `QueryEngine` 内部 — 它的作用域是对话，而不是应用程序。类似地，读文件缓存（`FileStateCache`）是每个 QueryEngine 的，中止控制器是每个轮次的，API 使用累积器是每个会话的。

这种分离反映了 React/Redux 模式：UI 状态（AppState）与领域状态（QueryEngine 消息）分离。

### Pi：带 JSONL 外部化的可变 AgentState

Pi 的状态管理有两个层次：

**内存中：`MutableAgentState`** — 一个普通可变对象，持有：
```typescript
{
    systemPrompt: string;
    model: Model<any>;
    thinkingLevel: ThinkingLevel;
    tools: AgentTool<any>[];     // getter/setter 复制数组
    messages: AgentMessage[];    // getter/setter 复制数组
    isStreaming: boolean;
    streamingMessage?: AgentMessage;
    pendingToolCalls: Set<string>;
    errorMessage?: string;
}
```

`tools` 和 `messages` 上的 getter/setter 模式使用**防御性复制**—外部调用者不能原地改变状态。这比 Redux 更简单，但实现了相同的目标——防止意外修改。

**磁盘上：JSONL 会话文件** — 所有状态的外部、持久化表示。每个会话是一个 `.jsonl` 文件，其中每一行是一个 JSON 对象，表示对话树中的一个条目。条目类型包括 `SessionMessageEntry`、`CompactionEntry`、`BranchSummaryEntry`、`ModelChangeEntry`、`ThinkingLevelChange` 等。每个条目都有 `id` 和 `parentId` 字段，形成**树形结构**。所有写入都是追加的 — 文件永远不会被原地重写，使会话具有抗损坏性和 git 友好性。

`buildSessionContext()` 函数通过从根节点走到当前叶节点，注入压缩摘要并跳过已丢弃的条目，来重建 LLM 可见的上下文。

**ExtensionContext**：扩展通过 `ExtensionContext` 访问状态，该上下文提供只读代理状态、消息访问、系统提示修改、工具管理、API 密钥解析、会话控制和压缩触发器。该上下文在**会话替换时失效**，以防止陈旧引用。

### 比较判决

| 维度 | free-code | Pi |
|-----------|-----------|-----|
| **内存状态** | 类 Redux 可观察（AppState）+ 每个引擎的消息 | 带防御性复制的可变 AgentState |
| **持久状态** | 会话转录（基于文件，格式未指定） | JSONL 会话文件（树形结构，仅追加） |
| **状态分离** | UI 状态（AppState）vs 领域状态（消息） | AgentState + SessionManager（磁盘） |
| **可审计性** | 中等 — 转录存在但格式是内部的 | 高 — JSONL 可检查、可共享、git 友好 |
| **分支支持** | 无原生分支 | 带有 `parentId` 链和 fork() 的树形结构 |
| **扩展访问** | 会话钩子（查询前/后） | 具有完整状态 API 的 ExtensionContext |

**胜出者：Pi** 在通过 JSONL 会话实现外部化、可检查状态方面。free-code 的 Redux store 很扎实，但 JSONL 方法从根本上更具可审计性和可共享性。

---

## F) LLM 集成：以 Claude 为先 vs 通过 pi-ai 包的多提供商

### free-code：五个提供商，以 Claude 为中心

API 客户端（`src/services/api/claude.ts`，3,419行）通过统一接口处理五个提供商：

- **Anthropic 第一方**（默认，通过 `ANTHROPIC_API_KEY`）
- **OpenAI Codex**（`CLAUDE_CODE_USE_OPENAI=1`）
- **AWS Bedrock**（`CLAUDE_CODE_USE_BEDROCK=1`）
- **Google Vertex AI**（`CLAUDE_CODE_USE_VERTEX=1`）
- **Anthropic Foundry**（`CLAUDE_CODE_USE_FOUNDRY=1`）

提供商选择通过环境变量自动完成。流式路径（`queryModelWithStreaming`）是一个产生 `StreamEvent` 对象的异步生成器。非流式回退（`queryModelWithoutStreaming`）适用于有问题的场景。

系统强制执行多项预算约束：`maxTurns`、`maxBudgetUsd`、`taskBudget`（通过 beta API 标头实现的令牌级预算），以及跨所有轮次的累计 `totalUsage` 追踪。

存在**Claude API 格式锁定**：工具与 Anthropic API 格式（`tool_use` 块、`BetaContentBlockParam`）深度集成。支持非 Anthropic LLM 需要转换层，这可能丢失语义保真度——例如，OpenAI 的工具调用格式在工具结果与工具调用的配对方式上与 Anthropic 不同。OpenAI Codex 适配器处理了这一点，但代表了维护负担。

### Pi：通过注册模式的 22+ 提供商

Pi 的 `pi-ai` 包通过**提供商注册表**模式提供跨约 22 个提供商的统一流式 API：

```typescript
const apiProviderRegistry = new Map<string, RegisteredApiProvider>();

registerApiProvider(provider, sourceId?): void {
    apiProviderRegistry.set(provider.api, {
        provider: { api, stream, streamSimple },
        sourceId,
    });
}
```

每个提供商实现带有 `stream()` 和 `streamSimple()` 函数的 `ApiProvider<TApi, TOptions>`。`streamSimple()` 入口点从模型的 API 标识符解析提供商并委托给适当的实现。

**支持的提供商**：OpenAI 生态系统（openai、azure-openai-responses、openai-codex）、Anthropic（anthropic、amazon-bedrock）、Google（google、google-vertex）、独立（deepseek、mistral、xai、groq、cerebras、fireworks、minimax、moonshotai、kimi-coding）、OAuth/网关（github-copilot、openrouter、vercel-ai-gateway、cloudflare-*、huggingface）、OpenCode（opencode、opencode-go）、小米系列。

API 密钥解析支持多种策略：环境变量、OAuth 流程（用于 GitHub Copilot）、动态解析（按请求，用于过期令牌），以及 `.pi/config.jsonl` 文件。

`SimpleStreamOptions` 接口支持传输选择（`sse`、`websocket`、`websocket-cached`、`auto`）、缓存保留、用于缓存感知后端的会话 ID、思考预算和中止信号。

### 比较判决

| 维度 | free-code | Pi |
|-----------|-----------|-----|
| **提供商数量** | 5（Anthropic、OpenAI、Bedrock、Vertex、Foundry） | 22+ |
| **提供商可扩展性** | 添加新提供商 = 修改 claude.ts | 添加新提供商 = 在注册表中注册 |
| **API 格式** | Anthropic 原生（tool_use 块） | 抽象化（提供商自行适配） |
| **预算控制** | maxTurns、maxBudgetUsd、taskBudget | 基本令牌估算 |
| **提示缓存** | 仅 Anthropic 原生 | 缓存保留提示 + 会话 ID |
| **OAuth 支持** | 无 | GitHub Copilot 设备码 OAuth |
| **传输** | HTTP 流式 | SSE、WebSocket、WebSocket-cached、auto |

**胜出者：Pi** 在提供商广度、可扩展性和传输灵活性方面。free-code 在预算/成本控制和深度 Claude 集成质量方面胜出。

---

## G) 上下文管理：autoCompact/snipCompact vs 压缩 + 分支摘要

### free-code：三层压缩系统

free-code 实现了分层的上下文管理策略：

**第一层：微压缩**（`microCompact.ts`）：在工具结果进入消息历史之前对其进行缩减。适用于 Read、Bash、Grep、Glob、WebSearch、WebFetch、Edit 和 Write 输出。两种策略：
- **基于时间**：旧的工具结果（超过可配置的 TTL）被替换为 `[Old tool result content cleared]`。
- **基于大小**：大型输出被截断/摘要。图像压缩至 2000 令牌。

**第二层：完全压缩**（`compact.ts`）：当微压缩不够时，系统：
1. 确定压缩边界（要保留的最后 N 条消息）。
2. 分支出一个子代理（通常是 Claude Haiku — 快速、廉价），附带摘要提示。
3. 用摘要替换边界前的消息。
4. 插入 `compact_boundary` 系统消息。
5. 运行压缩后钩子。

`compact_boundary` 标记使 `getMessagesAfterCompactBoundary()` 能够识别"当前"上下文 — LLM 永远不会看到陈旧的历史。

**第三层：截断压缩**（`snipCompact.ts`、`snipProjection.ts`）：用于具有极长对话的 SDK/无头调用者。在"截断边界"处进行激进的历史裁剪 — REPL 保留完整历史用于 UI 回滚，但内存中的消息被截断。

`/context` 命令提供用户可见性：应用 API 看到的相同转换，运行 `analyzeContextUsage()` 获取每部分的令牌计数，并通过 Ink 组件渲染彩色条形可视化。

### Pi：迭代压缩 + 分支摘要

Pi 的压缩系统（4个文件，位于 `packages/coding-agent/src/core/compaction/` 下，约740行）更紧凑但可以说更严谨：

**触发检测**（`_checkCompaction()`）：
- **溢出**：LLM 返回上下文溢出错误 → 立即压缩 + 自动重试。
- **阈值**：`contextTokens > contextWindow - reserveTokens`（默认保留：16,384 令牌）。

**截断点检测**（`findValidCutPoints()`）：从压缩边界扫描条目到最后一条条目。有效截断点是 `user`、`assistant`、`custom`、`bashExecution`、`branchSummary`、`compactionSummary` 消息。从不截断在 `toolResult` 处（必须跟随其工具调用）。找到保留最近条目不会超过 `keepRecentTokens`（默认：20,000）的第一个点。

**摘要生成**（`compact()` 通过 `generateSummary()`）：使用结构化提示，包含以下部分：目标、约束、进度（已完成/进行中/受阻）、关键决策、下一步、关键上下文。支持**迭代更新**：当存在先前的压缩时，使用更新提示将新信息合并到现有摘要中，而不是从头开始重新摘要。这在数学上类似于状态更新函数：`state_{t+1} = f(state_t, new_observations)`。

**分支摘要**：当导航会话树时（通过 `fork()` 或 `switchSession()`），Pi 生成正在离开的分支的**分支摘要**。存储为会话文件中的 `BranchSummaryEntry`，确保在探索替代路径时不丢失上下文。

**上下文重建**（`buildSessionContext()`）：从根节点走到叶节点，注入压缩摘要并跳过已丢弃的条目。

**令牌估算**：简单的 `chars/4` 启发式方法。没有令牌器感知的估算 — 对于非英语或代码密集型内容不太准确，但对大多数用例来说已足够。

### 比较判决

| 维度 | free-code | Pi |
|-----------|-----------|-----|
| **层次** | 3（微、全、截） | 1（压缩）+ 分支摘要 |
| **触发机制** | 令牌阈值监控 | 令牌阈值 + 上下文溢出自动重试 |
| **摘要模型** | 专用 Haiku 子代理 | 相同模型，非流式调用 |
| **摘要模板** | 无结构（对话摘要） | 结构化（目标/进度/决策/下一步） |
| **迭代更新** | 无 — 每次完全重新摘要 | 是 — 将新信息合并到现有摘要中 |
| **截断点逻辑** | 简单边界 | 精心保留轮次单元完整性，永不让工具结果成为孤儿 |
| **分支处理** | 无分支概念 | 分支摘要 + 树导航 |
| **用户可见性** | 带可视化的 `/context` 命令 | 事件（`compaction_start/end`） |
| **令牌估算** | 可能很复杂（内置于 Anthropic API） | chars/4 启发式 |

**胜出者：Pi** 在压缩质量方面（迭代更新、结构化模板、截断点精细处理）。free-code 在分层防御（微压缩捕获完全压缩可能遗漏的内容）和用户可见性工具方面胜出。

---

## H) 马尔可夫分解分析 — 关键焦点

两个代理都必须在相同的资源约束下运行：LLM 上下文窗口是有限的（~200K 令牌），而对话历史是无限的。软件开发本质上是非马尔可夫的 — 正确的下一步编辑取决于理解整个代码库和对话历史。两个代理都通过实现四个层次的马尔可夫分解策略来解决这个问题。

### free-code 的分解策略

free-code 实现**四个实践层次**的马尔可夫分解：

**第零层：IDE 状态缓存**：在 LLM 调用之前，`ideContextStore` 收集 IDE 已知的内容（打开的文件，最近的编辑）。被注入到系统提示中，为 LLM 提供最基本的上下文而不进行搜索。

**第一层：查询时构造**：`formatSystemPromptWithContext()` 将系统提示、用户输入、上下文文件和环境状态组装成一个有界提示。这是第一个马尔可夫边界 — LLM 永远看不到无限的代码库，只看到一次查询时准备的一批上下文。

**第二层：工具中介的状态访问**：LLM 不能直接看到文件。它必须使用工具：`Read("file.ts")` 返回特定的代码块；`Grep("pattern")` 返回匹配的行；`Bash("git diff")` 返回文件更改。完整的文件系统状态永远不会在上下文中 — 只有 LLM 明确请求的切片。

**第三层：压缩作为状态压缩**：当对话历史超过上下文窗口时，Haiku 子代理将历史摘要为紧凑形式。`compact_boundary` 标记创建一个"重置点"，LLM 从摘要而非原始历史开始操作。

**第四层：子代理边界**：`AgentTool` 生成具有自己上下文窗口的子代理。父代理永远不会看到子代理的完整历史 — 只有输出。这是嵌套的马尔可夫分解。

**判决：部分实现。** free-code 实现了实用的马尔可夫分解，但质量取决于 Haiku 摘要模型。糟糕的摘要可能丢失关键状态，使分解成为有损的。对话历史仍然是一条完整的链 — 下一轮次依赖于整个历史，而不仅仅是摘要。压缩创建周期性的"重置点"，但这些是软边界，而非硬状态边界。

### Pi 的分解策略

Pi 实现**三个明确层次**的马尔可夫分解，更精确地与理论框架对齐：

**第一层：轮次分解**：每个 LLM 调用是 `f(system_prompt, messages, tools) → output`。LLM 永远不需要推理"整个项目" — 它推理当前轮次。这是纯粹的马尔可夫状态转换：输出仅取决于输入。

**第二层：压缩分解**：当上下文溢出时，单独的 LLM 调用将关键信息提取为结构化摘要。摘要成为马尔可夫状态，取代非马尔可夫历史。关键的是，Pi 使用**迭代更新** — 重新压缩时，它将新信息合并到现有摘要中，而不是从头重新摘要。这在数学上等价于 `state_{t+1} = f(state_t, new_observations)`。

**第三层：会话树分解**：带有 `parentId` 链的 JSONL 会话树实现分支。当用户分叉探索替代方法时，Pi 生成旧分支的分支摘要。每个分支是独立的马尔可夫轨迹 — 状态仅取决于分支点加上后续轮次。

Pi 的压缩提示是明确结构化的用于状态捕获：

```
## 目标：重构 auth 模块以使用 JWT
## 进度
### 已完成
- [x] 创建了 JWT 工具函数
- [x] 更新了登录端点
### 进行中
- [ ] 迁移中间件
## 关键决策
- 使用 RS256 算法进行令牌签名
## 关键上下文
- 当前中间件文件：src/middleware/auth.ts
## 文件
<read-files>src/auth/login.ts\nsrc/auth/types.ts</read-files>
<modified-files>src/auth/jwt.ts\nsrc/auth/login.ts</modified-files>
```

这是**语义状态提取** — 不仅仅是压缩，而是代理所知道和正在做的事情的结构化表示。

**判决：强力实现。** Pi 的三层分解在理论上更加严谨。迭代压缩更新机制、结构化摘要模板、会话树分支和保留轮次单元完整性的截断点逻辑，都反映了为马尔可夫状态管理而进行的刻意工程，而非临时的上下文管理。

### 马尔可夫分解正面对比

| 维度 | free-code | Pi |
|-----------|-----------|-----|
| **分解层次** | 4（查询时、工具、压缩、子代理） | 3（轮次、压缩、会话树） |
| **压缩方式** | 摘要 → 替换历史 | 摘要 → 作为上下文消息注入 + 跳过已丢弃条目 |
| **迭代更新** | 无 — 每次重新摘要 | 是 — 合并到现有摘要中 |
| **摘要结构** | 无结构叙述 | 结构化模板（目标/进度/决策/下一步） |
| **状态外部化** | 转录文件（格式未指定） | 带压缩条目和分支摘要的 JSONL |
| **树形分支** | 无 | 完整会话树，带分支摘要 |
| **压缩质量** | 有损 — 取决于 Haiku 质量 | 有损但可恢复 — 迭代更新减少信息丢失 |
| **微压缩** | 是 — 每个工具结果缩减 | 否 — 仅完全压缩 |
| **理论纯度** | 实用但临时 | 为马尔可夫分解刻意工程 |

**马尔可夫分解总体胜出者：Pi。** 虽然 free-code 有更多层次的防御（微压缩捕获完全压缩遗漏的内容），但 Pi 的架构更明确地体现了马尔可夫分解命题。迭代更新机制、结构化摘要模板、会话树分支和外部化 JSONL 状态存储，都展示了在理论框架指导下做出的架构决策，而不仅仅是实用的上下文管理。

---

## I) 会话管理：内部转录 vs JSONL 会话

### free-code：内部会话存储

free-code 将会话转录持久化到磁盘，`QueryEngine` 从存储的数据重建之前的对话。然而，会话格式是应用程序内部的 — 没有文档化的模式、没有树形结构，也没有对分支或重新访问对话点的明确支持。会话主要作为对话的输入/输出记录存在，而非作为架构原语。

会话相关的功能包括：
- 启动时加载对话历史
- 通过 `--resume` 的 MCTS（蒙特卡洛树搜索）会话延续
- 用于审计追踪的转录持久化

### Pi：JSONL 会话文件作为架构原语

Pi 的 JSONL 会话模型是一级架构关注点。每个会话是一个 `.jsonl` 文件，其中每个条目是一个带类型的 JSON 对象，包含 `id`、`parentId` 和 `timestamp`：

| 条目类型 | 用途 |
|-----------|---------|
| `SessionHeader` | 会话元数据（id、cwd、parentSession） |
| `SessionMessageEntry` | 用户/助手/工具消息 |
| `CompactionEntry` | 摘要 + firstKeptEntryId + tokensBefore |
| `BranchSummaryEntry` | 分支导航摘要 |
| `ModelChangeEntry` | 提供商/模型切换事件 |
| `ThinkingLevelChange` | 思考预算更改事件 |
| `CustomEntry` | 扩展特定数据 |
| `CustomMessageEntry` | 上下文注入的消息 |
| `LabelEntry` | 条目上的命名标签 |
| `SessionInfoEntry` | 显示名称 |

`parentId` 链创建了一个**树形结构**，而不是线性日志。关键能力：
- **`buildSessionContext()`**：通过从根节点走到叶节点重建 LLM 可见的上下文。
- **`getBranch()`**：沿着路径获取所有条目。
- **`getTree()`**：带有分支点的完整树结构。
- **`fork(name)`**：从当前点创建新分支。
- **`switchSession()`**：切换到不同的会话文件。
- **`rename(name)`**：重命名当前会话。

所有写入都是**仅追加的** — 文件永远不会被原地重写。这使得会话：
- **抗损坏**（写入失败不破坏现有数据）
- **Git 友好**（差异仅显示新行）
- **可共享**（Pi 明确鼓励在 Hugging Face 上共享会话以进行现实世界的代理评估）
- **确定性可重建**（给定相同的 JSONL 文件，`buildSessionContext()` 产生相同的上下文）

### 比较判决

free-code 具有会话持久化，但将其视为便利功能。Pi 将会话视为**根本性的架构原语** — JSONL 文件就是外部化的代理状态，实现了分支、共享、分析和确定性重建。**胜出者：Pi。**

---

## J) 判决表（1–10 评分）

| 维度 | free-code | Pi | 备注 |
|-----------|:---------:|:-----:|-------|
| **架构清晰度** | 5 | **9** | Pi 的分层 monorepo 清晰可执行；free-code 的单体紧耦合 |
| **工具生态广度** | **9** | 4 | ~50 种工具 vs 7 种；free-code 原生覆盖更多领域 |
| **工具可扩展性** | 7 | **8** | Pi 的 Operations 接口比 free-code 的 MCP/插件系统更具可组合性 |
| **压缩质量** | 7 | **9** | Pi 的迭代更新 + 结构化模板优于 free-code 的一次性 Haiku 摘要 |
| **上下文管理层数** | **9** | 6 | free-code 的 3 层系统（微/全/截）捕获 Pi 的单层可能遗漏的内容 |
| **LLM 提供商支持** | 5 | **10** | 5 vs 22+ 提供商；Pi 的注册模式使添加提供商变得简单 |
| **状态外部化** | 4 | **9** | 带树形分支的 JSONL 会话 vs 内部转录格式 |
| **马尔可夫分解严谨性** | 6 | **9** | Pi 明确为马尔可夫状态管理工程；free-code 实用但临时 |
| **用户干预模型** | 6 | **8** | Pi 的 steer/followUp 队列比 free-code 的斜杠命令更优雅 |
| **子代理支持** | **9** | 2 | free-code 的 AgentTool 实现分层任务分解；Pi 刻意省略 |
| **权限系统** | **9** | 5 | free-code 的 allow/deny/ask/bypass 深度集成；Pi 依赖于扩展钩子 |
| **流式架构** | **8** | 7 | 两者都强；free-code 的异步生成器链更端到端 |
| **代码库可维护性** | 5 | **8** | Pi 更小的范围和清晰的模块边界使其更易于理解和修改 |
| **文档/可见性** | 7 | **7** | 两者都弱；free-code 有 `/context` 命令；Pi 有内联文档和 README |
| **生态系统规模** | **8** | 3 | free-code（Claude Code）拥有庞大的用户基础和社区；Pi 是小众的 |
| **自我可扩展性** | 6 | **9** | Pi 的"让 pi 构建自己的扩展"理念独特而强大 |
| **生产就绪度** | **8** | 6 | free-code 经过 Anthropic 内部团队实战测试；Pi 较年轻（v0.74.0） |
| **总计（满分170）** | **128** | **129** | 非常接近 — 不同的优化产生几乎相等的分数 |

---

## K) 用例推荐

### 选择 free-code（Claude Code）当：

1. **你需要开箱即用的最大能力。** 拥有约50种工具、子代理、计划模式、MCP 集成、LSP 支持和团队协调，free-code 涵盖了极其广泛的软件工程任务，无需任何配置。

2. **你处理受益于分层分解的复杂多步骤任务。** `AgentTool` 能够为独立子任务生成子代理 — 每个子代理都有自己独立的上下文窗口 — 而 `TodoWriteTool` 提供结构化的任务追踪。如果你的工作流涉及"重构 N 个模块，然后更新测试，然后部署"，free-code 的编排模型能够原生处理。

3. **你需要强大的权限控制。** free-code 集成的权限系统带有 allow/deny/ask 规则、bypass 模式和 plan 模式，提供了对代理可以执行的操作的细粒度控制。这在生产环境中很重要，因为你希望代理建议编辑但不经过审查就不执行它们。

4. **你想要一个成熟、经过实战测试的代理。** 作为 Anthropic 为 Claude 提供的生产伴侣工具，该架构已经针对现实世界的边缘情况进行了强化，包括上下文溢出、API 故障、模型回退链和成本管理。

5. **你使用 Claude 作为主要模型。** 深度 Claude 集成意味着工具调用、流式传输、提示缓存和预算控制可以最优地工作。如果你全力投入 Anthropic 生态系统，free-code 是自然的选择。

### 选择 Pi 当：

1. **你重视架构清晰度，并寻求一个你可以理解和修改的代码库。** Pi 的 5 包 monorepo 具有清晰的依赖方向和约 2,750 行的 `AgentSession` 类（vs free-code 在 1,934 个文件单体中的约 1,177 行 `QueryEngine`），使贡献者可以理解每个部分的工作原理。

2. **你需要多提供商灵活性。** 拥有 22+ 提供商，包括 OpenAI、Anthropic、Google、DeepSeek、Mistral、Groq、Cerebras、GitHub Copilot 等，Pi 可以路由到每个任务的最佳模型。注册模式使添加新提供商变得简单 — 实现 `ApiProvider` 并调用 `registerApiProvider()`。

3. **你想构建扩展，而不是分支代码。** Pi 的扩展系统带有生命周期钩子（每个代理事件、工具调用拦截、上下文修改、UI 自定义），可以在不触及核心代码的情况下实现深度定制。Operations 接口（`BashOperations`、`ReadOperations` 等）允许替换工具执行后端以用于 SSH、Docker 或沙箱环境。

4. **你关心可审计性和状态共享。** Pi 的 JSONL 会话文件带有树形结构历史、压缩条目和分支摘要，使代理的整个状态轨迹可检查和可共享。这对于研究、评估和调试至关重要 — 你可以重放、分析和分叉会话而不丢失上下文。

5. **你想要一个极简、自扩展的代理，不妨碍你的工作。** 拥有 7 种内置工具，没有子代理或计划模式，Pi 的核心足够小，可以在一天内理解。"让 pi 构建自己的扩展"的理念意味着你根据具体需求有机地增长代理能力，而不是承担你可能永远不会使用的约 50 种工具的重量。

### 混合方案

对于具有多样化需求的团队，**互补策略**可能是最优的：对受益于子代理分解和丰富工具集的复杂多步骤重构任务使用 free-code，对需要多提供商灵活性、会话共享或自定义扩展的聚焦、可重现工作流使用 Pi。这些代理不是相互排斥的 — 它们在相同的代码库上操作，填补不同的生态位。

---

## 附录：快速参考比较表

| 特性 | free-code | Pi |
|---------------|-----------|-----|
| **语言** | TypeScript + TSX | TypeScript（严格模式） |
| **运行时** | Bun >=1.3.11 | Node.js |
| **许可证** | MIT（商业产品的分支） | MIT |
| **版本** | 2.1.87 | 0.74.0 |
| **源文件** | ~1,934 | 5 个包 |
| **架构** | 单体 | 分层 monorepo |
| **工具数量** | ~50 | 7 |
| **LLM 提供商** | 5 | 22+ |
| **压缩** | 微 + 全（Haiku）+ 截断 | 迭代结构化 + 分支 |
| **会话** | 内部转录 | JSONL 树形结构 |
| **子代理** | 内置（AgentTool） | 无（设计上） |
| **UI** | Ink/React 终端 | 自定义 TUI + Lit Web UI |
| **模式** | REPL + SDK | 交互式 + Print + RPC + SDK |
| **可扩展性** | Skills、Plugins、MCP | Extensions、Skills、Operations 接口 |
| **马尔可夫纯度** | 实用/部分 | 刻意工程化 |

---

*报告基于对两份独立源代码层面架构报告的综合分析生成。所有声明均可追溯到引用的独立分析及其已验证的源代码。*
