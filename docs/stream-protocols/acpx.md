# acpx — Stream Protocol 与 Message 入库设计

调研对象：`codes/acpx`

## 核心定位

acpx 是 ACP runtime / CLI / replay 工具。它既消费 ACP JSON-RPC stream，也提供可嵌入的 `AcpRuntime` API，因此比 SDK 多了一层运行时事件和 session store。

## Stream Protocol

acpx 底层使用 ACP NDJSON / JSON-RPC：

- 子进程 stdio 转 `ReadableStream<AnyMessage>`。
- agent 通过 `session/update` 发出 `agent_message_chunk`、`tool_call_update` 等通知。
- runtime 层把通知转成 `AcpRuntimeEvent`。

`AcpRuntime.runTurn()` 返回 `AsyncIterable<AcpRuntimeEvent>`：

| event type | 说明 |
|---|---|
| `text_delta` | 文本或 thought 增量 |
| `status` | 状态、token、context window |
| `tool_call` | 工具调用及状态 |
| `done` | turn 完成 |
| `error` | 错误 |

这让嵌入方可以用 `for await` 消费流，不必直接处理 JSON-RPC。

## Message 入库保存协议

acpx 有明确 session 模型：

- `SessionRecord`：包含 schema、record id、ACP session id、agent session id、cwd、eventLog、messages、closed 等字段。
- `SessionMessage`：`User`、`Agent` 或 `Resume`。
- `SessionAgentMessage`：包含 `Text`、`Thinking`、`RedactedThinking`、`ToolUse` 和 `tool_results`。
- event log：active + rotated segments，保存 JSON-RPC message / session event，用于 replay 和恢复。

`recordSessionUpdate()` 会把 `session/update` reducer 到 conversation：

- `user_message_chunk` 生成 `User` message。
- `agent_message_chunk` 追加 agent text。
- `agent_thought_chunk` 追加 thinking。
- `tool_call` / `tool_call_update` 更新 agent tool use。
- `usage_update` 更新累计 token usage。
- `session_info_update` 更新 title / updatedAt。

## 设计评价

acpx 的设计在 SDK 和产品之间：stream 仍是 ACP 原生事件，但入库层已经有可恢复 conversation model。它很适合作为“协议适配器 + 轻量持久化 runtime”的参考。
