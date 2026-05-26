# Stream Protocols 与 Message 入库设计总览

本目录从 `codes/` 下的 16 个项目出发，对比它们的运行时 stream protocol 和 message 入库保存协议。`ai-sdk-stream-protocols.md` 是英文原文，保留不动；`ai-sdk-stream-protocols.zh-CN.md` 是对应中文译文。

## 总体结论

这些项目大致分成四类：

| 类型 | 项目 | Stream protocol | Message 入库协议 |
|---|---|---|---|
| Agent 产品 / UI | AionUi、DeepSeek-Reasonix、Proma、LobeHub、Claude Code UI、Codex | 明确区分 delta、tool、reasoning、done/error 等运行时事件 | 保存完整消息或完成态 item，delta 多数只用于实时 UI |
| 协议 / SDK | acp-typescript-sdk、acpx、anthropic-sdk-typescript、openai-node、agentscope-java | 提供流抽象、事件类型或 SSE/NDJSON 解析 | SDK 本身通常不入库，交给调用方；acpx 和 AgentScope 提供文件/Session 层 |
| 代理 / 桥接工具 | AgentAPI、CC GUI、ClawTeam | SSE、回调、PTY/看板快照等轻量协议 | 内存消息、文件队列、session 文件或下游系统持久化 |
| 教程 / 逆向资料 | learn-claude-code、claude-code-sourcemap | 解释或复原 Claude Code 原始 stream 事件 | 解释 transcript / JSONL 持久化，不一定作为独立产品入库 |

一个共同设计值得借鉴：**stream event 不是 message 本体**。流式事件服务 UI 的即时反馈，入库消息要尽量保存可恢复、可回放、可继续发给模型的完成态事实。

## 文档索引

| 项目 | Stream 传输 | 运行时事件粒度 | 入库/保存方式 | 文档 |
|---|---|---|---|---|
| AI SDK | HTTP text stream / SSE data stream | `text-delta`、tool input/output、reasoning、source、finish | 示例不规定入库，`UIMessage` 由应用保存 | [中文译文](./ai-sdk-stream-protocols.zh-CN.md) / [英文原文](./ai-sdk-stream-protocols.md) |
| AionUi | ACP NDJSON、WebSocket stream、局部 SSE | `TMessage` + ACP `session/update` | SQLite `conversations/messages`，流式文本 300ms/20 chunk 批量写 | [AionUi.md](./AionUi.md) |
| ClawTeam | Web board SSE 快照、文件/Redis wakeup | team snapshot、task/message event | `~/.clawteam` 文件目录、inbox 文件、event 文件、session 文件 | [ClawTeam.md](./ClawTeam.md) |
| DeepSeek-Reasonix | DeepSeek SSE、dashboard SSE、desktop JSONL sidecar | `LoopEvent` + typed event kernel | Session JSONL 保存 `ChatMessage`，`.events.jsonl` 保存产品事件并跳过 token delta | [DeepSeek-Reasonix.md](./DeepSeek-Reasonix.md) |
| Proma | Provider SSE + Electron IPC | `StreamEvent`、`AgentStreamPayload` | `~/.proma/conversations.json` + `{id}.jsonl` 追加完成态消息 | [Proma.md](./Proma.md) |
| acp-typescript-sdk | Web Streams + NDJSON | ACP `session/update` union | SDK 不入库，只定义 schema 和 stream | [acp-typescript-sdk.md](./acp-typescript-sdk.md) |
| acpx | ACP NDJSON + `AsyncIterable<AcpRuntimeEvent>` | `text_delta/status/tool_call/done/error` | `SessionRecord` JSON + event segments，conversation reducer 生成 `SessionMessage` | [acpx.md](./acpx.md) |
| AgentAPI | `/events` SSE | `message_update/status_change/screen_update/agent_error` | 进程内 `ConversationMessage`，最后 agent 消息可被覆盖更新 | [agentapi.md](./agentapi.md) |
| agentscope-java | Reactor `Flux<Event>` + WebFlux SSE | `REASONING/TOOL_RESULT/HINT/SUMMARY/AGENT_RESULT` | `Session` 接口；`JsonSession` 以 State/列表保存组件状态和 `Msg` | [agentscope-java.md](./agentscope-java.md) |
| anthropic-sdk-typescript | SDK `Stream` / `MessageStream` | Anthropic raw events + high-level callbacks | SDK 内存 snapshot，不负责应用入库 | [anthropic-sdk-typescript.md](./anthropic-sdk-typescript.md) |
| claude-code-sourcemap | Claude raw stream / SDK message bridge | `stream_event` + restored `Message` | 复原 Claude transcript 设计，`parentUuid` 串联消息 | [claude-code-sourcemap.md](./claude-code-sourcemap.md) |
| claudecodeui | WebSocket / provider stream normalization | `NormalizedMessage` 14 种 kind | SQLite/历史 API 保存归一化消息 | [claudecodeui.md](./claudecodeui.md) |
| codex | Responses stream、JSONL exec event | `EventMsg`、`TurnItem`、SDK `item.*` | rollout JSONL 保存 `ResponseItem/EventMsg/SessionMeta` | [codex.md](./codex.md) |
| jetbrains-cc-gui | Java callback / NDJSON daemon | `message_start/content_delta/message_end` | 插件侧轻量结果对象，下游负责归一化/保存 | [jetbrains-cc-gui.md](./jetbrains-cc-gui.md) |
| learn-claude-code | 教程中的 Claude Code 流说明 | 以教学方式解释 streaming/error recovery | 不作为产品入库，强调 transcript/resume 概念 | [learn-claude-code.md](./learn-claude-code.md) |
| lobehub | model-runtime callbacks、agent SSE/operation stream | `ChatStreamCallbacks` + `AgentEvent` | Drizzle/Postgres `messages/sessions/topics/message_plugins` | [lobehub.md](./lobehub.md) |
| openai-node | SDK `Stream`、Chat/Assistants/Responses stream | `content.delta`、`tool_calls.*.delta`、`thread.*` 等 | SDK 只累积 snapshot；应用自行保存 `messages` / response item | [openai-node.md](./openai-node.md) |
