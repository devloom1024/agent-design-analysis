# agentscope-java — Stream Protocol 与 Message 入库设计

调研对象：`codes/agentscope-java`

## 核心定位

AgentScope Java 是 Java Agent 框架。它使用 Reactor `Flux` 表达 agent stream，并能直接桥接到 Spring WebFlux SSE。

## Stream Protocol

`Agent.stream()` 返回：

```java
Flux<Event> events = agent.stream(messages, StreamOptions.defaults());
```

事件类型包括：

| EventType | 说明 |
|---|---|
| `REASONING` | 推理/模型输出，可多 chunk |
| `TOOL_RESULT` | 工具执行结果 |
| `HINT` | RAG / memory 注入上下文 |
| `SUMMARY` | maxIters 后的总结 |
| `AGENT_RESULT` | 最终回复，默认不在 stream 中 |

`StreamOptions` 支持：

- `incremental(true)`：chunk 只包含新增 delta。
- `incremental(false)`：chunk 包含当前完整累计文本。
- `includeReasoningChunk/includeReasoningResult`：分别控制中间 delta 和最终 reasoning。

WebFlux SSE 示例：

```java
@GetMapping(value = "/chat", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> chat(@RequestParam String message) {
  return agent.stream(...).map(event ->
    ServerSentEvent.<String>builder()
      .event(event.getType().name().toLowerCase())
      .data(event.getMessage().getTextContent())
      .build());
}
```

底层模型客户端也支持 SSE，例如 `OkHttpTransport` / `JdkHttpTransport` 解析 `data:` 行并以 `[DONE]` 结束。

## Message 入库保存协议

AgentScope 的 message 本体是 `Msg`：

- `id`
- `name`
- `role`
- `content: List<ContentBlock>`
- `metadata`
- `timestamp`

持久化通过 session 抽象完成：

- `Session.save(SessionKey, key, State)` 保存单个 state。
- `Session.save(SessionKey, key, List<? extends State>)` 保存列表。
- `JsonSession` 对列表做 incremental append。
- `SessionManager` 负责让 agent、memory、toolkit 等组件执行 `saveTo/loadFrom`。

因此 message 入库不是固定 DB schema，而是框架级 State persistence。

## 设计评价

AgentScope Java 的 stream 协议是响应式对象流，而不是自定义字符串协议。它的优点是很容易桥接 SSE/WebSocket，也能用 `StreamOptions` 在 delta 和 full-text 两种 UI 需求之间切换。
