# LobeHub — Stream Protocol 与 Message 入库设计

调研对象：`codes/lobehub`

## 核心定位

LobeHub 是完整 Chat/Agent 产品，协议层覆盖普通 LLM stream、tool streaming、heterogeneous agents、operation tracing 和数据库持久化。

## Stream Protocol

### ChatStreamCallbacks

`packages/model-runtime` 使用 callback 风格承接 provider stream：

- `onStart`
- `onText`
- `onThinking`
- `onToolsCalling`
- `onUsage`
- `onError`
- `onFinal`
- `onContentPart`
- `onReasoningPart`
- `onBase64Image`

这些 callback 把 provider chunk 转换成 UI store 更新和 server action。

### Heterogeneous Agent Event

`packages/heterogeneous-agents` 为 Claude Code、Codex 等 CLI agent 提供 adapter，将 raw stream 转成 `HeterogeneousAgentEvent[]`。例如 Codex adapter 处理 `turn.started`、`item.started`、`item.completed`，Claude Code adapter 处理 assistant/user/result/system/stream_event。

### Operation Trace

`OperationTraceRecorder` 保存 agent operation 的 step snapshot。它会过滤噪声事件，例如测试中明确要求去掉 `llm_stream`，避免 trace 被 token delta 膨胀。

## Message 入库保存协议

LobeHub 使用 Drizzle/Postgres schema。

核心表：

- `sessions`：agent/group session，含 `id`、`slug`、`title`、`type`、`userId`、`groupId`、`clientId`、`pinned`。
- `topics` / `threads`：会话下的主题和线程。
- `messages`：聊天消息本体。
- `message_plugins`：工具调用、intervention、插件参数和结果。
- `messages_files`：message 与文件关联。

`messages` 表字段包括：

- `role`
- `content`
- `editorData`
- `summary`
- `reasoning`
- `search`
- `metadata`
- `model`
- `provider`
- `error`
- `tools`
- `traceId`
- `observationId`
- `sessionId/topicId/threadId/parentId/agentId/groupId`

## 设计评价

LobeHub 采用“产品级数据库模型”：message 不只是文本，还关联 session/topic/thread、工具、插件、文件、reasoning、trace。它适合多 agent、多任务、长期记忆和审计场景，但 schema 成本也明显更高。
