# UI 渲染层分析

对 7 个项目的流式显示和历史消息渲染的详细对比研究，涵盖实时流式输出和 Session 历史两种场景。

## 技术栈全景对比

| 维度 | CC GUI | Claude Code UI | acpx | AgentAPI | AionUi | LobeHub | Proma |
|------|--------|---------------|------|----------|--------|---------|-------|
| **UI 框架** | React 19 + JCEF | React 18 | 纯 CLI (Commander) | Next.js 15 + React 19 | React 19 + Electron | Next.js 16 + React 19 | React 18 + Electron 39 |
| **UI 组件库** | Ant Design 6 | Tailwind CSS 3 | — (ANSI 颜色) | Tailwind CSS 4 + Radix UI | Arco Design + UnoCSS | @lobehub/ui v5 + antd | Tailwind CSS 3 + shadcn/ui (Radix) |
| **Markdown** | `marked` | `react-markdown` | — (纯文本) | — (纯文本) | `react-markdown` | `@lobehub/ui` Markdown | `react-markdown` |
| **代码高亮** | `highlight.js` | `react-syntax-highlighter` | — | — | `react-syntax-highlighter` (hljs) | Shiki (内置) | Shiki 3.22 |
| **虚拟滚动** | 自定义 `VirtualList` | 滚动分页 (20/页) | — | — | `react-virtuoso` v4 | `virtua` v0.48 | — (全量 JSONL 读取) |
| **Session 存储** | Java 后端文件 | `useSessionStore` Map | `~/.acpx/sessions/*.json` | — (无) | SQLite (better-sqlite3) | PostgreSQL + Drizzle ORM | `~/.proma/` JSON/JSONL 文件 |
| **富文本编辑器** | — | — | — | — | — | — | TipTap 3.19 |
| **状态管理** | React state | useSessionStore | — | React state | React state + context | zustand | Jotai (27 atom 文件) |

## 流式策略对比

| 策略 | 代表项目 | 机制 |
|------|---------|------|
| **轻量自定义渲染** | CC GUI | `renderStreamingContent()` 直接操作 state，按帧增量更新，不重新解析 markdown |
| **增量 re-parse** | Claude Code UI | `useChatRealtimeHandlers` 100ms 刷新，`react-markdown` 重新解析全文 |
| **stdout.write 直写** | acpx | TextOutputFormatter 直接 `process.stdout.write()`，打字机效果 |
| **纯文本 SSE** | AgentAPI | 原生 EventSource + `whitespace-pre-wrap`，无任何解析 |
| **IPC 流 + 合并** | AionUi | `useAcpMessage` hook，`composeMessageWithIndex` O(1) 索引合并 |
| **animated prop** | LobeHub | `@lobehub/ui` Markdown 的 `animated` 属性，内置打字机效果 |
| **原子写入 + 批量渲染** | Proma | `useStore()` 直接写 Jotai atom，`unstable_batchedUpdates` 合并渲染 |

## 流式渲染核心机制

```
┌─────────────────────────────────────────────────────────────────┐
│ 实时流式输出（Streaming）                                         │
├──────────────┬──────────────┬──────────────┬────────────────────┤
│  CC GUI      │ CC UI        │  acpx        │  AgentAPI          │
│  ─────────── │  ─────────── │  ─────────── │  ───────────       │
│  逐帧更新     │  WebSocket   │  stdout      │  SSE (EventSource) │
│  state chunk  │  100ms flush │  write()     │  text/plain        │
│  ↓            │  ↓           │  ↓           │  ↓                 │
│  marked.parse │  re-parse    │  终端输出     │  whitespace-pre    │
│  + hljs       │  + Prism     │  ANSI 颜色   │  -wrap             │
├──────────────┼──────────────┼──────────────┼────────────────────┤
│  AionUi       │  LobeHub     │  Proma                            │
│  ───────────  │  ─────────── │  ───────────                      │
│  IPC 消息通道  │  fetch SSE   │  IPC 事件推送                     │
│  ↓             │  ↓           │  ↓                                │
│  composeMsg    │  ChatPayload │  useStore().set() 原子写入        │
│  合并去重      │  ↓           │  ↓                                │
│  ↓             │  animated    │  unstable_batchedUpdates          │
│  react-markdown│  Markdown    │  批量渲染 → react-markdown + Shiki│
└──────────────┴──────────────┴───────────────────────────────────┘
```

