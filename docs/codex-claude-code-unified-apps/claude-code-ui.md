# Claude Code UI：Provider Sessions 统一层

## 范围与入口

Claude Code UI 同时支持 `claude` 与 `codex`：

- `server/shared/types.ts` 定义 `LLMProvider = 'claude' | 'codex' | 'gemini' | 'cursor'`。
- `server/shared/interfaces.ts` 定义 `IProvider.sessions.normalizeMessage(raw, sessionId)` 和 `fetchHistory()`。
- WebSocket 入口在 `chat-websocket.service.ts`：`claude-command` 走 `queryClaudeSDK()`，`codex-command` 走 `queryCodex()`。
- `sessionsService.normalizeMessage(provider, raw, sessionId)` 通过 provider registry 调度到 `claude-sessions.provider.ts` 或 `codex-sessions.provider.ts`。

统一方式是典型 provider adapter：Claude SDK 和 Codex SDK 的原生事件先各自转换一次，再输出同一个 `NormalizedMessage`。

## Stream Protocol

### WebSocket 入站命令

| type | 字段 | 说明 |
| --- | --- | --- |
| `claude-command` | `command`, `options` | 启动 Claude SDK turn。 |
| `codex-command` | `command`, `options` | 启动 Codex SDK turn。 |
| `abort-session` | `provider`, `sessionId` | 根据 provider 分派到 Claude/Codex abort。 |
| `claude-permission-response` | `requestId`, `allow`, `updatedInput`, `message`, `rememberEntry` | 只用于 Claude permission approval。Codex 侧通过 SDK sandbox/approval policy 配置。 |
| `check-session-status` | `provider`, `sessionId` | 查询 active session，返回 `session-status`。 |
| `get-active-sessions` | 无 | 返回各 provider 活跃 session 列表。 |

### 统一出站消息：`NormalizedMessage`

所有 provider 出站都使用：

