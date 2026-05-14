# AgentAPI 架构分析

## 项目概述

AgentAPI 是一个 Go 语言 HTTP API 服务器，通过**伪终端（PTY）仿真**控制多种 AI 编码工具的 CLI，将终端输出解析为结构化的消息流。同时支持实验性的 ACP 协议模式。

- 仓库: https://github.com/coder/agentapi
- 技术栈: Go + Next.js (Chat UI) + SQLite
- 核心理念: "控制终端，而非调用 API"

## 统一抽象层 — 双层抽象

### 第一层: IO 抽象 — `AgentIO` 接口

```go
// lib/screentracker/conversation.go
type AgentIO interface {
    Write(data []byte) (int, error)   // 向 agent 发送数据（键盘输入/ACP 请求）
    ReadScreen() string               // 读取当前终端屏幕/ACP 响应
}
```

两个实现：
- **`termexec.Process`** (`lib/termexec/termexec.go`) — 基于 PTY 伪终端
- **`ACPAgentIO`** (`x/acpio/acpio.go`) — 基于 ACP 结构化协议

### 第二层: 会话管理 — `Conversation` 接口

```go
// lib/screentracker/conversation.go
type Conversation interface {
    Messages() []ConversationMessage
    Send(...MessagePart) error
    Start(context.Context)
    Status() ConversationStatus  // stable | changing | initializing
    Text() string
    SaveState() error
}
```

两个实现：
- **`PTYConversation`** — 定时屏幕快照 + 差异算法拆分消息
- **`ACPConversation`** — 流式 chunk 回调构建消息

### 第三层: HTTP API — `Server`

持有 `Conversation` 接口，通过 SSE 推送事件流。

## 核心机制: PTY 终端输出解析

```
[25ms 定时快照]
    ↓
[稳定性检测] — RingBuffer 保存最近 N 次快照，全部一致时判定稳定
    ↓
[差异分析] — screenDiff() 计算新旧屏幕差异
    ↓
[消息后处理]
  ├── RemoveUserInput — 移除回显的用户输入
  ├── removeMessageBox — 移除 TUI 底部输入框
  ├── FormatToolCall — 清理工具调用回显
  └── trimEmptyLines — 去除空行
```

### 两阶段写入 (writeStabilize)

1. **阶段 1 (回显检测)**: 逐 part 写入，等待屏幕变化并稳定（最多 2s）
2. **阶段 2 (处理检测)**: 发送回车键 `\r`，等待屏幕变化（最多 15s）

## 支持的 Provider（12 种）

| Agent 类型 | 别名 | 自动检测 | TUI 输入框格式 |
|-----------|------|---------|---------------|
| Claude Code | `claude` | ✓ | `>` 符号框 |
| Goose | `goose` | ✓ | `>` 符号框 |
| Aider | `aider` | ✓ | `>` 符号框 |
| OpenAI Codex | `codex` | ✗ (需 `--type`) | `›` 符号 |
| Google Gemini | `gemini` | ✗ (需 `--type`) | 专用输入框 |
| GitHub Copilot | `copilot` | ✗ | 通用格式 |
| Sourcegraph Amp | `amp` | ✗ (需 `--type`) | `╭╮╰╯` 圆角框 |
| Cursor CLI | `cursor` | ✗ | 专用输入框 |
| Auggie | `auggie` | ✗ | 通用格式 |
| Amazon Q | `amazonq` | ✗ | 通用格式 |
| Opencode | `opencode` | ✗ | `╹▀▀` 底部框 |
| Custom | `custom` | 回退 | 通用格式 |

## 两种传输模式

| 模式 | 优点 | 缺点 |
|------|------|------|
| **PTY** (默认) | 通用性强，不依赖 agent 实现细节 | 需解析终端输出，有延迟 |
| **ACP** (实验性) | 结构化消息，无需解析 | 依赖 agent 支持 ACP 协议 |

## Provider 切换机制

```bash
agentapi server -- claude          # 自动检测
agentapi server --type=codex -- codex  # 显式指定
```

`AgentType` 一旦确定，贯穿整个系统：
- 注入 `PTYConversationConfig` 用于消息格式化（选择合适的输入框移除策略）
- 通过 `/status` API 返回给前端
- 在 SSE 事件流中传递

## 关键设计模式

| 模式 | 位置 | 说明 |
|------|------|------|
| 策略模式 | `msgfmt/msgfmt.go` — `FormatAgentMessage()` | 按 AgentType 选择不同的消息清理策略 |
| 适配器模式 | `AgentIO` 接口 | PTY / ACP 两种通信方式适配为统一接口 |
| 桥接模式 | `Conversation` 接口 | 桥接 HTTP API 和底层 IO |
| 观察者模式 | `EventEmitter` | SSE 事件发布-订阅 |
| 模板方法 | `PTYConversation.sendMessage()` | 定义消息发送固定流程 |

## 核心文件

| 文件 | 职责 |
|------|------|
| `lib/screentracker/conversation.go` | AgentIO + Conversation 接口定义 |
| `lib/screentracker/pty_conversation.go` | PTY 会话实现（核心解析逻辑） |
| `lib/screentracker/diff.go` | 屏幕差异算法 |
| `lib/screentracker/ringbuffer.go` | 环形缓冲区（泛型） |
| `lib/termexec/termexec.go` | PTY 进程管理 |
| `lib/msgfmt/msgfmt.go` | 按 AgentType 的消息格式化 |
| `lib/msgfmt/message_box.go` | TUI 输入框识别与剥离 |
| `lib/msgfmt/agent_readiness.go` | Agent 就绪检测 |
| `lib/httpapi/server.go` | HTTP 服务器 + 路由 |
| `lib/httpapi/models.go` | API 数据模型 |
| `lib/httpapi/events.go` | SSE EventEmitter |
| `x/acpio/acpio.go` | ACP 协议 IO 实现 |
| `x/acpio/acp_conversation.go` | ACP 会话实现 |
| `cmd/server/server.go` | CLI 入口 + flag 定义 |
