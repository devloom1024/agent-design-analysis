# claude-code-sourcemap — Stream Protocol 与 Message 入库设计

调研对象：`codes/claude-code-sourcemap`

## 核心定位

claude-code-sourcemap 是对 Claude Code 包的 sourcemap/源码复原项目。它不是全新的 Agent 产品，但提供了理解 Claude Code stream event 和 transcript 设计的重要材料。

## Stream Protocol

复原代码中存在 `SDKMessage` 到本地 REPL message 的桥接：

- `remote/sdkMessageAdapter.ts` 接收 remote backend 通过 WebSocket 发送的 SDK-format messages。
- `stream_event` 会转成 REPL `StreamEvent`。
- `assistant` / `user` / `result` / `system` 等 SDK message 会转成本地 `Message`。

典型 stream event 是 Anthropic/Claude 风格：

- `content_block_delta`
- `content_block_stop`
- assistant text / thinking
- tool use / tool result
- result / error

Remote session 使用 WebSocket 订阅：

```text
/v1/sessions/ws/{sessionId}/subscribe
```

并通过 HTTP POST 发送用户消息。

## Message 入库保存协议

Claude Code 的核心思路是 transcript message：

- 完整 user / assistant message 才进入 transcript。
- streaming raw event 用于 UI 过程态。
- message 之间通过 `uuid` / `parentUuid` 形成链。
- compact/resume 依赖 transcript 和 compact metadata。

复原包中还有 tool result 大内容持久化字段，例如 `persistedOutputPath` / `persistedOutputSize`，说明工具输出可外置到文件，message 中只保存引用和摘要。

## 设计评价

Claude Code 的设计也体现出“stream event 与 transcript 分离”：UI 可以实时显示 raw stream，持久化则保存完整 message 链。对自研 Agent 来说，`parentUuid` 链式 transcript 和大工具输出外置保存都很值得借鉴。
