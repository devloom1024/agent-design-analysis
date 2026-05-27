# AgentAPI：终端屏幕统一为 ConversationMessage

## 统一处理

AgentAPI 支持 Claude Code 与 Codex：

- `AgentTypeClaude = "claude"`
- `AgentTypeCodex = "codex"`
- README 示例：`agentapi server -- claude` 与 `agentapi server --type=codex -- codex`

它的统一层不是 provider SDK，而是 `screentracker.Conversation` 接口：

```go
type Conversation interface {
  Messages() []ConversationMessage
  Send(...MessagePart) error
  Start(context.Context)
  Status() ConversationStatus
  Text() string
  SaveState() error
}
```

PTY 模式对 Claude/Codex 都适用：读取终端屏幕，做 screen diff，移除输入框/用户 echo/tool report，再生成统一消息。实验 ACP 模式也实现同一接口，但持久化不同。

## Stream Protocol

### HTTP event 类型

```go
type EventType string
const (
  EventTypeMessageUpdate EventType = "message_update"
  EventTypeStatusChange  EventType = "status_change"
  EventTypeScreenUpdate  EventType = "screen_update"
  EventTypeError         EventType = "agent_error"
)
```

| type | payload | 说明 |
| --- | --- | --- |
| `message_update` | `MessageUpdateBody` | 某条 conversation message 新增或变化。 |
| `status_change` | `StatusChangeBody` | agent 状态变化。 |
| `screen_update` | `ScreenUpdateBody` | 原始终端屏幕文本变化。 |
| `agent_error` | `ErrorBody` | 错误/警告事件。 |

### Event payload

#### `MessageUpdateBody`

```go
type MessageUpdateBody struct {
  Id      int
  Role    ConversationRole
  Message string
  Time    time.Time
}
```

| 字段 | 说明 |
| --- | --- |
| `Id` | 消息 id，同时也是顺序号。 |
| `Role` | `user` 或 `agent`。 |
| `Message` | 按终端展示格式整理后的文本。默认仍可能包含 80 列换行。 |
| `Time` | 消息时间。 |

#### `StatusChangeBody`

```go
type StatusChangeBody struct {
  Status    AgentStatus
  AgentType msgfmt.AgentType
}
```

| 字段 | 说明 |
| --- | --- |
| `Status` | `running` 或 `stable`。 |
| `AgentType` | `claude`、`codex` 等。 |

#### `ScreenUpdateBody`

| 字段 | 说明 |
| --- | --- |
| `Screen` | 当前终端屏幕，右侧空白会 trim。 |

#### `ErrorBody`

| 字段 | 说明 |
| --- | --- |
| `Message` | 错误文本。 |
| `Level` | `warning` 或 `error`。 |
| `Time` | 错误时间。 |

### PTY stream 生成逻辑

| 阶段 | 说明 |
| --- | --- |
| snapshot loop | 周期性读取 `AgentIO.ReadScreen()`。 |
| screen diff | `screenDiff(screenBeforeLastUserMessage, screen, agentType)` 得到 agent 新输出。 |
| provider 格式化 | `FormatAgentMessage(agentType, message, userInput)`。Codex 走 `formatCodexMessage()`，Claude 走 generic。 |
| tool call 清理 | `FormatToolCall(agentType, message)`。Claude 和 Codex 都会移除 `coder_report_task` 工具噪声并记录日志。 |
| message update | 如果最后一条是 agent，则原地更新；否则创建新 agent message。 |
| status | 稳定快照达到阈值且无待发送消息时为 `stable`，否则 `changing/initializing` 映射为 HTTP `running`。 |

### ACP stream 生成逻辑

实验 ACP 模式中 `ACPAgentIO.SessionUpdate()` 只抽取文本化内容：

| ACP update | 行为 |
| --- | --- |
| `AgentMessageChunk` | 追加 chunk 到 response，并触发 `onChunk(text)`。 |
| `ToolCall` | 格式化成 `"[Tool: kind] title"` 文本 chunk。 |
| `ToolCallUpdate` | 格式化成 `"[Tool Status: status]"` 文本 chunk。 |

`ACPConversation` 在 `Send()` 时创建 user message 和空 agent placeholder，chunk 到达时持续更新最后一条 agent message。

## Message 持久化

### 运行时 message：`ConversationMessage`

```go
type ConversationMessage struct {
  Id      int              `json:"id"`
  Message string           `json:"message"`
  Role    ConversationRole `json:"role"`
  Time    time.Time        `json:"time"`
}
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `Id` | 顺序 id。PTY 中通常等于 slice index；ACP 中由 `nextID` 递增。 |
| `Message` | 纯文本消息。AgentAPI 不保存工具结构、thinking 结构或 token usage。 |
| `Role` | `user` 或 `agent`。 |
| `Time` | 消息创建/更新时间。 |

### PTY state file

只有配置 `--state-file` 时启用；默认 `load-state` 和 `save-state` 会随 `state-file` 自动开启。结构：

```go
type AgentState struct {
  Version           int                   `json:"version"`
  Messages          []ConversationMessage `json:"messages"`
  InitialPrompt     string                `json:"initial_prompt"`
  InitialPromptSent bool                  `json:"initial_prompt_sent"`
}
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `Version` | state schema 版本，当前只接受 `1`。 |
| `Messages` | conversation message 数组。 |
| `InitialPrompt` | 初始 prompt 的可见字符串。 |
| `InitialPromptSent` | 初始 prompt 是否已发送；恢复时避免重复发送。 |

持久化实现：

- `SaveState()` 只在 dirty 时写。
- 先写 `stateFile.tmp`，`fsync` 后 rename 到目标文件。
- 权限：目录 `0700`，文件 `0600`。
- `loadStateLocked()` 校验版本并恢复 `messages/initialPrompt`。

### ACP 模式持久化

`ACPConversation.SaveState()` 返回错误：

```go
return xerrors.Errorf("ACP mode doesn't support state persistence")
```

因此 ACP 模式当前没有 message 持久化。

### 设计评价

AgentAPI 的 schema 最简单，优点是几乎能包任何 TUI agent；Claude 和 Codex 差异主要集中在输入框清理与 message formatting。缺点同样明显：stream protocol 只有文本和状态，tool/reasoning/usage 都被压成普通字符串或完全丢失，不适合做精细 UI 和审计。
