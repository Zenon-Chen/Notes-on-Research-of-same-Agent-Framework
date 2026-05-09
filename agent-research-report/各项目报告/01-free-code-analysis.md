# free-code：全面架构分析

## Claude Code 代理如何通过类 Bash 接口将非马尔可夫项目分解为马尔可夫状态

**分析日期：** 2026年5月  
**源代码版本：** 2.1.87（通过 npm source map 暴露重建）  
**源代码仓库：** `paoloanzn/free-code` —— 一个去除遥测、移除护栏的 Anthropic Claude Code 分支

---

## 目录

1. [项目概览](#a-项目概览)
2. [架构概览](#b-架构概览)
3. [代理编排模型](#c-代理编排模型)
4. [工具系统设计](#d-工具系统设计)
5. [状态管理](#e-状态管理)
6. [LLM 集成](#f-llm-集成)
7. [上下文管理](#g-上下文管理)
8. [非马尔可夫分解分析](#h-非马尔可夫分解分析)
9. [设计模式与理念](#i-设计模式与理念)
10. [优势与不足](#j-优势与不足)
11. [总结表](#总结表)

---

## (a) 项目概览

### 什么是 free-code？

free-code 是 **Claude Code**（Anthropic 的终端原生 AI 编程代理）的一个功能完整、去除遥测的分支。Claude Code 最初是作为闭源商业产品开发的；其源代码于 2026 年 3 月 31 日通过 npm 发行版中的 source map 暴露被重建，随后由社区分叉并清理。

这是一个**生产级编程代理**：它读取、写入、编辑和导航代码库；执行 shell 命令；管理 git 工作流；并协调子代理——全部通过终端接口完成。它使用 LLM（大语言模型）推理支持的复杂编排模型来处理任意规模的项目。

### 关键统计

| 指标 | 值 |
|--------|-------|
| **语言** | TypeScript（严格模式），TSX 用于终端 UI |
| **运行时** | Bun (>=1.3.11) |
| **包管理器** | Bun |
| **源文件** | 约 1,934 个 TypeScript/TSX 文件 |
| **UI 框架** | Ink（面向终端的 React） |
| **LLM 提供商** | Anthropic（Claude Opus 4.6、Sonnet 4.6、Haiku 4.5）、OpenAI Codex（GPT-5.3+）、AWS Bedrock、Google Vertex AI |
| **功能标志** | 共 88 个；54 个可干净编译；34 个需要额外运行时支持 |
| **工具** | 约 50 个已注册（bash、文件读/写/编辑、grep、glob、notebook 编辑、网页抓取、网页搜索、LSP、MCP、任务管理、代理、团队等） |
| **实验性功能** | Ultraplan、语音模式、Token 预算、代理集群、REPL 工具、网页浏览器工具、工作流脚本、协调器模式等 |

### 谁创建的？

原始的 Claude Code 由 **Anthropic**（模型提供商）构建，作为其 Claude 语言模型家族的配套工具。free-code 分支由 GitHub 上的 **paoloanzn** 创建，应用了三类更改：

1. **遥测移除**：所有 OpenTelemetry/gRPC、GrowthBook 分析和 Sentry 错误报告端点均被死代码消除或存根化。
2. **护栏移除**：硬编码的"网络风险"指令块、拒绝模式以及服务器推送的托管设置安全覆盖层被剥离。模型自身的安全训练仍然适用——仅移除了 CLI 级别的提示注入。
3. **功能解锁**：所有 54 个可干净编译的实验性功能标志均已启用。

---

## (b) 架构概览

### 单体架构 vs. Monorepo

free-code 是一个**单一的单体 Bun/TypeScript 应用程序**。它不是一个 monorepo——所有源代码都位于单个 `src/` 目录下，使用 Bun 原生打包。构建系统（`scripts/build.ts`）也是一个 Bun 脚本，在编译时执行功能标志感知的死代码消除。

### 关键模块

```
src/
├── entrypoints/
│   ├── cli.tsx              # CLI 引导，REPL 入口点
│   ├── init.ts              # 首次运行初始化向导
│   ├── mcp.ts               # MCP 服务器入口点
│   └── sdk/                 # 无头 SDK 接口（Agent SDK 集成）
├── screens/
│   └── REPL.tsx             # 主交互 UI 循环（Ink/React 终端）
├── QueryEngine.ts           # 核心代理生命周期和回合管理（1,177 行）
├── Tool.ts                  # 基础工具接口、类型、上下文（290+ 行）
├── tools.ts                 # 工具注册表——定义所有可用工具（389 行）
├── context.ts               # 系统/用户上下文聚合（git 状态、CLAUDE.md）
├── commands.ts              # 斜杠命令注册表
├── state/
│   ├── AppState.tsx          # React 上下文提供者，用于状态管理
│   ├── AppStateStore.ts      # 完整状态类型定义
│   └── store.ts              # 类 Redux 的存储实现
├── services/
│   ├── api/
│   │   └── claude.ts         # Anthropic/OpenAI/Bedrock API 客户端（3,419 行）
│   ├── compact/               # 上下文压缩子系统
│   │   ├── compact.ts         # 完整对话压缩
│   │   ├── microCompact.ts    # 每个工具结果的微压缩
│   │   ├── apiMicrocompact.ts # API 级别的上下文管理
│   │   ├── autoCompact.ts     # 自动压缩触发
│   │   ├── snipCompact.ts     # 长对话的历史裁剪
│   │   └── snipProjection.ts  # 裁剪边界投影
│   ├── analytics/            # 存根化的遥测（GrowthBook、事件）
│   ├── mcp/                  # Model Context Protocol 集成
│   └── tokenEstimation.ts    # 上下文预算的 Token 计数
├── tools/                    # 各工具实现（约 45 个子目录）
│   ├── BashTool/             # Shell 命令执行
│   ├── FileReadTool/         # 文件读取，支持 PDF/图片
│   ├── FileEditTool/         # 字符串替换编辑
│   ├── FileWriteTool/        # 文件创建/覆盖
│   ├── GlobTool/             # 快速文件模式匹配
│   ├── GrepTool/             # 使用正则表达式的内容搜索
│   ├── AgentTool/            # 子代理生成
│   ├── SkillTool/            # 技能加载和调用
│   ├── TaskCreateTool/       # 后台任务创建
│   ├── LSPTool/              # 语言服务器协议集成
│   ├── WebFetchTool/         # URL 内容抓取
│   └── ...（还有 35+ 个）
├── utils/                    # 共享工具（约 120 个文件）
│   ├── Shell.ts              # Shell 执行包装器
│   ├── messages.ts           # 消息规范化、工具配对
│   ├── contextAnalysis.ts    # /context 命令的上下文使用分析
│   ├── queryContext.ts       # 系统提示构建
│   ├── processUserInput/     # 用户输入处理管道
│   └── ...
├── hooks/                    # UI 状态的 React hooks
├── components/               # 终端 UI 组件（Ink/React）
├── skills/                   # 技能定义和加载
├── plugins/                  # 插件系统（社区扩展）
├── bridge/                   # IDE 桥接（VS Code/JetBrains 集成）
├── memdir/                   # 内存目录（从文件注入上下文）
└── tasks/                    # 后台任务管理
```

### 入口点

1. **`src/entrypoints/cli.tsx`** —— 主 CLI 入口点。引导 Bun 运行时，初始化 Ink/React 渲染器，挂载 `REPL.tsx` 屏幕组件，并启动代理输入循环。

2. **`src/entrypoints/sdk/`** —— 无头/Agent SDK 接口，将 `QueryEngine` 暴露为可控组件，用于编程式使用（非交互式终端）。

3. **`src/entrypoints/mcp.ts`** —— 将代理暴露为 MCP（Model Context Protocol）服务器，允许其他工具将其作为工具提供者调用。

---

## (c) 代理编排模型

### QueryEngine：中枢神经系统

代理的核心是 `src/QueryEngine.ts` 中的 `QueryEngine` 类。这是一个独立的、会话作用域的类，拥有对话的完整生命周期：

```
┌──────────────────────────────────────────────────────────┐
│                      QueryEngine                          │
│                                                          │
│  ┌─────────────┐   ┌─────────────┐   ┌───────────────┐  │
│  │  Config     │   │  mutable-   │   │ fileState-    │  │
│  │  (tools,    │   │  Messages[] │   │ Cache         │  │
│  │   model,    │   │  (full      │   │ (readFile     │  │
│  │   budget)   │   │   history)  │   │  state)       │  │
│  └─────────────┘   └─────────────┘   └───────────────┘  │
│                                                          │
│  submitMessage(userInput) ──► async generator ──► yield  │
│                                │            StreamEvent  │
│                                │            │ Message    │
│                                │            │ Result     │
│                                ▼                         │
│                      ┌─────────────────┐                 │
│                      │ processUserInput │                │
│                      │ (slash commands, │                │
│                      │  file expansion, │                │
│                      │  skill loading)  │                │
│                      └────────┬────────┘                 │
│                               ▼                          │
│                      ┌─────────────────┐                 │
│                      │ query() loop    │                 │
│                      │ (per-turn       │                 │
│                      │  iteration)     │                 │
│                      └────────┬────────┘                 │
│                               │                          │
│         ┌─────────────────────┼─────────────────────┐    │
│         ▼                     ▼                     ▼    │
│   ┌──────────┐        ┌─────────────┐      ┌─────────┐  │
│   │ pre-     │        │ claude.ts   │      │ post-   │  │
│   │ query    │        │ queryModel  │      │ query   │  │
│   │ hooks    │        │ (streaming) │      │ hooks   │  │
│   │ + compact│        │             │      │ + flush │  │
│   └──────────┘        └─────────────┘      └─────────┘  │
└──────────────────────────────────────────────────────────┘
```

### 回合生命周期

每次 `submitMessage()` 调用都会发起一个**回合**——一个完整的循环，包含：

1. **查询前阶段**（`src/QueryEngine.ts`，约第 400-540 行）：
   - 解析主循环模型（处理 API 错误的回退链）
   - 运行查询前钩子（用户定义的脚本，用于检查/修改查询）
   - 检查上下文预算——如果接近限制，触发自动压缩
   - 构建系统初始化消息（工具、MCP 客户端、技能、插件）
   - 处理斜杠命令（这些直接修改 `mutableMessages`，不需要 API 调用）
   - 加载通过文件路径发现的动态技能
   - 产出系统消息（初始化、压缩边界、工具摘要）

2. **查询阶段**（分派到 `src/query.ts` 中的 `query()`，由 `QueryEngine` 调用）：
   - 构建系统提示：`fetchSystemPromptParts()` + 自定义/追加提示
   - 为 API 规范化消息：`normalizeMessagesForAPI()` 剥离内部元数据，应用微压缩
   - 调用 `src/services/api/claude.ts` 中的 `queryModelWithStreaming()`——核心异步生成器，从 Claude/OpenAI/Bedrock 逐 token 流式传输
   - 处理工具调用：当模型返回 `tool_use` 块时，代理暂停流式传输，执行工具，产出结果，然后继续

3. **查询后阶段**：
   - 运行查询后钩子（清理输出、记录结果）
   - 累积使用统计（token 计数、成本追踪）
   - 检查预算耗尽（最大美元数、最大回合数、最大 token 数）
   - 将会话记录持久化到磁盘
   - 产出最终结果，包含结构化输出、使用情况、成本和权限拒绝

### 异步生成器模式

整个查询流程使用**异步生成器**（`async function*`），产出 `StreamEvent | Message` 对象。这对架构至关重要：

- **流式传输到 UI**：REPL 在每个事件到达时渲染——无缓冲
- **中止安全**：`AbortController` 通过所有异步层传播取消信号
- **无头兼容**：SDK 路径使用相同的生成器 API，无需 Ink
- **部分交付**：`yield* normalizeMessage(message)` 在回合完成前交付部分助手响应

---

## (d) 工具系统设计

### 工具基类

所有工具都继承 `src/Tool.ts` 中定义的 `Tool` 接口。核心类型结构：

```typescript
// src/Tool.ts（简化）
export type Tool = {
  name: string                          // 例如 "Bash", "Read", "Grep"
  description: string                   // 作为工具描述发送给 LLM
  prompt?: string                       // 额外的系统提示指令
  inputSchema: ToolInputJSONSchema      // Zod schema → JSON Schema 用于 API
  isEnabled(): boolean                  // 功能门控检查
  isReadOnly(): boolean                 // 是否会修改文件系统？
  // 核心执行：
  call(
    input: any,
    context: ToolUseContext,            // 完整上下文对象
    options: {
      signal: AbortSignal
      toolUseID: string
    }
  ): Promise<ToolResultBlockParam | Message>  // 返回工具结果
  
  // UI 渲染（Ink/React）：
  renderResult?(...): React.ReactNode
  renderToolUse?(...): React.ReactNode
  renderCompactSummary?(...): React.ReactNode
  
  // 权限辅助：
  isDangerous?(input: any): boolean
  userFacingName(...): string
  renderPermissionRule(...): React.ReactNode
  
  // 异步子代理控制：
  userShouldConfirm?(...): boolean
  needsApproval?(...): boolean
}
```

### 工具注册表

主工具注册表是 `src/tools.ts`（第 193-251 行）中的 `getAllBaseTools()`。它返回一个扁平的 `Tool` 实例数组，条件性地包含基于以下条件：

- **功能标志**（`feature('PROACTIVE')`、`feature('KAIROS')` 等）
- **用户类型**（`process.env.USER_TYPE === 'ant'` 用于门控内部工具）
- **环境变量**（`ENABLE_LSP_TOOL`、`CLAUDE_CODE_VERIFY_PLAN`）
- **运行时能力**（嵌入式搜索工具在可用时替换 Glob/Grep）

工具集是**可扩展的**：MCP 工具在运行时通过 `useMergedTools.ts` 合并，技能工具是动态发现的。

### 工具类别

| 类别 | 工具 | 描述 |
|----------|-------|-------------|
| **文件系统** | `Read`、`Write`、`Edit`、`Glob`、`Grep`、`NotebookEdit` | 直接的文件系统操作——所有都是有状态的、非马尔可夫的 |
| **Shell** | `Bash`、`PowerShell` | 在有状态 shell 环境中执行任意命令 |
| **网络** | `WebFetch`、`WebSearch` | 外部信息检索 |
| **代理** | `AgentTool`、`SkillTool`、`TaskCreateTool`、`TaskOutputTool` | 子代理编排和任务管理 |
| **规划** | `EnterPlanModeTool`、`ExitPlanModeV2Tool`、`TodoWriteTool` | 结构化思考和分解 |
| **状态/配置** | `ConfigTool`、`BriefTool`、`SnipTool`、`CtxInspectTool` | 代理自我管理 |
| **IDE/LSP** | `LSPTool`、`REPLTool` | 语言服务器集成、交互式 REPL |
| **MCP** | `ListMcpResourcesTool`、`ReadMcpResourceTool` | 外部工具提供者集成 |
| **团队** | `TeamCreateTool`、`TeamDeleteTool`、`SendMessageTool`、`ListPeersTool` | 多代理集群协调 |
| **实用工具** | `AskUserQuestionTool`、`ToolSearchTool`、`SleepTool` | UX 和操作辅助工具 |

### Bash 工具：通用接口

`BashTool`（`src/tools/BashTool/BashTool.tsx`）可以说是生态系统中**最重要的工具**。它包装任意 shell 命令：

```
LLM 决定："我需要检查哪些文件被修改了"
       │
       ▼
tool_use: { name: "Bash", input: { command: "git diff --name-only",
                                    description: "检查被修改的文件" } }
       │
       ▼
BashTool.call() → exec(command) → { stdout: "...", stderr: "...", exitCode: 0 }
       │
       ▼
tool_result: { content: "src/QueryEngine.ts\nsrc/tools.ts\n..." }
       │
       ▼
LLM："我看到两个被修改的文件。让我读取它们……" → tool_use: { name: "Read", ... }
```

Bash 工具的特性：
- **命令解析**：使用基于 AST 的 shell 解析（`parseForSecurity`）来分析命令结构
- **后台执行**：长时间运行的命令可以通过 `spawnShellTask` 后台执行，输出由 `TaskOutputTool` 处理
- **沙盒支持**：在沙盒环境中运行时，命令通过 `SandboxManager` 路由
- **文件历史追踪**：当 Bash 修改被追踪的文件时，`fileHistoryTrackEdit` 会记录
- **输出截断**：`EndTruncatingAccumulator` 将 stdout/stderr 限制在 `TOOL_SUMMARY_MAX_LENGTH` 以内

### Read/Write/Edit 三件套

这三个工具构成了**确定性文件系统接口**：

- **`FileReadTool`**（`src/tools/FileReadTool/FileReadTool.ts`）：读取文件并带行号，支持 PDF 提取（`readPDF`）、图片渲染（通过 `maybeResizeAndDownsampleImageBuffer` 调整大小）、notebook 解析和二进制检测。与 `fileStateCache` 集成以避免重复读取未更改的文件。
- **`FileWriteTool`**：原子性地创建或覆盖文件。支持编码检测、行尾规范化。
- **`FileEditTool`**：使用 `edit` 命令模式执行精确的字符串替换——旧字符串 → 新字符串。在非唯一匹配时失败，强制精确性。

---

## (e) 状态管理

### AppState：全局存储

应用程序状态在 `src/state/AppStateStore.ts` 中定义为一个大型 TypeScript 接口，并通过类 Redux 的存储（`src/state/store.ts`）管理。关键状态切片：

```typescript
// src/state/AppStateStore.ts（简化）
type AppState = DeepImmutable<{
  // 核心配置
  settings: SettingsJson              // 所有用户设置
  mainLoopModel: ModelSetting         // 当前模型（例如 "claude-sonnet-4-6"）
  verbose: boolean
  
  // 权限系统
  toolPermissionContext: ToolPermissionContext  // 模式、允许/拒绝规则、绕过状态
  
  // UI 状态
  statusLineText: string | undefined
  expandedView: 'none' | 'tasks' | 'teammates'
  footerSelection: FooterItem | null
  
  // 代理功能
  kairosEnabled: boolean              // 助手模式标志
  fastMode: FastModeState
  
  // 远程/桥接状态
  replBridgeEnabled: boolean
  replBridgeConnected: boolean
  remoteConnectionStatus: 'connecting' | 'connected' | 'reconnecting' | 'disconnected'
  
  // 任务管理（可变，从 DeepImmutable 中排除）
  tasks: { [taskId: string]: TaskState }
  foregroundedTaskId?: string
  
  // MCP 集成
  mcp: { ... }
  
  // 推测执行（预测性工具执行）
  speculation: SpeculationState
  
  // IDE 集成
  bridgeStatus: BridgeState
  
  // 文件历史
  fileHistory: FileHistoryState
  attribution: AttributionState
  
  // 钩子
  sessionHooks: SessionHooksState
  
  // 待办事项列表
  todoList?: TodoList
  
  // 代理定义
  agentDefinitions: AgentDefinitionsResult
}>
```

### 哪些状态不在 AppState 中

关键的是，**对话消息不在 AppState 中**。消息数组存在于 `QueryEngine.mutableMessages` 中——它的作用域是对话，而不是应用程序。类似地：

- **读取文件缓存**（`FileStateCache`）是每个 QueryEngine 的，不是全局的
- **中止控制器**是每个回合的，不是持久的
- **API 使用累积器**是每个会话的

这种分离是有意的：AppState 代表**代理的配置和持久化偏好**，而 QueryEngine 管理**临时的对话状态**。这反映了经典的 React/Redux 模式，其中 UI 状态（AppState）与领域状态（QueryEngine 消息）分离。

### 存储实现

存储（`src/state/store.ts`）实现了一个简单的可观察模式：

```typescript
// 概念性代码
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
    subscribe: (fn) => { listeners.push(fn); return () => /* 取消订阅 */ }
  }
}
```

这通过 React 上下文（`AppStoreContext`）使用 `useSyncExternalStore` hook 进行消费，以实现并发模式安全的订阅。

---

## (f) LLM 集成

### 多提供商架构

`src/services/api/claude.ts`（3,419 行）中的 API 客户端通过统一接口处理**五个 API 提供商**：

```
                     ┌──────────────────────────┐
                     │     queryModel()          │
                     │  (src/services/api/       │
                     │   claude.ts:709)          │
                     └──────────┬───────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                  ▼
    ┌─────────────────┐ ┌──────────────┐ ┌────────────────┐
    │ Anthropic SDK   │ │ OpenAI Codex │ │ AWS Bedrock    │
    │ (beta.messages. │ │ (自定义      │ │ (BedrockRuntime│
    │  create)        │ │  适配器)     │ │  + proxy)      │
    └─────────────────┘ └──────────────┘ └────────────────┘
```

提供商选择是自动的：
- **默认**：Anthropic 第一方 API（`ANTHROPIC_API_KEY`）
- **`CLAUDE_CODE_USE_OPENAI=1`**：OpenAI Codex 端点
- **`CLAUDE_CODE_USE_BEDROCK=1`**：AWS Bedrock 路由
- **`CLAUDE_CODE_USE_VERTEX=1`**：Google Vertex AI
- **`CLAUDE_CODE_USE_FOUNDRY=1`**：Anthropic Foundry（企业版）

### 流式和非流式路径

API 客户端有两条执行路径：

1. **流式**（`queryModelWithStreaming`，第 752 行）：一个异步生成器，产出 `StreamEvent` 对象（内容块、文本增量、工具使用块、消息增量）。由 REPL 用于实时显示。

2. **非流式回退**（`queryModelWithoutStreaming`，第 709 行）：用于流式传输有问题的场景（bedrock 配合大工具集、API 错误等情况）的传统请求/响应。可配置超时（默认 300 秒，远程会话为 120 秒）。

### 上下文构建

每次 API 调用前，系统构建完整的上下文：

1. **系统提示**：由 `fetchSystemPromptParts()` 构建——包括工具定义、技能描述、MCP 资源列表、CLAUDE.md 内容、git 状态、环境信息（操作系统、日期、工作区路径）和自定义用户提示。

2. **消息**：`normalizeMessagesForAPI()` 处理完整的对话历史：
   - 剥离内部元数据字段（调用者信息、isMeta 标志）
   - 确保每个 `tool_use` 都有对应的 `tool_result`
   - 应用微压缩来缩减大型工具输出
   - 移除 advisor/tool_reference 块（ant 专用的内部路由）

3. **工具**：通过 `toolToAPISchema()` 转换为 Anthropic API 格式，并进行功能标志感知的过滤（被权限规则禁用的工具会被排除）。

### 预算和成本追踪

系统强制执行多项预算约束：
- **`maxTurns`**：最大对话回合数
- **`maxBudgetUsd`**：最大 API 花费，通过 `getTotalCost()` 追踪
- **`taskBudget`**：作为 beta API 头（`task-budgets-2026-03-13`）传递的 token 级别预算
- **Token 使用量**：`totalUsage` 在所有回合中累积，通过 `getModelUsage()` 在最终结果中报告

---

## (g) 上下文管理

### 问题

LLM 具有有限的上下文窗口（默认 200K token，[1m] 后缀模型为 1M）。编程代理在单次长对话中很容易超出此限制。因此，代理必须**管理自己的上下文**——决定保留什么、总结什么和丢弃什么。

### 微压缩：每个工具结果的缩减

第一道防线是**微压缩**（`src/services/compact/microCompact.ts`）。它在工具结果进入消息历史之前对其进行缩减：

```typescript
// src/services/compact/microCompact.ts（简化）
const COMPACTABLE_TOOLS = new Set([
  FILE_READ_TOOL_NAME,    // Read
  ...SHELL_TOOL_NAMES,    // Bash, PowerShell
  GREP_TOOL_NAME,          // Grep
  GLOB_TOOL_NAME,          // Glob
  WEB_SEARCH_TOOL_NAME,    // WebSearch
  WEB_FETCH_TOOL_NAME,     // WebFetch
  FILE_EDIT_TOOL_NAME,     // Edit
  FILE_WRITE_TOOL_NAME,    // Write
])
```

微压缩应用**基于时间**和**基于大小**的策略：

1. **基于时间的微压缩**：旧的工具结果（超过可配置的 TTL）被替换为 `[旧的工具结果内容已清除]`。TTL 通过 `getTimeBasedMCConfig()` 动态调整。

2. **基于大小的截断**：对于图片结果，内容被压缩到 `IMAGE_MAX_TOKEN_SIZE`（2000 token）。对于文本结果，大型输出被总结。

3. **缓存的微压缩**（功能 `CACHED_MICROCOMPACT`）：ant 专用的优化，在会话之间持久化微压缩结果。

### 完全压缩：当需要总结时

当微压缩不够时，**完全压缩**会启动（`src/services/compact/compact.ts`）：

```
对话变得过大……
       │
       ▼
autoCompact.ts：Token 计数超过阈值
       │
       ▼
compact.ts：
  1. 识别压缩边界（要保留的最后 n 条消息）
  2. Fork 一个子代理（Claude Haiku，快速/便宜）使用以下提示：
     "总结到目前为止的对话。包括：关键决策、
      修改的文件、当前任务、下一步。"
  3. 用摘要替换边界之前的消息
  4. 插入 compact_boundary 系统消息
  5. 运行压缩后钩子（清理、缓存刷新）
```

压缩边界标记（`compact_boundary`）至关重要——它告诉 `getMessagesAfterCompactBoundary()` 从哪里开始读取"当前"上下文，确保 LLM 永远不会看到过时或冗余的历史。

### /context 命令

`/context` 斜杠命令（`src/commands/context/context.tsx`）为用户提供上下文使用情况的可视化：

- 应用 API 看到的相同转换：`getMessagesAfterCompactBoundary()` + `projectView()`（CONTEXT_COLLAPSE 功能）
- 运行 `microcompactMessages()` 以获取发送到 API 的确切消息形状
- 使用 `analyzeContextUsage()` 进行分析——每个部分的 token 计数、跨度折叠可视化
- 渲染为 Ink 组件（`<ContextVisualization />`），显示上下文分配的彩色条形图

### 裁剪压缩（HISTORY_SNIP）

对于可能运行极长对话的 SDK/无头调用者，**裁剪压缩**子系统（`snipCompact.ts` / `snipProjection.ts`）提供激进的历史裁剪：

- 识别"裁剪边界"——可以安全删除旧消息的位置
- 投影裁剪历史的压缩视图
- 通过 `snipReplay` 配置回调注入 QueryEngine
- REPL 保留完整历史用于 UI 回滚；只有内存中的消息被裁剪

---

## (h) 非马尔可夫分解分析

### 马尔可夫分解命题

> **命题**：代理框架通过类 Bash 接口将固有的非马尔可夫大型项目分解为纯马尔可夫状态，使状态在有限的上下文窗口内可被 LLM 消化。

**马尔可夫状态**意味着下一个动作仅取决于当前状态，而不取决于完整的历史。**非马尔可夫项目**是指状态太大（数百万行代码、数千个文件、git 历史、构建状态）而无法放入任何实际上下文窗口的项目。

问题是：**free-code 是否成功地执行了这种分解？**

### H.1：free-code 如何分解大型项目

free-code 使用一种**多层分解策略**，涵盖工具设计、上下文管理和代理架构：

#### 第 1 层：查询时状态构建（按需切片）

代理**不**将整个项目预加载到上下文中。相反，它在**查询时**构建状态：

```
每个回合构建的上下文：
├── 系统上下文（src/context.ts:getSystemContext）
│   ├── git 状态（分支、最近提交、修改的文件）——每个对话缓存
│   └── 缓存破坏器注入（ant 专用）
├── 用户上下文（src/context.ts:getUserContext）
│   ├── CLAUDE.md 文件（项目级指令，通过文件系统遍历发现）
│   ├── 操作系统、日期、工作区路径
│   ├── 指令格式（Markdown 或默认）
│   └── 环境目录列表
├── 内存文件（src/memdir/memdir.ts）
│   └── 用户定义的指令文件注入到上下文中
└── 系统提示部分（src/utils/queryContext.ts）
    ├── 工具定义（schema、描述）
    ├── MCP 资源列表
    ├── 技能描述
    ├── 插件摘要
    └── --append-system-prompt / --custom-system-prompt
```

这是**第一种分解**：代理每个回合都以项目状态的紧凑摘要（git 状态 + CLAUDE.md 指令）开始，而不是整个代码库。

#### 第 2 层：工具中介的状态访问（类 Bash 接口）

LLM **无法看到文件**。它必须使用工具来访问状态：

```
LLM 的内部状态：           外部状态访问：
┌──────────────────┐       ┌──────────────────────────┐
│ "我知道           │       │ 文件系统（实际状态）       │
│  QueryEngine.ts  │       │                           │
│  但不了解它的      │  ──►  │   Read("QueryEngine.ts")  │
│  内容"           │  ◄──  │   → 内容块（N 行，        │
│                  │       │     带行号）               │
│                  │       └──────────────────────────┘
│ "还有什么被        │
│  修改了？"        │  ──►  ┌──────────────────────────┐
│                  │       │ Bash("git diff --stat")   │
│                  │  ◄──  │   → 3 个文件被修改         │
└──────────────────┘       └──────────────────────────┘
```

这是**第二种分解**：LLM 只在需要时拉取它需要的状态。完整的文件系统状态永远不会出现在上下文中——只有 LLM 明确请求的切片。

#### 第 3 层：压缩作为状态压缩

当对话历史变得过大时：

```
压缩前：
  用户："修复登录 bug"
  助手：[读取 auth.ts，读取 Login.tsx，读取 api.ts……]
  助手：[Bash "npm test" → 测试输出 2000 行……]
  助手：[编辑 auth.ts → 结果……]
  助手：[编辑 Login.tsx → 结果……]
  ……（10,000+ token 的原始工具输出）

压缩后：
  [压缩边界]
  摘要："修复了 auth.ts（JWT 验证）和
   Login.tsx（错误处理）中的登录 bug。测试通过。下一步：检查
   会话过期。"
```

这是**第三种分解**：完整的执行轨迹（非马尔可夫）被压缩为摘要（马尔可夫）。摘要捕获**语义状态**，同时丢弃操作细节。

### H.2：工具在分解中的角色

每个工具类别在分解中扮演特定角色：

| 工具 | 非马尔可夫状态 | 马尔可夫转换 |
|------|-------------------|--------------------------|
| **Bash** | 运行任意命令；输出可能是数兆字节 | 输出被截断（`TOOL_SUMMARY_MAX_LENGTH`），随时间微压缩 |
| **Read** | 包含数千行的完整文件 | 返回块（offset/limit），在 `FileStateCache` 中跟踪读取状态以避免重复读取 |
| **Grep** | 在整个代码库中搜索模式 | 返回匹配行和文件路径——一个经过相关性过滤的切片 |
| **Glob** | 项目中的所有文件 | 返回排序列表（最多 100 个结果）——仅文件名 |
| **Edit** | 文件修改（状态变更） | 返回旧→新差异——仅捕获增量 |
| **TodoWrite** | 跨多个子任务的任务规划 | 创建结构化待办列表——LLM 可以追踪的马尔可夫计划 |
| **AgentTool** | 具有自己历史的复杂子任务 | 子代理独立运行，返回摘要——嵌套的马尔可夫分解 |

**关键洞察**是每个工具都将一个基本无界的外部状态转换为有界的、可消化的块。LLM 永远不会访问"整个代码库"——它访问的是*它读取的特定文件、它搜索的特定 grep 结果、它请求的特定 bash 输出*。

### H.3：外部状态 vs. 内部状态

free-code 在以下两者之间保持清晰的边界：

**外部状态**（由工具管理，LLM 无法直接看到）：
- 文件系统内容（通过 Read/Glob/Grep 读取）
- Git 仓库状态（通过 Bash 或 git 上下文访问）
- 运行中的进程（Bash 可以生成和后台运行）
- MCP 外部资源（通过 ListMcpResourcesTool/ReadMcpResourceTool 访问）
- 文件历史/编辑追踪（`FileHistoryState`）

**内部状态**（在上下文中对 LLM 可见）：
- 对话消息（完整的工具使用/工具结果链）
- 工具定义（schema 和描述）
- 系统/用户上下文（CLAUDE.md、git 状态摘要、操作系统信息）
- 任务预算和成本追踪
- 待办列表

两者之间的**桥梁**是工具调用机制：

```
外部状态               工具调用               内部状态
─────────────          ──────────            ──────────────
filesystem/            Bash/Read/Grep        tool_result
git repo/              /Write/Edit/Glob      消息
进程/                  输出
                                              │
                                              ▼
                                        LLM 上下文窗口
                                        (200K-1M tokens)
                                              │
                                              ▼
                                        压缩：
                                        旧结果 →
                                        摘要
                                              │
                                              ▼
                                        压缩边界
                                        (马尔可夫状态)
```

### H.4：Bash 接口作为核心设计元素

Bash 工具不仅仅是另一个工具——它是**通用回退接口**。这是设计理念的核心：

1. **完备性**：代理需要的任何没有专用工具的操作都可以通过 Bash 完成。LLM 可以编写 Python 脚本、运行 git 命令、调用包管理器、启动服务器——全部通过 Bash。

2. **对 LLM 自然**：LLM 是在 shell 命令上预训练的。它们用 `ls`、`grep`、`git diff`、`npm test` 来"思考"。Bash 接口直接映射到 LLM 的训练分布，使其成为最可靠的工具。

3. **可组合性**：`Bash("grep -r 'function' src/ | head -20")` 等同于 `Grep("function", "src/")`，但是在一个带有结果过滤的单一操作中。LLM 可以组合 shell 管道，否则需要多次专用工具调用。

4. **无状态的状态性**：像 `git log --oneline -5` 这样的 bash 命令提供 git 历史的快照。LLM 不需要跟踪完整的提交 DAG——它只看到最后 5 个提交。每个 bash 命令从文件系统重新推导出它需要的状态。

5. **逃生出口**：当专用工具有 bug 或局限性时，LLM 回退到 Bash。例如，如果 GlobTool 无法找到具有复杂模式的文件，`Bash("find . -name '*.ts' -newer package.json")` 将始终有效。

### H.5：评估：free-code 是否实现了马尔可夫分解？

**答案：是的，部分且务实地实现了。**

free-code 没有在理论意义上实现纯马尔可夫分解。对话历史仍然是一个完整的工具调用和结果链——下一个回合依赖于整个历史，而不仅仅是当前摘要。然而，它实现了**实际的近似**：

1. **上下文窗口就是马尔可夫状态**。在任何给定回合，LLM 的有效状态由上下文窗口限定。压缩确保这永远不会超过模型限制。

2. **工具按需提供状态**。LLM 不知道完整的项目——它在需要时查询文件系统。每个工具结果都是一个自包含的快照。

3. **压缩创建周期性的"重置点"**。压缩后，LLM 从摘要操作，而不是原始历史。这些摘要是马尔可夫的，因为它们编码了"发生了什么"而不包含逐步跟踪。

4. **子代理强制执行局部马尔可夫边界**。当 `AgentTool` 为子任务生成子代理时，该子代理在其自己的上下文窗口中操作，仅返回结果。父代理永远不会看到子代理的完整历史——只有输出。

其**局限性**在于压缩质量取决于摘要器模型（通常是 Haiku，一个更小/更便宜的模型）。糟糕的摘要可能会丢失关键状态，使得分解是有损的而非无损的。

---

## (i) 设计模式与理念

### 可观察的设计模式

| 模式 | 使用位置 | 描述 |
|---------|------------|-------------|
| **异步生成器** | `QueryEngine.submitMessage()`、`claude.ts` | 整个查询管道是一系列 `yield*` 委托；UI 通过 `for await` 消费 |
| **命令模式** | 工具系统、斜杠命令 | 每个工具/斜杠命令封装一个命名操作，带有 schema 验证的输入 |
| **观察者/发布-订阅** | AppState 存储、会话钩子 | 组件订阅状态更改；钩子观察查询前/后生命周期事件 |
| **策略模式** | API 提供商选择、权限模式 | 不同的 API 后端和权限策略是可互换的策略 |
| **责任链** | `processUserInput()` 管道 | 用户输入流经斜杠命令解析 → 内存文件加载 → 技能发现 → 消息规范化 |
| **装饰器模式** | `withStreamingVCR`、`withRetry` | API 调用被包装以回放缓存和重试逻辑 |
| **工厂模式** | 工具创建、代理生成 | `getAllBaseTools()`、`createSubagentContext()` 产生已配置的实例 |
| **响应式模式** | Ink/React UI、`useSyncExternalStore` | 终端 UI 通过 React 并发模式兼容的 hooks 响应状态变化 |

### 涌现的设计理念

通读约 1,934 个源文件，几个哲学原则浮现出来：

1. **LLM 是决策引擎**：架构围绕 LLM 是智能代理这一假设设计。所有工具都是被动的——它们执行 LLM 决定的内容。代理框架提供基础设施，而非策略。

2. **文件系统作为真相来源**：尽管有各种内部状态管理，正确性的最终仲裁者是文件系统。修改文件的工具（Write、Edit、Bash）总是报告结果。没有"虚拟文件系统"——操作是真实的、不可逆的、外部可验证的。

3. **权限作为设计约束**：权限系统（`ToolPermissionContext`）深度集成。每个工具调用都经过 `canUseTool()`，它检查允许/拒绝规则、权限模式（default/acceptEdits/bypassPermissions/plan）和钩子拦截器。这不是事后才想到的——它编织在工具执行路径中。

4. **功能标志作为架构**：使用 `bun:bundle` 的 `feature()` 进行死代码消除意味着实验性功能在禁用时运行时成本为零。这使得可以在不膨胀生产二进制文件的情况下进行积极的实验。在 88 个标志中，54 个可以干净编译——61% 的成功率表明存在健康的功能门控文化。

5. **通过插件和 MCP 实现可扩展性**：代理支持三种扩展机制：
   - **技能**（`src/skills/`）：基于上下文中的文件路径激活的专门指令
   - **插件**（`src/plugins/`）：具有自己生命周期管理的社区扩展
   - **MCP**（`src/services/mcp/`）：通过 Model Context Protocol 提供的外部工具服务器

6. **尽可能确定性**：`FileEditTool` 使用精确字符串匹配（不是正则表达式，不是行号）来防止歧义。如果 `oldString` 匹配多次，工具会失败。`GlobTool` 按修改时间排序返回文件——确定性的、可复现的。

7. **可观察性作为一等关注**：代理发出大量的诊断日志（`logForDiagnosticsNoPII`、`headlessProfilerCheckpoint`），维护成本追踪（`cost-tracker.ts`），报告按模型的 token 使用量（`getModelUsage`），并提供 `/context` 命令用于上下文窗口可视化。遥测在此分支中被移除，但可观察性基础设施仍然存在。

8. **多 LLM 级联**：系统战略性地使用多个 LLM 模型：
   - **Sonnet/Opus**：主循环——主要推理引擎
   - **Haiku**：压缩摘要、快速分类任务
   - **Codex**：当 `CLAUDE_CODE_USE_OPENAI=1` 时——替代后端
   - **顾问模型**（ant 专用）：一个单独的分析模型，审查主模型的决策

---

## (j) 优势与不足

### 优势

1. **全面的工具生态系统**：拥有约 50 个工具，涵盖文件系统、shell、git、搜索、LSP、MCP 和代理编排，代理几乎可以执行任何软件工程任务。Bash 工具作为没有专用工具的操作的通用逃生出口。

2. **健壮的上下文管理**：三层压缩系统（微压缩 → 完全压缩 → 裁剪压缩）可以处理任意长度的对话。上下文可视化（`/context`）使用户对上下文分配有透明度。

3. **多提供商灵活性**：支持五个 API 提供商（Anthropic、OpenAI、Bedrock、Vertex、Foundry），使代理对提供商中断具有弹性，并允许成本优化。

4. **权限安全网**：集成的权限系统，带有允许/拒绝/询问规则、绕过模式和计划模式，使用户对代理可以执行的操作有细粒度控制。

5. **流式架构**：全程使用异步生成器，实现实时 UI 更新、优雅的中止处理，以及使用相同代码路径的无头 SDK 兼容性。

6. **功能标志工程**：`bun:bundle` 的死代码消除方法是优雅的——实验性功能在禁用时编译为零，保持二进制文件精简。

7. **子代理组合**：`AgentTool` 允许任务的层次化分解——父代理可以为独立的子任务生成子代理，每个子代理都有自己的上下文窗口。

8. **MCP 集成**：对 Model Context Protocol 的支持意味着工具生态系统可以通过外部服务器扩展，而无需修改代理源代码。

### 不足

1. **单体代码库**：约 1,934 个源文件，代码库庞大且紧密耦合。`ToolUseContext` 类型有约 60 个字段。依赖注入是临时的而非系统化的。

2. **没有形式化验证**：工具执行除了 schema 验证和权限检查外不提供验证。没有对工具输出进行沙盒化（例如，确认 Write 工具没有损坏文件）。

3. **压缩是有损的**：完全压缩依赖 Haiku 模型来总结对话历史。如果摘要遗漏了关键细节，代理可能会不可恢复地丢失上下文。这是马尔可夫分解链中最薄弱的环节。

4. **没有结构化任务验证**：`VerifyPlanExecutionTool` 存在但被功能标志（`CLAUDE_CODE_VERIFY_PLAN`）门控。没有系统化的方法来验证代理是否完成了它说会完成的事情。

5. **Bash 工具是无界的**：Bash 工具可以执行任意命令，在运行时间、输出大小或副作用方面没有上限。虽然权限模式门控 Bash 使用，但一旦获得许可，代理就拥有完整的 shell 访问权。

6. **工具级别的供应商锁定**：工具与 Anthropic API 格式深度集成（`tool_use` 块、`BetaContentBlockParam`）。支持其他 LLM API 需要转换层，可能会丢失语义保真度。

7. **功能标志的脆弱性**：88 个功能标志中有 34 个在当前快照中无法编译。死代码消除方法是脆弱的——添加功能标志需要理解完整的导入图，以避免破坏条件性 require。

8. **没有确定性回放**：虽然 `withStreamingVCR` 提供回放缓存，但它不是为跨代码更改的确定性回放而设计的。没有办法重新运行对话并获得相同的结果（因为 LLM 是非确定性的）。

---

## 总结表

| 特性 | free-code（Claude Code 分支） |
|---------------|------------------------------|
| **语言** | TypeScript（严格模式）+ TSX |
| **运行时** | Bun >=1.3.11 |
| **UI** | Ink（面向终端的 React） |
| **源代码行数** | 约 1,934 个文件 |
| **模式** | 单体应用程序 |
| **核心循环** | `QueryEngine`（异步生成器，每个对话一个） |
| **工具数量** | 约 50 个（核心：30 个，条件性：约 20 个） |
| **LLM 提供商** | Anthropic、OpenAI Codex、AWS Bedrock、Google Vertex、Foundry |
| **默认上下文窗口** | 200K token（[1m] 型号为 1M） |
| **压缩策略** | 微压缩（每个结果）+ 完全压缩（基于摘要）+ 裁剪（历史裁剪） |
| **状态管理** | 类 Redux 的可观察存储（AppState）+ 每个引擎的可变消息 |
| **权限模型** | 集成的允许/拒绝/询问规则，带绕过和计划模式 |
| **可扩展性** | 技能（路径触发）、插件（社区）、MCP（外部工具） |
| **流式传输** | 全程异步生成器；实时 UI + 无头 SDK 共享相同路径 |
| **马尔可夫分解** | 部分的：工具提供按需状态切片；压缩压缩历史；子代理限定上下文；但压缩是有损的，上下文在没有干预的情况下无限增长 |
| **Bash 接口的中心性** | 关键的：通用回退、LLM 原生、可组合，是外部状态和内部上下文之间的主要桥梁 |
| **关键优势** | 全面的工具集、健壮的压缩、多提供商、流式架构、功能标志 |
| **关键劣势** | 单体耦合、有损压缩、没有形式化任务验证、Bash 无界性、功能标志脆弱性 |

---

## 结论

free-code（Claude Code）代表了基于终端的 AI 编程代理的最先进技术。其架构围绕一个强大的设计洞察：**LLM 不需要在上下文中持有整个项目——它只需要工具来按需查询项目状态的特定切片，以及一个压缩系统将历史压缩为可管理的摘要**。

马尔可夫分解命题**在实践中得到验证，但在实现上不完美**。代理成功地将"理解和修改大型代码库"的非马尔可夫问题分解为由上下文窗口限定的一系列马尔可夫状态，但分解是有损的（压缩摘要可能会遗漏细节），并且需要持续的主动管理（自动压缩触发、token 预算监控）。

类 Bash 接口确实是设计的核心——它提供了一个通用的、LLM 原生的桥梁，在确定性的外部状态（文件系统、git、进程）和概率性的内部状态（LLM 的上下文窗口）之间。每个其他工具都是对 Bash 的优化；Bash 是不可化约的核心。

未来的改进可能会集中在使压缩更少损失（更好的摘要），添加形式化任务验证（确认代理确实做了它说会做的事情），以及减少单体耦合以支持替代架构的实验。

---

*报告基于 free-code 仓库（版本 2.1.87）的直接源代码分析生成。所有架构声明均对照实际源文件验证，而非二手文档。*