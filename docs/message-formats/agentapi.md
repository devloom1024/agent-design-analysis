# AgentAPI — 消息格式

## 概述

AgentAPI 的 Go 语言类型系统确保编译时安全。消息分为 HTTP API 层（`Message`）和内部会话层（`ConversationMessage`）。

## HTTP API 层

### Message（API 响应）

```go
type Message struct {
    Id      int                 `json:"id"`       // 唯一顺序 ID
    Content string              `json:"content"`  // 消息内容，格式化为 80 字符/行
    Role    ConversationRole    `json:"role"`     // "user" | "agent"
    Time    time.Time           `json:"time"`     // ISO 时间戳
}
```

### MessageRequestBody（API 请求）

```go
type MessageRequestBody struct {
    Content string      `json:"content"`
    Type    MessageType `json:"type"`    // "user" | "raw"
}
```

### MessageType

```go
type MessageType string
const (
    MessageTypeUser MessageType = "user"  // 记录到历史，等待 agent 开始处理后响应
    MessageTypeRaw  MessageType = "raw"   // 直接写入 PTY 作为击键，不保存到历史
)
```

## 内部会话层

### ConversationMessage

```go
type ConversationMessage struct {
    Id      int              `json:"id"`
    Message string           `json:"message"`     // 消息文本
    Role    ConversationRole `json:"role"`         // "user" | "agent"
    Time    time.Time        `json:"time"`
}
```

### ConversationRole

```go
type ConversationRole string
const (
    ConversationRoleUser  ConversationRole = "user"
    ConversationRoleAgent ConversationRole = "agent"
)
```

### ConversationStatus

```go
type ConversationStatus string
const (
    ConversationStatusChanging     ConversationStatus = "changing"      // Agent 正在输出
    ConversationStatusStable       ConversationStatus = "stable"        // Agent 空闲
    ConversationStatusInitializing ConversationStatus = "initializing"   // 正在初始化
)
```

### MessagePartText（发送组成）

```go
type MessagePartText struct {
    Content string  // 文本内容
    Alias   string  // 替代显示文本
    Hidden  bool    // 是否隐藏（bracketed paste 转义序列等）
}
```

## Emitter 事件接口

```go
type Emitter interface {
    EmitMessages([]ConversationMessage)                   // 消息列表更新
    EmitStatus(ConversationStatus)                        // 状态更新
    EmitScreen(string)                                    // 原始屏幕内容（仅 /internal/screen）
    EmitError(message string, level ErrorLevel)           // 错误（warning / error）
}
```

## 流式处理（"替换式"策略）

PTYConversation 通过 `updateLastAgentMessageLocked` 实现独特的**替换式**流式策略：

1. 定时快照（25ms 间隔）读取终端屏幕
2. `screenDiff(oldScreen, newScreen)` 计算增量
3. 如果最后一条是 agent 消息 → **原地覆盖更新**最后一条的内容
4. 如果最后一条是用户消息 → 创建新的 `ConversationMessage` 追加

这与其他项目"每个 chunk 产生一条新消息"的策略完全不同。

## ACP 模式

ACP 模式下使用 `ACPAgentIO` + `ACPConversation`：
- `SessionUpdate` 回调处理结构化 ACP 通知
- `agent_message_chunk` → 文本追加到响应
- `ToolCall` → 格式化为 `[Tool: <kind>] <title>`
- `ToolCallUpdate` → 格式化为 `[Tool Status: <status>]`
