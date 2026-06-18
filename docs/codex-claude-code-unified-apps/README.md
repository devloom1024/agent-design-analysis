# 同时支持 Codex 与 Claude Code 的统一协议与消息存储分析

本目录只分析源码中明确同时支持 **OpenAI Codex / Codex CLI** 与 **Claude Code / Claude Code SDK/CLI** 的应用或运行时。重点不是列功能，而是回答三个问题：

1. 它们如何把 Codex 与 Claude Code 统一成同一套应用内抽象。
2. stream protocol 有哪些事件类型、数据结构、字段语义。
3. 持久化 message 的格式是什么，每个字段如何使用。

## 纳入与排除

| 项目 | 是否纳入 | 依据 |
| --- | --- | --- |
| Agent Spaces | 是 | `ClaudeCodeRuntime` 与 `CodexRuntime` 都实现 `AgentRuntime`，统一输出 `AgentRuntimeEvent`，消息落到 workspace channel JSON。 |
| LobeHub | 是 | `packages/heterogeneous-agents` 中有 `ClaudeCodeAdapter` 与 `CodexAdapter`，统一成 `HeterogeneousAgentEvent` / `AgentStreamEvent`，DB 中用 `messages` 与 `message_plugins` 存储。 |
| acpx | 是 | 内置 `codex` 与 `claude` ACP adapter 命令，把两者统一为 ACP JSON-RPC / `AcpRuntimeEvent`，再投影为 `SessionConversation`。 |
| Claude Code UI | 是 | `LLMProvider = 'claude' | 'codex' | ...`，`IProvider.sessions.normalizeMessage()` 把 Claude/Codex 历史和实时事件统一成 `NormalizedMessage`。 |
| JetBrains CC GUI | 是 | 同一 `ClaudeSession` 管理 Claude 与 Codex，分别经 `ClaudeMessageHandler` / `CodexMessageHandler` 转成统一 `ClaudeSession.Message` 与 webview `ClaudeMessage`。 |
| AionUi | 是 | 同一 chat schema 同时包含 `acp_*` 与 `codex_*` 消息类型，SQLite `messages` 表统一持久化。 |
| AgentAPI | 是 | `--type=claude` 与 `--type=codex` 进入同一 `screentracker.Conversation` / HTTP event API；另有实验 ACP conversation。 |
| Proma | 否 | 搜到 Claude Code 兼容 SDK 和 Codex 模型相关内容，但没有确认到同时接入 Codex CLI/runtime 与 Claude Code 的统一运行时实现。 |

## 统一抽象对比

| 项目 | 统一层 | Claude 输入 | Codex 输入 | Stream 输出 | 持久化策略 |
| --- | --- | --- | --- | --- | --- |
| Agent Spaces | `AgentRuntime` | Claude Code SDK messages | Codex JSON-RPC/CLI events | `AgentRuntimeEvent` + `MessagePart` | `messages.json` + `tool-details.json`；agent session/usage 进 SQLite |
| LobeHub | `AgentEventAdapter` | Claude Code `stream-json` NDJSON | Codex `exec --json` NDJSON | `HeterogeneousAgentEvent` -> `AgentStreamEvent` | PostgreSQL `messages` / `message_plugins` / `threads` |
| acpx | ACP client/runtime | `@agentclientprotocol/claude-agent-acp` | `@zed-industries/codex-acp` | ACP `session/update` -> `AcpRuntimeEvent` | `~/.acpx/sessions/*.json` + event log NDJSON segments |
| Claude Code UI | provider registry | Claude SDK query stream | OpenAI Codex SDK stream | WebSocket `NormalizedMessage` | SQLite `sessions` metadata；消息从 provider 原生 JSONL/session files 读取并归一化 |
| JetBrains CC GUI | `ClaudeSession` | Claude SDK bridge | Codex SDK bridge | Java callback + webview update/delta | 运行时内存 `Message`；历史从 `~/.claude` / `~/.codex` JSONL 读取，索引缓存在 `~/.codemoss/cache` |
| AionUi | chat message model | ACP-like update/tool stream | Codex JSON-RPC event | `TMessage` / `TMessageType` | SQLite `messages(content JSON)` |
| AgentAPI | `Conversation` | PTY screen or ACP | PTY screen or ACP | HTTP/SSE-like `Event` | 可选 `--state-file` JSON；ACP 模式不支持 state persistence |

## 设计模式归纳

### 1. Adapter 归一化

LobeHub、Claude Code UI、Agent Spaces 都用 provider adapter/runtime 把原生 stream 变成应用内事件。优点是 UI 和持久化层不需要理解 Claude 与 Codex 的全部差异；缺点是 adapter 必须维护 provider-specific 的细节，例如 Codex `item.completed` 的工具结果、Claude Code `stream_event.content_block_delta` 的增量去重。

### 2. 协议中介层

acpx 以 ACP 为中介：Claude 与 Codex 都先变成 ACP JSON-RPC，再被 acpx 投影到 `AcpRuntimeEvent` 和 `SessionConversation`。优点是未来接更多 agent 时成本低；缺点是如果 ACP adapter 丢失原生细节，应用层也拿不到。

### 3. 终端屏幕抽象

AgentAPI 主要把 Claude/Codex 当作 TUI 进程，通过 PTY screen diff 得到“用户消息/agent 消息”。优点是适配范围广；缺点是 stream protocol 只能表达文本屏幕变化、状态和错误，工具调用只是格式化文本而不是结构化对象。

### 4. 原生历史 + 应用索引

Claude Code UI 和 JetBrains CC GUI 并不总是复制完整消息到自己的数据库；它们通常保留 provider 原生历史文件作为 transcript source of truth，再用 SQLite 或 JSON index 存 session metadata。这样的存储更轻，但 message schema 由 Claude/Codex 原生文件决定。

## 文档索引

- [Agent Spaces](./agent-spaces.md)
- [LobeHub](./lobehub.md)
- [acpx](./acpx.md)
- [Claude Code UI](./claude-code-ui.md)
- [JetBrains CC GUI](./jetbrains-cc-gui.md)
- [AionUi](./aionui.md)
- [AgentAPI](./agentapi.md)

