# Agent Spaces：AgentRuntimeEvent 与 MessagePart

## 统一处理

Agent Spaces 通过 runtime adapter 统一 Claude Code 与 Codex：

- `ClaudeCodeRuntime`
- `CodexRuntime`
- 共同实现 `AgentRuntime`
- 统一输出 `AgentRuntimeEvent`
- WebSocket runner 把 runtime events 写成 channel message 的 `parts`

核心文件：

- `packages/server/src/adapters/agent-runtime-types.ts`
- `packages/server/src/adapters/claude-code-runtime/index.ts`
- `packages/server/src/adapters/codex-runtime.ts`
- `packages/server/src/ws/agent-runner.ts`
- `packages/server/src/ws/message-parts.ts`
- `packages/server/src/services/message.ts`
- `packages/server/src/services/tool-detail.ts`

## Stream Protocol

### `AgentRuntimeEvent`

| type | 数据结构 | 说明 |
| --- | --- | --- |
| `output` | `{ content: string }` | assistant 普通文本输出。 |
| `session` | `{ sessionId: string }` | provider session id。Claude/Codex resume 依赖它。 |
| `reasoning` | `{ content: string }` | 推理内容。 |
| `tool_use` | `{ id, name, input }` | 工具调用声明。 |
| `tool_result` | `{ toolUseId, content, isError? }` | 工具调用结果。 |
| `hook_event` | provider-specific | hook 或外部事件。 |

字段说明：

| 字段 | 说明 |
| --- | --- |
| `content` | 文本或 reasoning/tool result 文本。 |
| `sessionId` | 原生 Claude/Codex session id。 |
| `id` | 工具调用 id。 |
| `name` | 工具名。 |
| `input` | 工具输入 JSON。 |
| `toolUseId` | 结果关联的工具 id。 |
| `isError` | tool result 是否失败。 |

### `MessagePart`

运行时事件会被转成 channel message parts：

| part type | 关键字段 | 说明 |
| --- | --- | --- |
| `text` | text/content | 普通文本。 |
| `user_message` | text/content | 用户输入。 |
| `reasoning` | text/content | 推理块。 |
| `chain` | title/items | 多步骤链式执行。 |
| `terminal` | command/output/status | 终端命令和输出。 |
| `confirmation` | question/options/status | 审批/确认请求。 |
| `context` | files/selection | 上下文信息。 |
| `subagent` | agent/session/status | 子代理消息。 |
| `ask_user_question` | question/options/answer | ask-user 交互。 |

## Message 持久化

### 存储位置

channel messages 存为 JSON：

```text
~/.agent-spaces-data/workspaces/{workspaceId}/channels/{channelId}/messages.json
```

工具详情另存：

```text
~/.agent-spaces-data/workspaces/{workspaceId}/channels/{channelId}/tool-details.json
```

agent session/usage 另有 SQLite：

```text
~/.agent-spaces-data/agents/agents.sqlite
```

### `Message`

```ts
type Message = {
  id: string;
  channelId: string;
  senderId: string;
  senderRole?: string;
  content: string;
  type: string;
  status?: string;
  attachments?: unknown[];
  parts?: MessagePart[];
  metadata?: Record<string, unknown>;
  replies?: Message[];
  codeRef?: unknown;
  createdAt: string;
}
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `id` | message 唯一 id。 |
| `channelId` | 所属 channel。 |
| `senderId` | 发送者 id，可以是用户或 agent。 |
| `senderRole` | 发送者角色。 |
| `content` | 兼容纯文本摘要；结构化内容在 `parts`。 |
| `type` | message 类型，如普通消息、agent 输出等。 |
| `status` | 运行状态，如 pending/running/done/error。 |
| `attachments` | 附件。 |
| `parts` | 结构化 message part 数组，保存 text、reasoning、terminal、confirmation 等。 |
| `metadata` | provider/session/model/tool usage 等扩展元数据。 |
| `replies` | 嵌套回复。 |
| `codeRef` | 代码引用。 |
| `createdAt` | 创建时间。 |

### 设计评价

Agent Spaces 的优点是 runtime event 很小，message part 很适合 UI 组合渲染。缺点是 `MessagePart` 承担了大量 provider 语义，若 Codex/Claude 新增复杂事件，需要持续扩展 part union。