```ts
type NormalizedMessage = {
  id: string;
  sessionId: string;
  timestamp: string;
  provider: 'claude' | 'codex' | 'gemini' | 'cursor';
  kind: MessageKind;
  role?: 'user' | 'assistant';
  content?: string;
  displayText?: string;
  commandName?: string;
  commandMessage?: string;
  commandArgs?: string;
  isLocalCommand?: boolean;
  isLocalCommandStdout?: boolean;
  isCompactSummary?: boolean;
  images?: unknown;
  toolName?: string;
  toolInput?: unknown;
  toolId?: string;
  toolResult?: { content?: string; isError?: boolean; toolUseResult?: unknown };
  isError?: boolean;
  text?: string;
  tokens?: number;
  canInterrupt?: boolean;
  requestId?: string;
  input?: unknown;
  context?: unknown;
  reason?: string;
  newSessionId?: string;
  status?: string;
  summary?: string;
  tokenBudget?: unknown;
  subagentTools?: unknown;
  toolUseResult?: unknown;
  sequence?: number;
  rowid?: number;
  [key: string]: unknown;
}
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `id` | 前端去重和列表 key。实时事件由 `createNormalizedMessage()` 生成；历史消息可派生自原生 uuid、tool id 或 rowid。 |
| `sessionId` | provider 原生 session/thread id。Claude 是 Claude session id；Codex 是 Codex thread id。 |
| `timestamp` | ISO 时间。实时事件生成时写入；历史事件从 JSONL/SQLite 读取。 |
| `provider` | 统一 provider discriminator，用于前端选择渲染逻辑和 abort/status 路由。 |
| `kind` | 统一事件类型，见下表。 |
| `role` | 对话角色。文本、thinking、tool_use 多为 `assistant`；用户历史消息为 `user`。 |
| `content` | 文本正文、delta 文本、错误内容或 tool result 文本。 |
| `displayText` | 展示层友好文本，主要用于本地命令、compact summary 等。 |
| `commandName` / `commandMessage` / `commandArgs` | Claude 本地 slash/local command 的结构化字段。 |
| `isLocalCommand` | 标记是否为本地命令消息。 |
| `isLocalCommandStdout` | 标记本地命令 stdout。 |
| `isCompactSummary` | 标记 compact summary 历史消息。 |
| `images` | 多模态图片或附件信息，provider 自行填充。 |
| `toolName` | 工具名，如 Claude `Bash`、Codex `command_execution`。 |
| `toolInput` | 工具输入对象。Claude 来自 `tool_use.input`；Codex 来自 item 字段。 |
| `toolId` | 工具调用 id，用于把 `tool_use` 与 `tool_result` 关联。 |
| `toolResult` | 工具结果对象；`content` 是结果文本，`isError` 标记错误，`toolUseResult` 保留 provider 原生结果。 |
| `isError` | 当前消息是否代表错误。 |
| `text` | 状态类事件的短文本，例如 `token_budget`。 |
| `tokens` | token 数值字段，兼容部分 provider 历史。 |
| `canInterrupt` | UI 是否可中断当前执行。 |
| `requestId` | permission request 或其它异步请求 id。 |
| `input` | permission/tool prompt 输入。 |
| `context` | provider 额外上下文。 |
| `reason` | cancel/permission cancel 原因。 |
| `newSessionId` | 新 session 创建后告知前端的 canonical id。 |
| `status` | 状态字符串。 |
| `summary` | session summary 或标题摘要。 |
| `tokenBudget` | token usage/budget 对象，例如 `{ used, total }`。 |
| `subagentTools` | 子代理工具信息，主要来自 Claude 侧扩展。 |
| `toolUseResult` | 保留原生 tool use result，避免丢 provider 私有字段。 |
| `sequence` | 历史排序/流式顺序字段。 |
| `rowid` | SQLite provider 如 Cursor 的排序辅助；Codex/Claude 历史通常不依赖它。 |
| index signature | 允许 provider 扩展字段，但正式渲染仍应优先使用上方字段。 |

### `MessageKind` 类型

| kind | 数据结构 | 说明 |
| --- | --- | --- |
| `text` | `{ role, content }` | 完整文本消息或历史文本块。 |
| `thinking` | `{ role:'assistant', content }` | Claude thinking block 或 Codex reasoning item。 |
| `tool_use` | `{ role:'assistant', toolId, toolName, toolInput, content? }` | 工具调用声明。 |
| `tool_result` | `{ role?: 'user', toolId, toolResult, content? }` | 工具返回。历史加载时通常会合并到对应 `tool_use`，不单独显示。 |
| `stream_delta` | `{ content }` | token/delta 流。Claude 对 `content_block_delta.text_delta` 直接使用；Codex 当前主要发送 completed item，较少真正 token delta。 |
| `stream_end` | `{}` | 一段流结束。 |
| `error` | `{ content, isError?: true }` | provider/runtime 错误。 |
| `complete` | `{ exitCode, aborted?, success?, isNewSession? }` | turn/process 完成。 |
| `status` | `{ text, tokenBudget?, status? }` | token budget 或运行状态。 |
| `permission_request` | `{ requestId, toolName, input }` | Claude permission 请求。 |
| `permission_cancelled` | `{ requestId, reason }` | Claude permission 请求取消。 |
| `session_created` | `{ newSessionId }` | 新建 Claude/Codex session 后通知前端。 |
| `interactive_prompt` | `{ input?, context? }` | 交互提示。 |
| `task_notification` | `{ summary?, status? }` | 任务通知消息。 |

### Codex 原生事件映射

`openai-codex.js` 使用 OpenAI Codex SDK：

| Codex SDK event | transform 后 | NormalizedMessage |
| --- | --- | --- |
| `thread.started` | `{ type:'thread_started', threadId }` | `session_created`，并保存 active session。 |
| `turn.started` | `{ type:'turn_started' }` | 通常不直接展示。 |
| `item.started` / `item.updated` | 跳过 | 避免未完成工具状态噪声。 |
| `item.completed` + `agent_message` | `{ type:'item', itemType:'agent_message', message:{role:'assistant', content:item.text} }` | `kind:'text'`。 |
| `item.completed` + `reasoning` | `itemType:'reasoning'` | `kind:'thinking'`。 |
| `item.completed` + `command_execution` | `{ command, output, exitCode, status }` | `kind:'tool_use'`，并可能生成/关联 tool result。 |
| `item.completed` + `file_change` | `{ changes, status }` | `kind:'tool_use'`，工具名为文件变更。 |
| `item.completed` + `mcp_tool_call` | `{ server, tool, arguments, result, error, status }` | `kind:'tool_use'` 或 `tool_result`。 |
| `item.completed` + `web_search` | `{ query }` | `kind:'tool_use'`。 |
| `item.completed` + `todo_list` | `{ items }` | `kind:'tool_use'`。 |
| `turn.completed` | `{ type:'turn_complete', usage }` | `complete` + `status(token_budget)`。 |
| `turn.failed` | `{ type:'turn_failed', error }` | `error`。 |
| `error` | `{ type:'error', message }` | `error`。 |

### Claude 原生事件映射

Claude SDK query stream 在 `claude-sdk.js` 中先转换为内部 raw message，再调用 `claude-sessions.provider.ts`：

| Claude raw | NormalizedMessage |
| --- | --- |
| `content_block_delta` + `text_delta` | `stream_delta`，`content = delta.text`。 |
| message stop/end | `stream_end`。 |
| `assistant.message.content[].text` | `text`。 |
| `assistant.message.content[].thinking` | `thinking`。 |
| `assistant.message.content[].tool_use` | `tool_use`，字段落到 `toolId/toolName/toolInput`。 |
| `user.message.content[].tool_result` | `tool_result`，再在历史加载中合并到对应 tool use。 |
| permission callback | `permission_request` / `permission_cancelled`。 |

## Message 持久化

Claude Code UI 不把完整对话复制进自己的 SQLite `messages` 表。它的持久化分两层：

1. Provider 原生 transcript：Claude/Codex 各自的 JSONL/session 文件是完整消息来源。
2. 应用 SQLite `sessions` 表：只存 session 索引和展示元数据。

### SQLite `sessions` 表

定义在 `server/modules/database/schema.ts`：

```sql
CREATE TABLE IF NOT EXISTS sessions (
  session_id TEXT NOT NULL,
  provider TEXT NOT NULL DEFAULT 'claude',
  custom_name TEXT,
  project_path TEXT,
  jsonl_path TEXT,
  isArchived BOOLEAN DEFAULT 0,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (session_id),
  FOREIGN KEY (project_path) REFERENCES projects(project_path)
    ON DELETE SET NULL
    ON UPDATE CASCADE
);
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `session_id` | 主键。Claude session id 或 Codex thread/session id。 |
| `provider` | `claude` 或 `codex`，决定 `fetchHistory()` 调哪个 provider reader。 |
| `custom_name` | 用户自定义 session 名称；为空时用 session id 或 provider 解析出的标题。 |
| `project_path` | 所属项目路径，关联 `projects.project_path`。 |
| `jsonl_path` | provider 原生历史文件路径。删除 session 时可选择一并删除该文件。 |
| `isArchived` | 软删除/归档标记。active 列表只查 `0`。 |
| `created_at` | 首次索引/创建时间。 |
| `updated_at` | 最近同步或更新名称/状态时间。 |

### 历史消息格式

历史 API 返回的不是数据库 row，而是 provider reader 重新解析原生历史后生成的：

```ts
type FetchHistoryResult = {
  messages: NormalizedMessage[];
  total: number;
  hasMore: boolean;
  offset: number;
  limit: number | null;
  tokenUsage?: unknown;
}
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `messages` | 已转换成 `NormalizedMessage` 的可渲染历史。`tool_result` 常被合并到 `tool_use` 后过滤。 |
| `total` | 原始可用消息总数。 |
| `hasMore` | 是否还有分页数据。 |
| `offset` | 当前页起始偏移。 |
| `limit` | 当前页大小；`null` 表示不限制。 |
| `tokenUsage` | provider 聚合 token 使用信息。 |

### 设计评价

优点是不会破坏 provider 原生历史，Claude 和 Codex 的 session resume 也能继续依赖原生文件。缺点是应用级 message schema 不掌握完整对话，跨 provider 搜索、迁移和一致性修复必须反复解析原生 transcript。
