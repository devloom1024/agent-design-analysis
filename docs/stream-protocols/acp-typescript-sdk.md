# acp-typescript-sdk — Stream Protocol 与 Message 入库设计

调研对象：`codes/acp-typescript-sdk`

## 核心定位

这是 Agent Client Protocol 的 TypeScript SDK。它不实现具体产品 UI，也不负责 message 入库；它负责定义 ACP JSON-RPC schema 和 transport stream 抽象。

## Stream Protocol

`src/stream.ts` 定义核心 `Stream`：

```ts
type Stream = {
  writable: WritableStream<AnyMessage>;
  readable: ReadableStream<AnyMessage>;
};
```

`ndJsonStream(output, input)` 把字节流转换为 ACP message 流：

- 输入：`ReadableStream<Uint8Array>`
- 输出：`WritableStream<Uint8Array>`
- 读取时用 `TextDecoder` 按 `\n` 分割。
- 每行 `JSON.parse` 成 `AnyMessage` 后 `controller.enqueue(message)`。
- 写入时 `JSON.stringify(message) + '\n'`。

这是一种 **NDJSON over Web Streams** 协议，适合 stdio、子进程、WebSocket adapter 等场景。

## 运行时事件形态

SDK schema 中 `SessionUpdate` 是核心流式通知 union：

- `user_message_chunk`
- `agent_message_chunk`
- `agent_thought_chunk`
- `tool_call`
- `tool_call_update`
- `plan`
- `available_commands_update`
- `current_mode_update`
- `config_option_update`
- `session_info_update`
- `usage_update`

这些 update 通常包在 JSON-RPC notification `session/update` 中。

## Message 入库保存协议

SDK 本身没有数据库或文件持久化协议。它只提供：

- JSON-RPC message 类型。
- `session/update` 流事件类型。
- NDJSON stream 编解码。

调用方需要自行决定如何把 `agent_message_chunk` 合并为完整 assistant message、如何保存 tool call 和 usage。AionUi、acpx 等项目就是 SDK 消费者示例。

## 设计评价

acp-typescript-sdk 的核心价值是协议纯度：transport 是 `ReadableStream/WritableStream`，语义是 JSON-RPC + `session/update`。它避免绑定 HTTP/SSE 或具体数据库，让上层产品能自由选择入库模型。
