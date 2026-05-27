# AI 编码工具统一调度架构研究

对 8 个开源项目如何统一调用 Claude Code、Codex、Gemini、OpenCode 等 AI 编码工具的研究总结。

## 项目全景对比

| 维度 | CC GUI | Claude Code UI | acpx | AgentAPI | AionUi | LobeHub | Proma | Agent Spaces |
|------|--------|---------------|------|----------|--------|---------|-------|--------------|
| **类型** | IDE 插件 | Web UI | CLI 客户端 | HTTP API 服务器 | 桌面应用 | Web 应用 | 桌面应用 | 本地多 Agent 工作台 |
| **技术栈** | Java + Node.js | Express + React | TypeScript CLI | Go + Next.js | Electron + React | Next.js monorepo | Electron + Bun monorepo | Express + Next.js + SQLite |
| **统一方式** | Bridge 抽象类 | IProvider 五面接口 | ACP 协议 | AgentIO + Conversation | ClientFactory + ACP | LobeRuntimeAI | Provider Adapter + Claude Agent SDK | AgentRuntime + AgentRuntimeEvent |
| **Agent CLI 数** | 2 (Claude, Codex) | 4 (Claude, Codex, Cursor, Gemini) | 16 (全部通过 ACP) | 12 (PTY 终端仿真) | 17+ (ACP + 非 ACP) | 80+ (LLM Provider) | Chat 3 Provider / Agent 8 Channel | 5 Runtime (Claude, Codex, OpenAgentSDK, LangChain, Hermes) |
| **通信协议** | NDJSON over stdio | SDK / child_process | JSON-RPC 2.0 over stdio | PTY / ACP | ACP + IPC/WebSocket | HTTP (OpenAI/Anthropic SDK) | IPC + SDK child_process / fetch SSE | SDK stream -> WS/SSE |
| **消息标准化** | SDK 原生事件 | NormalizedMessage | AcpRuntimeEvent | ConversationMessage | TMessage | ChatStreamPayload | AgentEvent + ChatStreamState | MessagePart + chain/tool detail |
| **进程模式** | Daemon + Per-process | SDK / spawn | spawn (一类一进程) | PTY spawn | ACP spawn | HTTP 请求 | SDK child_process / fetch | SDK runtime + per-agent config dir |
| **核心设计模式** | 模板方法 + 策略 | 抽象工厂 + 适配器 | 注册表 + 适配器 | 策略 + 桥接 | 工厂 + 状态机 | 工厂 + Router | 适配器 + 事件总线 + 原子状态 | Runtime 工厂 + 事件归一化 + 混合存储 |

## 统一的五种架构范式

### 范式一：私有协议适配层（CC GUI、Claude Code UI）

在应用内部定义一套抽象接口，每个 AI 工具实现该接口。切换 provider 即切换接口实现。

- **优点**: 对上层完全透明，细粒度控制
- **缺点**: 每新增一个工具需完整实现接口
- **代表接口**: `BaseSDKBridge`、`IProvider`

### 范式二：标准化协议（acpx、AionUi）

采用 Agent Client Protocol (ACP) — 基于 JSON-RPC 2.0 over stdio 的行业标准协议。任何支持 ACP 的 CLI 工具可即插即用。

- **优点**: 协议级互操作，工具方只需实现 ACP 即可接入
- **缺点**: 依赖工具方支持 ACP 协议
- **代表实现**: acpx 的 `AcpClient`、AionUi 的 `AcpConnection`

### 范式三：终端仿真（AgentAPI）

通过 PTY 伪终端控制 CLI 工具，用屏幕差异算法将终端输出解析为结构化消息。

- **优点**: 无需工具方任何适配，通用性最强
- **缺点**: 终端解析复杂，有延迟，脆弱
- **代表实现**: `PTYConversation.screenDiff()`

### 范式四：API 协议工厂（LobeHub）

