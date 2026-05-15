# AgentAPI — Agent 生命周期

## 三态会话状态

```
Initializing → Changing ⇄ Stable
```

- **Initializing**: 环形缓冲区未满，正在初始化
- **Changing**: 屏幕正在变化 / 有出站消息 / 最后一条是用户消息
- **Stable**: 屏幕稳定且无待发送消息，agent 空闲等待输入

## Server 启动流程

```
1. 解析 agent 类型（14 种内置 + custom）
2. 配置终端尺寸
3. 读取初始 prompt（stdin 或 --initial-prompt）
4. 配置状态持久化（stateFile + save/load flags）
5. 选择传输模式：
   - PTY: SetupProcess() 创建 PTY 子进程
   - ACP: SetupACP() 建立 ACP 连接
6. 创建 HTTP 服务器
7. 启动进程监控 goroutine
8. 启动 HTTP 服务器
9. 等待 graceful shutdown
10. 关闭前 SaveState("shutdown")
11. 5s 超时停止 HTTP 服务器
```

## PTYConversation 双协程架构

### Snapshot Loop（定时器驱动）

```
每隔 SnapshotInterval:
  1. ReadScreen() — 读取 PTY 屏幕内容
  2. RingBuffer 存入快照
  3. updateLastAgentMessageLocked() — 计算增量
     a. screenDiff(oldScreen, newScreen) — 找新增行
     b. FormatMessage() — 移除用户回显、输入框
     c. FormatToolCall() — 提取工具调用
     d. 更新/创建 ConversationMessage
  4. ReadyForInitialPrompt() — Agent 就绪检测
  5. loadStateLocked() — 恢复持久化状态
  6. 发送初始 prompt（如有）
  7. 屏幕稳定 + 有出站消息 → stableSignal 通知 Send Loop
  8. Emitter 发布消息/状态/屏幕事件
```

**关键过滤规则**：
- `writingMessage = true` 时跳过更新（防止捕获终端回显）
- 最后一条是用户消息 → 创建新消息
- 最后一条是 agent 消息 → **原地覆盖更新**（实现流式累积）

### Send Loop（goroutine）

```
监听 stableSignal 和 ctx.Done():
  1. 从 outboundQueue 取消息
  2. sendMessage():
     a. 读取发送前屏幕快照
     b. writingMessage = true
     c. writeStabilize() 两阶段写入
     d. 写入用户消息到 messages 列表
     e. writingMessage = false
  3. sendingMessage = false
```

### writeStabilize() 两阶段算法

```
Phase 1 (Echo Detection, 2s timeout):
  逐 part 写入 AgentIO → 等待屏幕变化并稳定
  不超时则继续（TUI agent 可能不回显）

Phase 2 (Processing Detection, 15s timeout):
  写入 \r 回车符 → 等待屏幕变化
  每 3s 重试回车 → 超时返回 fatal 错误
```

### Send() 流程

```
1. 验证消息（不能为空、不能前后空白）
2. 检查 status == Stable，否则 ErrMessageValidationChanging
3. 创建 errCh → 放入 outboundQueue
4. 阻塞等待 <-errCh
```

## 状态持久化

### SaveState()

```
conversation 编码为 JSON → tmp 文件原子写入 → rename
```

### AgentState 结构

```go
type AgentState struct {
    Version           int
    Messages          []ConversationMessage
    InitialPrompt     string
    InitialPromptSent bool
}
```

## PTY vs ACP 模式差异

| 维度 | PTY | ACP |
|------|-----|-----|
| 通信方式 | 屏幕轮询 + diff | NDJSON over stdio |
| 消息交互 | 写文本 + 回车，等屏幕稳定 | session/prompt RPC |
| 状态检测 | RingBuffer 快照稳定检测 | session/update 通知 |
| 超时 | Echo 2s, Process 15s | ACP 客户端实现 |
| 状态持久化 | 支持 JSON atom write | 不支持 |
