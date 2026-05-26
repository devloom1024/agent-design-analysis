# openai-node — Stream Protocol 与 Message 入库设计

调研对象：`codes/openai-node`

## 核心定位

openai-node 是 OpenAI 官方 TypeScript SDK。它提供 Chat Completions、Assistants、Responses、Realtime 等 stream helper，但不负责应用层持久化。

## Stream Protocol

### ChatCompletionStream

`src/lib/ChatCompletionStream.ts` 将 streamed chat completion chunk 累积成 snapshot，并发出事件：

- `content`
- `content.delta`
- `refusal.delta`
- `tool_calls.function.arguments.delta`
- `logprobs.content.delta`
- `logprobs.refusal.delta`
- final completion

它也支持：

- `fromReadableStream(stream)`
- `toReadableStream()`
- `for await (const chunk of stream)`

### AssistantStream

`src/lib/AssistantStream.ts` 处理 Assistants API stream event：

- `thread.message.delta`
- `thread.run.step.delta`
- `textDelta`
- `toolCallDelta`
- `runStepDelta`
- completed / failed / requires_action 等状态

SDK 会累积 delta，形成 message/run step snapshot。

## Message 入库保存协议

SDK 不提供数据库。它只维护内存状态：

- chat completion snapshot
- assistant message snapshot
- run step snapshot
- tool call arguments 累积

应用如果要入库，应保存 OpenAI API 的完成态对象或自己的业务消息：

- Chat Completions：保存请求 `messages` 和最终 assistant message / tool calls。
- Responses API：保存 response item。
- Assistants：保存 thread/message/run step 或业务侧投影。

## 设计评价

openai-node 的设计重点是把 SSE/raw chunk 做成类型化 stream helper。它适合 SDK 层复用，但应用仍必须自己定义“哪些完成态对象进入数据库”。
