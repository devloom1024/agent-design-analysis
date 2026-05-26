# Codex — Stream Protocol 与 Message 入库设计

调研对象：`codes/codex`

## 核心定位

Codex 把内部 agent protocol、Responses API item、UI item、exec JSONL stream、rollout 持久化拆得很清楚。

## Stream Protocol

Rust 协议层的核心是：

- `Submission`：客户端到 agent。
- `Event`：agent 到客户端。
- `EventMsg`：运行时事件 union。

`EventMsg` 覆盖：

- turn lifecycle：`TurnStarted`、`TurnComplete`、`TurnAborted`
- item lifecycle：`ItemStarted`、`ItemCompleted`
- delta：`AgentMessageContentDelta`、`ReasoningContentDelta`、`PlanDelta`
- tool/runtime：exec、MCP、patch、web search、approval 等事件
- raw model item：`RawResponseItem`

对外 exec / SDK 层再包装为更稳定的 JSONL event：

- `thread.started`
- `turn.started`
- `item.started`
- `item.updated`
- `item.completed`
- `turn.completed`
- `turn.failed`
- `error`

## Message 入库保存协议

Codex 的持久化称为 rollout：

- 文件格式：JSONL。
- 路径：`~/.codex/sessions/YYYY/MM/DD/rollout-<timestamp>-<thread_id>.jsonl`。
- writer：后台 task 异步写入，失败保留 pending item 后续重试。

入库 item 分几类：

- `ResponseItem`：模型历史本体，和 OpenAI Responses API 对齐。
- `EventMsg`：只保存少量 replay/审计需要的事件。
- `SessionMeta` / `TurnContext` / `Compacted`：恢复、列表、compact 所需元数据。

UI 使用 `TurnItem` 作为完成态展示模型，包括 `UserMessage`、`AgentMessage`、`Reasoning`、`Plan`、`McpToolCall`、`FileChange`、`ContextCompaction` 等。delta 只用于实时 UI，不是最终持久化本体。

## 设计评价

Codex 是分层最完整的参考之一：内部事件足够丰富，SDK event 足够稳定，rollout 保存模型可恢复事实。它清楚地区分了 UI item、provider response item 和 replay event。
