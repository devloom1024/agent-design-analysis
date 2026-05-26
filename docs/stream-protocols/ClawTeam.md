# ClawTeam — Stream Protocol 与 Message 入库设计

调研对象：`codes/ClawTeam`

## 核心定位

ClawTeam 是多 Agent 团队协作工具。它不直接实现 LLM token stream，而是围绕 team、task、inbox、board、session 这些协作对象建立协议。

## Stream Protocol

ClawTeam 的实时流主要是 Web board 的 SSE：

- 入口：`clawteam board serve`
- HTTP 路由：`/api/events/{team}`
- 传输：`text/event-stream`
- payload：完整 team snapshot JSON
- 前端：`new EventSource('/api/events/' + teamName)`

`clawteam/board/server.py` 中 `_serve_sse()` 循环读取 `BoardCollector.collect_team(team)`，并发送：

```text
data: { ...team snapshot... }

```

这不是 token delta，而是 **定时快照流**。它适合看板，因为看板关心最终 task/member/message 状态，而不是每个字段的 patch。

此外，ClawTeam 还有文件轮询 inbox watch、Redis wakeup publish、tmux/subprocess agent 注入等机制。这些更像协作事件通知，不是前端 chat stream。

## Message 入库保存协议

ClawTeam 使用文件系统作为主存储：

- team/task 数据由 `clawteam/store/file.py` 等文件 store 管理。
- inbox 是每个 agent 的文件队列；`receive` 是破坏性读取，`peek` 是非破坏性读取。
- event log 以 `events/evt-*.json` 等文件保存，用于 board message history 和 gource 可视化。
- session resume 信息由 `clawteam/spawn/sessions.py` 管理，保存 native client session id、cwd、client 等状态。

消息模型核心是 `TeamMessage`：

- `type`: 默认 `message`，也可表示其他事件。
- `from_agent` / `to` / `content` / timestamp 等字段。
- inbox 文件承载未读消息，event log 承载历史消息和 task 事件。

## 设计评价

ClawTeam 的协议不是“LLM 输出流”，而是“团队状态流”。SSE 推完整快照，文件队列保存协作消息，session 文件只保存 resume 需要的 native client id。这种设计简洁、可调试，不需要数据库即可支撑多 agent 协作。
