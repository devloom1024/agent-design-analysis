# Claude Code UI — Stream Protocol 与 Message 入库设计

调研对象：`codes/claudecodeui`

## 核心定位

Claude Code UI 是多 provider 的 UI 层。它的关键设计是把 Claude/Codex/Gemini/Cursor 等原始输出归一到 `NormalizedMessage`。

## Stream Protocol

项目侧不暴露 AI SDK 那种标准 data stream，而是把 provider 原始事件转换为 `NormalizedMessage.kind`：

| kind | 说明 |
|---|---|
| `stream_delta` | 流式文本增量 |
| `stream_end` | content block 结束 |
| `text` | 完整文本 |
| `thinking` | 思考内容 |
| `tool_use` | 工具调用 |
| `tool_result` | 工具结果 |
| `permission_request` | 权限请求 |
| `interactive_prompt` | 交互式输入 |
| `status` | 状态 / token budget |
| `complete` / `error` | 结束或错误 |

典型时序：

```text
session_created
  -> stream_delta*
  -> stream_end*
  -> status?
  -> complete | error
```

## Message 入库保存协议

持久化层保存归一化后的 message，而不是 provider raw chunk。`FetchHistoryResult` 返回：

- `messages: NormalizedMessage[]`
- `total`
- `hasMore`
- `offset`
- `limit`
- `tokenUsage`

`NormalizedMessage` 字段包含 `id`、`sessionId`、`timestamp`、`provider`、`kind`、`role`、`content/text/displayText`、tool 字段、permission 字段、sequence/rowid 等。

## 设计评价

Claude Code UI 的协议重点是 provider normalization。它牺牲了一部分 provider 原始细节，换来统一历史查询、统一 UI 渲染和跨 provider stream 处理。
