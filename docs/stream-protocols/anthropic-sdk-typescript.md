# anthropic-sdk-typescript — Stream Protocol 与 Message 入库设计

调研对象：`codes/anthropic-sdk-typescript`

## 核心定位

这是 Anthropic 官方 TypeScript SDK。它负责把 Anthropic Messages API 的 SSE/raw stream event 封装成可迭代 stream 和高级事件回调，不负责应用层 message 入库。

## Stream Protocol

核心类是 `src/lib/MessageStream.ts` 和 beta 版本 `BetaMessageStream.ts`。

它们消费 Anthropic raw event：

- `message_start`
- `content_block_start`
- `content_block_delta`
- `content_block_stop`
- `message_delta`
- `message_stop`

`content_block_delta.delta.type` 进一步分为：

- `text_delta`
- `citations_delta`
- `input_json_delta`
- `thinking_delta`
- `signature_delta`
- beta 中还有 `compaction_delta`

SDK 会把这些 raw event 累积到 message snapshot，并发出高级回调：

- `text`
- `citation`
- `inputJson`
- `thinking`
- `message`
- `contentBlock`
- `finalMessage`

也支持 `fromReadableStream()` / `toReadableStream()`，方便后端代理后给前端继续消费。

## Message 入库保存协议

SDK 不提供数据库持久化。它只在内存中维护：

- 当前 message snapshot。
- content block 的累计文本 / thinking / tool input JSON。
- final message。

调用方如果要入库，通常保存 Anthropic `Message` 完成态，或保存应用自己的 `UIMessage` / conversation message。token 级 `text_delta` 不应直接作为 message 行保存，除非要做审计级 replay。

## 设计评价

Anthropic SDK 的设计重点是“raw event 到 snapshot 的 reducer”。它让调用方既能监听实时 delta，又能在最终拿到完整 `Message`，这个分层和多数 Agent 产品的 stream/message 分层一致。
