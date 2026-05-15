# Agent 会话与进程生命周期分析

对 6 个项目的 Agent 会话创建、进程管理、消息交互和关闭流程的详细对比研究。

## 生命周期全景对比

| 维度 | CC GUI | Claude Code UI | acpx | AgentAPI | AionUi | LobeHub |
|------|--------|---------------|------|----------|--------|---------|
| **会话状态枚举** | 隐式(AtomicBoolean) | active/aborted | SessionRecord.closed | Initializing/Changing/Stable | 7 态 FSM | 6 态 FSM |
| **进程模式** | Daemon(长期) / Per-Process | Per-Request SDK/子进程 | Per-Session 子进程 | PTY 子进程 / ACP | Per-Session 子进程 | 无进程管理 |
| **重连机制** | 自动重启(3次) | WebSocket Writer 重连 | session/load + replay | N/A | resume + spawnAndInit | interrupt/resume |
| **心跳检测** | 15s (daemon) | 无 | 无 | 屏幕轮询 | IdleReclaimer(30s) | 无 |
| **空闲回收** | 无 | 无 | 持久化 client 缓存 | 无 | 5 分钟空闲自动 suspend | 无 |

## 会话状态机对比

### CC GUI — 隐式状态
```
isRunning / sdkPreloaded / activeRequestCount → 无显式状态枚举
```

### Claude Code UI — 二态
```
active → aborted (或移除)
```

### acpx — 会话记录驱动
```
SessionRecord.closed: boolean — 软关闭标记
```

### AgentAPI — 三态
```
Initializing → Changing ⇄ Stable
```

### AionUi — 七态形式化 FSM
```
idle → starting → active → prompting → active → suspended → resuming → active → ...
  ↓                ↓                           ↓
error ←────────────┴───────────────────────────┘
```

### LobeHub — 六态
```
idle → running → waiting_for_human → done
                  ↓                   ↑
              interrupted ←───────────┘
```

## 进程启动/关闭策略

| 项目 | 启动超时 | 关闭策略 | 优雅关闭 |
|------|---------|---------|---------|
| CC GUI (Daemon) | 30s | stdin.close → destroyForcibly | 发送 shutdown → 3s 等待 |
| CC GUI (Per-Process) | 无显式 | 进程自然退出 | 无 |
| Claude Code UI | SDK 默认 | instance.interrupt() | 无 |
| acpx | 60s initialize | stdin.end → SIGTERM → SIGKILL | 三阶段: stdin → SIGTERM → SIGKILL |
| AgentAPI | 无显式 | 进程自然退出 | 无 |
| AionUi | 60s initialize | stdin.end → SIGTERM → SIGKILL | 三阶段 |
| LobeHub | 无(非进程) | status = 'done' | 无 |

## 各项目详情

| 项目 | 文档 |
|------|------|
| CC GUI (JetBrains 插件) | [jetbrains-cc-gui.md](./jetbrains-cc-gui.md) |
| Claude Code UI | [claudecodeui.md](./claudecodeui.md) |
| acpx | [acpx.md](./acpx.md) |
| AgentAPI | [agentapi.md](./agentapi.md) |
| AionUi | [AionUi.md](./AionUi.md) |
| LobeHub | [lobehub.md](./lobehub.md) |