## 消息组件清单对比

| 消息类型 | CC GUI | Claude Code UI | acpx | AgentAPI | AionUi | LobeHub | Proma |
|---------|--------|---------------|------|----------|--------|---------|-------|
| **文本消息** | MarkdownBlock | ChatMessage | 文本行 | ConversationMessage | MessageText | Default/AIChatMessage | ChatMessageItem / AgentMessageItem |
| **思考/推理** | — | Reasoning / ReasoningTrigger | — | — | MessageThinking | ThinkingGroup | ThinkingBlock (含 signature) |
| **工具调用** | ToolBlock (自定义) | ToolRenderer + configs | `[tool]` 行 | `[Tool: kind]` | MessageToolCall / MessageToolGroup | ChatToolPayload | ToolActivityItem (含耗时) |
| **权限请求** | PermissionDialog (JCEF) | PermissionRequestsBanner | TTY y/N | — | MessageAcpPermission | Intervention | PermissionRequestCard (含风险分析) |
| **计划/任务** | PlanApprovalDialog | — | — | — | — | WorkflowCollapse | ExitPlanApproval |
| **压缩消息** | — | — | — | — | — | CompressedGroup | CompactingIndicator / CompressedBlock |
| **子 Agent** | — | — | — | — | — | AssistantGroup | SubAgentCard |
| **用户问题** | AskUserQuestionDialog | AskUserQuestion 表单 | — | — | AcpUserQuestion | AskUserQuestion | AskUserQuestionForm |
| **上下文窗口** | — | — | — | — | — | — | TokenUsageBar / ContextWindowBar |
| **重试状态** | — | — | — | — | — | — | RetryBanner |

## 历史消息（Session）机制

| 维度 | CC GUI | Claude Code UI | acpx | AgentAPI | AionUi | LobeHub | Proma |
|------|--------|---------------|------|----------|--------|---------|-------|
| **存储方式** | Java backend 管理 | `useSessionStore` (zustand-like Map) | JSON 文件 | ❌ 不支持 | SQLite | PostgreSQL | JSON/JSONL 文件 |
| **加载策略** | 通过 sessionId 同步 | `loadChatHistory()` 全量 → 分页渲染 | 文件 read → parse | — | `ComposeHistoryMessages` | Drizzle ORM 查询 | JSONL 全量 parse → 渲染 |
| **分页** | VirtualList 虚拟滚动 | 20 条/页，滚动加载更多 | — | — | react-virtuoso 自动 | virtua 按需加载 | 无分页（全量加载） |
| **脱机使用** | ✓ 本地文件 | ✓ sessionStore Map | ✓ 本地 JSON | — | ✓ SQLite | ✓ PostgreSQL | ✓ 本地 JSONL |
| **跨会话恢复** | ✓ | ✓ | ✓ | — | ✓ | ✓ | ✓ (含分叉/回退) |
| **会话分叉** | — | — | — | — | — | — | ✓ forkSession() |
| **文件回退** | — | — | — | — | — | — | ✓ rewindSession() |

## 各项目详情

| 项目 | 文档 |
|------|------|
| CC GUI (JetBrains 插件) | [jetbrains-cc-gui.md](./jetbrains-cc-gui.md) |
| Claude Code UI | [claudecodeui.md](./claudecodeui.md) |
| acpx | [acpx.md](./acpx.md) |
| AgentAPI | [agentapi.md](./agentapi.md) |
| AionUi | [AionUi.md](./AionUi.md) |
| LobeHub | [lobehub.md](./lobehub.md) |
| Proma | [proma.md](./proma.md) |