所有 Provider 通过工厂模式生成，统一使用 OpenAI 或 Anthropic 兼容的 HTTP API 协议，不支持 ACP/CLI 模式。

- **优点**: 80+ Provider，规模最大
- **缺点**: 仅面向 LLM API，不直接控制 CLI 编码工具
- **代表实现**: `openaiCompatibleFactory`、`RouterRuntime`

### 范式五：双模式混合（Proma）

同一应用内 Chat 和 Agent 两套模式独立运作。Chat 使用 Provider Adapter + fetch SSE；Agent 使用 Claude Agent SDK + child_process。

- **优点**: 轻量 Chat 和强大 Agent 模式共存，各取所长
- **缺点**: 两套消息体系增加维护成本，Agent 模式依赖 SDK 二进制 (~200MB)
- **代表实现**: `ClaudeAgentAdapter`、`sse-reader.ts`

## 双端专题

- [Web + Desktop 双端架构分析](./web-desktop-dual-end-analysis.md) — AionUi 与 LobeHub 的双端实现对照

## 关键设计决策对比

| 决策点 | 选择 A (SDK 内嵌) | 选择 B (子进程) | 选择 C (PTY 仿真) | 选择 D (HTTP API) |
|--------|-----------------|----------------|-----------------|-----------------|
| Claude Code UI | ✓ Claude/Codex | ✓ Cursor/Gemini | — | — |
| CC GUI | ✓ Claude/Codex (SDK) | 回退模式 | — | — |
| acpx | — | ✓ 全部通过 spawn | — | — |
| AgentAPI | — | — | ✓ 默认模式 | — |
| AionUi | — | ✓ 全部通过 spawn | — | — |
| LobeHub | — | — | — | ✓ HTTP API |
| Proma | ✓ Agent (SDK) | — | — | ✓ Chat (fetch SSE) |

## 消息标准化策略

所有项目都实现了某种形式的"消息归一化"：

```
原始消息（SDK事件 / CLI输出 / ACP更新 / PTY屏幕）
    ↓  适配器/转换器
统一消息格式（NormalizedMessage / TMessage / ConversationMessage）
    ↓
前端渲染
```

| 项目 | 统一消息类型 | 归一化方式 |
|------|-------------|----------|
| CC GUI | SDK 原生流事件 | ClaudeStreamAdapter / SDK 直接回调 |
| Claude Code UI | `NormalizedMessage` | sessionsService.normalizeMessage() → 策略分发 |
| acpx | `AcpRuntimeEvent` | ACP 协议本身就定义了标准化事件 |
| AgentAPI | `ConversationMessage` | PTY 屏幕差异 → FormatMessage() |
| AionUi | `TMessage` | AcpAdapter 转换 ACP SessionUpdate |
| LobeHub | OpenAI Chat Stream | 各 Provider 适配器统一到 OpenAI 流格式 |
| Proma | `AgentEvent` / `ChatStreamState` | convertSDKMessage() / adapter.parseSSELine() |
| Agent Spaces | `AgentRuntimeEvent` / `MessagePart` | Claude/Codex SDK stream -> runtime event -> chain/text/context parts |

## 各项目文档

| 项目 | 文档 |
|------|------|
| CC GUI (JetBrains 插件) | [jetbrains-cc-gui.md](./jetbrains-cc-gui.md) |
| Claude Code UI (CloudCLI) | [claudecodeui.md](./claudecodeui.md) |
| acpx (ACP CLI 客户端) | [acpx.md](./acpx.md) |
| AgentAPI (PTY HTTP 服务器) | [agentapi.md](./agentapi.md) |
| AionUi (桌面应用) | [AionUi.md](./AionUi.md) |
| LobeHub (Web 应用) | [lobehub.md](./lobehub.md) |
| Proma (桌面工作台) | [proma.md](./proma.md) |
| Agent Spaces (本地多 Agent 工作台) | [agent-spaces.md](./agent-spaces.md) |
