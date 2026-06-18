# AgentAPI — Stream Protocol 与 Message 入库设计

调研对象：`codes/agentapi`

## 核心定位

AgentAPI 为 Claude Code、Goose、Aider 等 CLI Agent 提供 HTTP API。它把终端/ACP 会话转成浏览器可消费的 REST + SSE。

## Stream Protocol

主流入口是 `GET /events`：

- 响应类型：`text/event-stream`
- 初始阶段：发送重建当前 conversation 和 status 所需的事件。
- 后续阶段：只发送自上次 event 后发生的更新。

OpenAPI 中定义的 SSE event 包括：

| event | data |
|---|---|
| `message_update` | `MessageUpdateBody` |
| `status_change` | agent 状态 |
| `screen_update` | 原始屏幕内容 |
| `agent_error` | 错误 |

`lib/httpapi/events.go` 的 `EventEmitter.EmitMessages()` 假设“只有最后一条消息会变化，或追加新消息”，因此运行中最后一条 agent message 会频繁产生 `message_update`。

## Message 入库保存协议

AgentAPI 的核心 conversation 保存在进程内：

```go
type ConversationMessage struct {
    Id      int
    Message string
    Role    ConversationRole // user | agent
    Time    time.Time
}
```

PTY 模式下 `PTYConversation` 通过终端屏幕 diff 构造消息：

- 用户输入追加为 `user` message。
- agent 输出先创建最后一条 `agent` message。
- 后续屏幕变化会原地覆盖最后一条 agent message。

这更像运行时状态，不是数据库持久化。AgentAPI 的 resume 依赖底层 agent/PTY/ACP 能力，而不是自身保存完整 transcript。

## 设计评价

AgentAPI 采用“最后消息替换式流式更新”，非常适合终端代理，因为终端屏幕本来就是可覆盖状态。它的 SSE event 简洁，前端只需要用 `message_update` 覆盖列表中的对应 message。
