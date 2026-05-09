# Coding Agent Framework 研究报告

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Language](https://img.shields.io/badge/lang-中文-red.svg)]()

对三个主流 AI Coding Agent 框架的深度架构分析，以及关于 **Agent Framework 本质** 的统一理论。

---

## 目录

- [研究对象](#研究对象)
- [研究报告](#研究报告)
- [基础范式：三层架构](#基础范式三层架构)
  - [通用 Agent 循环](#通用-agent-循环)
  - [工具接口：确定性桥梁](#工具接口确定性桥梁)
  - [上下文工程：核心学科](#上下文工程核心学科)
- [核心类比](#核心类比)
- [项目结构](#项目结构)
- [报告统计](#报告统计)

---

## 研究对象

| 项目 | 类型 | 运行时 | 核心特点 |
|------|------|--------|----------|
| **free-code** (Claude Code) | Anthropic 官方 Coding Agent | Bun | QueryEngine 循环, Redux 状态管理, autoCompact |
| **opencode** | 多包 Agent 框架 | Bun | Effect 函数式层, V2 架构, LSP 集成, 18个包 |
| **pi** | 模块化 Agent 框架 | Node.js/npm | 5包清晰分离, agent-loop 模式, JSONL 会话 |

---

## 研究报告

### 综合报告

- [**00-综合研究报告**](agent-research-report/00-comprehensive-research-report.md) — 核心假设的完整评估，横跨所有三个框架的全景分析

### 个体分析报告（3篇）

| # | 报告 | 说明 |
|---|------|------|
| 1 | [**free-code 全面架构分析**](agent-research-report/各项目报告/01-free-code-analysis.md) | Claude Code 如何通过类 Bash 接口将非马尔可夫项目分解为马尔可夫状态 |
| 2 | [**OpenCode Agent 框架综合分析**](agent-research-report/各项目报告/03-opencode-analysis.md) | Effect-TS 函数式架构，18包多包体系，V2 Loop |
| 3 | [**Pi Agent Harness 综合架构分析**](agent-research-report/各项目报告/04-pi-analysis.md) | 5包模块化设计，agent-loop 模式，JSONL 会话存储 |

### 两两对比报告（3篇）

| # | 对比 | 说明 |
|---|------|------|
| 4 | [**free-code vs OpenCode**](agent-research-report/对比/02-free-code-vs-opencode.md) | 两款终端原生 AI 编码智能体如何应对非马尔可夫状态分解 |
| 5 | [**free-code vs Pi**](agent-research-report/对比/03-free-code-vs-pi.md) | 通过马尔可夫分解视角比较两个终端原生 AI 编码代理 |
| 6 | [**OpenCode vs Pi**](agent-research-report/对比/06-opencode-vs-pi.md) | 跨11个架构维度的结构化比较 |

---

## 基础范式：三层架构

所有三个项目共享相同的三层架构。这不是巧合，而是**LLM 作为马尔可夫状态机的必然结果**。

```
┌──────────────────────────────────────────────────────────────────┐
│ 第一层：外部状态（非马尔可夫，实际上无限）                        │
│                                                                  │
│  文件系统  │  Git历史  │  数据库  │  API  │  进程               │
│                                                                  │
│  100K文件，数百万行代码，完整提交DAG，运行中的服务                  │
│  太大了，无法适应任何上下文窗口。深层历史依赖。                     │
│  LLM 无法直接观察或操作。                                         │
└────────────────────────────┬─────────────────────────────────────┘
                             │  工具查询特定切片 / 工具修改特定目标
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ 第二层：Agent 运行时（桥接层）                                     │
│                                                                  │
│  工具执行器 │ 会话管理器 │ 压缩引擎 │ 权限评估器                  │
│  LLM提供商  │ 上下文构造器│ 子Agent创建器│ 扩展宿主               │
│                                                                  │
│  这是阻抗匹配层。在无限的、非马尔可夫的外部世界和                   │
│  有界的、马尔可夫的 LLM 上下文窗口之间进行转换。                   │
│  每个 Agent 框架的核心都是这一层。                                │
└────────────────────────────┬─────────────────────────────────────┘
                             │  当前轮次上下文
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ 第三层：LLM 上下文窗口（马尔可夫，有界，有限）                     │
│                                                                  │
│  系统提示 + 工具定义 + 对话历史 + 当前状态 + 近期工具结果          │
│                                                                  │
│  受限于 200K-1M token。每个轮次自包含。                           │
│  LLM 仅看到这些。仅基于这些做决策。                               │
│  这是马尔可夫决策表面。                                           │
└──────────────────────────────────────────────────────────────────┘
```

**第一层（外部状态）：** 三个项目处理方式相同——文件系统、git 仓库和外部服务是基础事实。所有修改都是真实的、不可逆的、可从外部验证的。

**第二层（Agent 运行时）：**
- free-code：`QueryEngine` 类管理 Agent 生命周期，`services/compact/` 包含压缩子系统
- opencode：`packages/opencode/src` 中 43 个模块涵盖工具、会话、压缩、权限、LSP
- pi：`agent-session.ts` 和 `compaction/` 模块构成运行时核心，最小化但完整

**第三层（LLM 上下文窗口）：** 三个项目汇聚到同一模式：系统提示 + 工具定义 + 任务状态 + 压缩历史 + 环境信息 + 近期工具结果。

---

### 通用 Agent 循环

核心执行循环在三个项目中几乎完全相同。这种趋同程度表明**这是基于 LLM 的 Agent 的唯一可行架构**：

```
用户输入 → 模板/系统提示构建 → 上下文组装 → LLM调用 →
响应解析 → 工具执行 → 结果集成 → 压缩检查 → 循环
```

| 步骤 | free-code | opencode | pi |
|------|-----------|----------|-----|
| 用户输入 | 自然语言/slash命令 | 自然语言/slash命令 | 自然语言/RPC请求 |
| 提示构建 | fetchSystemPromptParts() | Config→Provider→Session | 模板 + transformContext() |
| 上下文组装 | normalizeMessagesForAPI() | 8部分上下文构建 | convertToLlm边界 |
| LLM调用 | Claude SDK流式 | Vercel AI SDK流式 | pi-ai流式 |
| 工具执行 | findToolByName + 权限检查 | 工具注册表 + Bus中介 | beforeToolCall钩子 |
| 压缩 | autoCompact + snipCompact | 消息压缩 + 会话溢出 | 自动压缩 + 分支摘要 |

---

### 工具接口：确定性桥梁

工具接口是概率性 LLM（第三层）与确定性世界（第一层）之间的**确定性桥梁**。

**为什么必须确定性：** 在 MDP 中，转换函数 T(s,a)→s' 必须是可预测的。如果相同的文件读取返回不同的内容，LLM 无法从经验中学习。每个轮次变成对不可预测世界的探索。

三个项目都强制确定性：
- free-code：`FileEditTool` 使用精确字符串匹配——如果 oldString 匹配多次则失败
- opencode：Effect-TS 类型化工具执行确保结果遵循类型化契约；tree-sitter 做确定性 AST 解析
- pi：工具通过 TypeBox 模式返回类型化结果；edit 工具使用精确字符串匹配

**Bash 作为最小公分母：** Bash 完备、确定、有界、LLM 原生、可组合。但专用工具更优——LSP 提供结构化语义信息，AST-grep 提供基于解析树的转换。原则：**bash 是通用桥梁；专用工具是优化的桥梁。**

---

### 上下文工程：核心学科

进入上下文窗口的内容决定了 LLM 能做什么。上下文窗口是系统中最稀缺、最昂贵的资源。

上下文窗口的 token 预算如何分配：

1. **系统提示**：预加载领域知识，无需 LLM 通过工具调用去发现
2. **工具定义**：更多工具=更多能力但更少的对话空间（pi 的 7 工具最小主义 vs opencode 的 17+ 工具）
3. **对话历史**：压缩将历史转换为紧凑摘要，摘要质量直接决定未来决策质量
4. **当前状态**：待办列表、关键文件、环境信息——当前轮次的马尔可夫快照
5. **保留空间**：为 LLM 响应预留空间

**上下文策展是有损压缩问题。** Agent 必须决定保留什么、摘要什么和丢弃什么。

---

## 核心类比

| OS 概念 | Agent Framework 对应 | 说明 |
|---------|---------------------|------|
| 进程 | Agent 会话 | 每个会话是一个独立的执行单元，拥有自己的上下文 |
| 内存(RAM) | LLM 上下文窗口 | 有限的、珍贵的、必须管理的资源 |
| 磁盘 | 会话存储、git 仓库 | 持久化状态，进程结束后依然存在 |
| I/O 子系统 | 工具系统(bash, LSP, MCP) | 与外部世界交互的唯一渠道 |
| 虚拟内存 | 上下文压缩 | 让 LLM 以为拥有更多上下文，实际在不停地换页 |
| 缺页中断 | 工具调用 | 当需要的内容不在上下文窗口时，通过工具从磁盘获取 |
| 调度器 | Agent 循环、子Agent 创建 | 决定何时运行什么 |
| 权限模型 | 工具权限评估器 | 控制进程可以访问哪些资源 |
| 系统调用 | bash 接口 | 用户空间(LLM)与内核空间(Agent Runtime)之间的最小稳定接口 |
| fork() | 子Agent 创建 | 复制上下文，独立执行，返回结果 |

---

## 项目结构

```
Agent调研/
├── README.md                                       ← 你在这里
├── agent-research-report/                          ← 研究报告
│   ├── 项目总览与统一模型.md                        ← 完整理论总览
│   ├── 00-comprehensive-research-report.md         ← 综合研究报告
│   ├── 各项目报告/                                  ← 个体分析
│   │   ├── 01-free-code-analysis.md
│   │   ├── 03-opencode-analysis.md
│   │   └── 04-pi-analysis.md
│   └── 对比/                                        ← 两两对比
│       ├── 02-free-code-vs-opencode.md
│       ├── 03-free-code-vs-pi.md
│       └── 06-opencode-vs-pi.md
├── free-code/                                      ← free-code 源码 (Claude Code fork, Bun)
├── opencode/                                       ← OpenCode 源码 (多包框架, Bun/Effect-TS)
└── pi/                                             ← Pi 源码 (模块化框架, Node.js/npm)
```

> **注释：** 为方便学习用途，本项目附带了 `free-code`、`opencode`、`pi` 三个项目的完整源码。所有源码版权归原作者所有，此处仅用于研究和学习目的。

---

## 报告统计

| 类别 | 数量 | 总字数 |
|------|------|--------|
| 个体报告 | 3篇 | — |
| 对比报告 | 3篇 | — |
| 综合报告 | 1篇 | — |
| 总览文档 | 1篇 | — |
| **合计** | **8篇** | **约 40,000+ 字**（全中文） |

---

## 许可

MIT License
