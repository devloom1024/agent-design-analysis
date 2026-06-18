# DeepSeek-Reasonix — Stream Protocol 与 Message 入库设计

调研对象：`codes/DeepSeek-Reasonix`

## 核心定位

DeepSeek-Reasonix 是围绕 DeepSeek cache-first loop 设计的 coding agent。它明确把三类对象拆开：

- Provider stream：DeepSeek/OpenAI-compatible SSE。
- Runtime event：给 TUI、Desktop、Dashboard 的实时事件。
- Durable message：用于 resume、cache、上下文重建的 `ChatMessage`。

## Stream Protocol

### Provider SSE

`src/client.ts` 使用 `eventsource-parser` 解析上游 SSE。`StreamChunk` 把 provider chunk 归一为：

- `contentDelta`
- `reasoningDelta`
- `toolCallDelta`
- `usage`
- `finishReason`
- `raw`

SSE 中的 `[DONE]` 作为结束标记。

### LoopEvent

`src/loop/types.ts` 定义 `LoopEvent`，它是核心运行时事件：

| role | 含义 |
|---|---|
| `assistant_delta` | 文本增量 |
| `assistant_final` | 完整 assistant 输出 |
| `tool_call_delta` | 工具参数增量 / 活跃信号 |
| `tool_start` | 工具开始执行 |
| `tool` | 工具完成 |
| `status` | 静默阶段状态 |
| `done` | turn 完成 |
| `error` / `warning` | 错误和警告 |
| `steer` | turn 中注入用户引导 |

### Dashboard / Desktop transport

Web dashboard 使用 `/api/events` SSE，把 loop 事件转换成 dashboard event。Desktop 使用 Tauri sidecar：Node stdout 输出 JSON line，Rust 读到后用 `rpc:event` 发给 WebView。

## Message 入库保存协议

Reasonix 的持久化是分层的：

1. `ChatMessage`

`src/types.ts` 中的 `ChatMessage` 是模型上下文和 session JSONL 的本体，字段包括 `role`、`content`、`tool_calls`、`tool_call_id`、`reasoning_content` 等。它服务 resume 和下一轮请求。

2. Session JSONL

`src/memory/session.ts` 负责 session 保存。长会话以 JSONL 保存 append-only chat history，并配合 compact/recovery 做恢复。

3. Event sidecar

`src/adapters/event-sink-jsonl.ts` 和 `src/core/events.ts` 保存产品化 typed event。关键点是跳过 `model.delta` 这类 token 级事件，避免 sidecar 被流式文本膨胀。

## 设计评价

Reasonix 的设计非常适合自研 Agent 参考：provider stream 只负责解析，runtime event 服务 UI，durable message 服务恢复和 cache。尤其是“delta 不等于入库 message”和“event sidecar 跳过 token delta”的策略，能同时保证 UI 实时性和历史可维护性。
