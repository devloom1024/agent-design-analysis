# Proma — Stream Protocol 与 Message 入库设计

调研对象：`codes/Proma`

## 核心定位

Proma 同时有 Chat 模式和 Agent 模式。Chat 模式直接对接 Anthropic/OpenAI/Google 等 provider SSE；Agent 模式消费 Claude Agent SDK 消息，再统一转成前端事件。

## Stream Protocol

### Chat SSE

`packages/core/src/providers/sse-reader.ts` 是通用 SSE 读取器：

- 发起 `fetch` POST。
- 使用 `ReadableStream.getReader()` 和 `TextDecoder` 读取 chunk。
- 按行解析 `data:`。
- 跳过 `[DONE]`。
- 调用 provider adapter 的 `parseSSELine(jsonLine)`。
- 累积 `content`、`reasoning`、`thinkingBlocks`、`toolCalls`。

Provider adapter 负责差异化解析：

- `anthropic-adapter.ts`：`content_block_delta`、`thinking_delta`、`input_json_delta`。
- `openai-adapter.ts`：`choices[0].delta.content`、`reasoning_content`、`tool_calls`。
- `google-adapter.ts`：Gemini candidates/parts。

### Agent EventBus

`apps/electron/src/main/lib/agent-event-bus.ts` 定义 `AgentEventBus`，以 `sessionId + AgentStreamPayload` 分发事件。它支持 middleware 链，最终通过 Electron IPC 送到 renderer。

## Message 入库保存协议

Chat 对话由 `conversation-manager.ts` 管理：

- 对话索引：`~/.proma/conversations.json`
- 消息文件：`~/.proma/conversations/{id}.jsonl`
- 追加消息：`appendMessage(id, message)` 使用 `appendFileSync` 逐行写入 JSON。
- 修改历史：`saveConversationMessages(id, messages)` 全量覆写 JSONL。

流式过程中，renderer 通过 Jotai atom 展示临时 `ChatStreamState`。最终 assistant 消息在 `chat-service.ts` 的流结束后创建：

- `role: 'assistant'`
- `content: accumulatedContent`
- `reasoning`
- `toolActivities`
- `attachments`
- `model`

然后调用 `appendMessage(conversationId, assistantMsg)` 入库。

## 设计评价

Proma 的协议边界很清楚：SSE reader 只做 provider-neutral 的读取与累积，adapter 只做 provider-specific 解析，入库只保存完整 `ChatMessage`。JSONL 逐行追加使本地桌面应用很容易备份、迁移和排障。
