# AionUi — Stream Protocol 与 Message 入库设计

调研对象：`codes/AionUi`

## 核心定位

AionUi 是一个桌面/WebUI Agent 产品，支持 Gemini、ACP、Codex、远程 Agent 等多种 backend。它的协议设计分为两层：

- 运行时流：子进程 stdio / WebSocket / 局部 SSE 产生增量事件。
- UI/入库消息：统一落到 `TMessage` 联合类型，再写入 SQLite。

## Stream Protocol

主要流协议有三种。

1. ACP NDJSON stream

`src/process/acp/infra/NdjsonTransport.ts` 把子进程 stdin/stdout 转成 `ReadableStream<AnyMessage>` / `WritableStream<AnyMessage>`，并委托 `@agentclientprotocol/sdk` 的 `ndJsonStream`。这说明 ACP 主路径不是 HTTP SSE，而是 **newline-delimited JSON over byte stream**。

2. WebSocket stream

同一个 `NdjsonTransport` 还提供 `fromWebSocket()`，把 WebSocket `message` 拆成多行 JSON，写出时 `JSON.stringify(message) + '\n'`。语义仍是 ACP JSON message，只是 transport 从 stdio 换成 WebSocket。

3. 局部 SSE

例如 `weixinLoginRoutes.ts` 和前端 `WeixinConfigForm.tsx` 使用 `EventSource('/api/channel/weixin/login')` 做微信扫码登录进度流。这类 SSE 是功能型流，不是主聊天协议。

## 运行时事件形态

ACP 的 `session/update` 会被转换为 AionUi 自己的 `TMessage`：

| ACP update | AionUi message |
|---|---|
| `agent_message_chunk` | `text`，按 `msg_id` 追加 |
| `agent_thought_chunk` | `thinking` 或提示类消息 |
| `tool_call` / `tool_call_update` | `acp_tool_call` |
| `plan` | `plan` |
| `available_commands_update` | `available_commands` |
| Codex events | `codex_tool_call`、`codex_permission`、file diff 等 |

`TMessage` 是 UI 和数据库的共同结构，包含 `id`、`msg_id`、`conversation_id`、`type`、`content`、`position`、`status`、`createdAt` 等字段。

## Message 入库保存协议

AionUi 使用 SQLite。`SqliteConversationRepository` 负责 `createConversation`、`getMessages`、`insertMessage` 等操作，底层委托 `AionUIDatabase`。

流式文本不逐 token 写库，而是通过 `StreamingMessageBuffer` 做批量写入：

- key：`messageId/msg_id`。
- 模式：`accumulate` 追加或 `replace` 覆盖。
- flush 条件：300ms 或累计 20 个 chunk。
- 写入策略：先用 `getMessageByMsgId(conversationId, messageId, 'text')` 查找；存在则 `updateMessage`，否则 `insertMessage`。
- 入库 message 仍是完整 `TMessage`，状态通常为 `pending`，完成后由外层更新为完成态。

## 设计评价

AionUi 的关键设计是：transport 可变，消息模型稳定。ACP、Codex、Gemini、远程 Agent 的 stream event 都尽量投影成 `TMessage`，因此 UI 渲染和 SQLite 入库可以共享同一套协议。批量 flush 避免了 token 级 UPDATE 放大，是桌面应用里很实用的取舍。
