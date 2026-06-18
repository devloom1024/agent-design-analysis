# acpx：用 ACP 统一 Codex 与 Claude Code

## 结论

acpx 的统一方式是“把不同 agent 先变成 ACP agent”。它内置 `codex` 与 `claude`：

```ts
codex:  npx @zed-industries/codex-acp@^0.12.0
claude: npx -y @agentclientprotocol/claude-agent-acp@^0.31.0
```

应用层通过 `AcpRuntime` 面向同一组方法：`ensureSession()`、`startTurn()`、`runTurn()`、`cancel()`、`close()`。stream 的原始来源是 ACP JSON-RPC `session/update`，运行时再转成 `AcpRuntimeEvent`。持久化有两层：一层是 session record JSON，保存归一化后的 `SessionConversation`；另一层是 ACP JSON-RPC event log，保存原始协议消息。

关键源码：

- `codes/acpx/src/agent-registry.ts`
- `codes/acpx/src/runtime/public/contract.ts`
- `codes/acpx/src/runtime/public/events.ts`
- `codes/acpx/src/session/conversation-model.ts`
- `codes/acpx/src/session/events.ts`
- `codes/acpx/src/session/persistence/serialize.ts`
- `codes/acpx/src/session/persistence/repository.ts`
- `codes/acpx/src/types.ts`

## 统一处理 Claude Code 与 Codex

`AGENT_REGISTRY` 把 agent 名称映射为 ACP server 命令。`AcpClientOptions.agentCommand` 不关心它背后是 Claude 还是 Codex，只要求对方实现 ACP。

统一流程：

```text
agent name: claude/codex
  -> resolveAgentCommand()
  -> ACP adapter process
  -> ACP JSON-RPC initialize / session/new / session/prompt
  -> session/update notification
  -> recordSessionUpdate()
  -> SessionConversation
```

这种设计中，Claude 与 Codex 的差异被封装在外部 ACP adapter 包中。acpx 内部主要处理 ACP 标准结构，包括 text chunk、thought chunk、tool call、usage、mode/config/session info。

## Stream Protocol

### ACP JSON-RPC 原始事件

acpx 保存的原始协议消息类型是 `AcpJsonRpcMessage = AnyMessage`。写入 event log 前会用 `isAcpJsonRpcMessage()` 校验。典型 `session/update` 结构：

```json
{
  "jsonrpc": "2.0",
  "method": "session/update",
  "params": {
    "sessionId": "...",
    "update": {
      "sessionUpdate": "agent_message_chunk",
      "content": { "type": "text", "text": "..." }
    }
  }
}
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `jsonrpc` | JSON-RPC 版本，通常为 `2.0`。 |
| `method` | ACP 方法名。stream 主要是 `session/update`，也会有 request/response 类消息。 |
| `params.sessionId` | ACP session id。 |
| `params.update.sessionUpdate` | ACP session update 标签，决定后续 payload 结构。 |
| `params.update.*` | 具体 update payload，例如 `content`、`toolCallId`、`rawInput`、`status`、`usage`。 |
| `id` | 如果是 request/response 消息，则可能有 JSON-RPC id。event writer 会把它记录到 `lastRequestId`。 |

### `AcpSessionUpdateTag`

```ts
type AcpSessionUpdateTag =
  | 'agent_message_chunk'
  | 'agent_thought_chunk'
  | 'tool_call'
  | 'tool_call_update'
  | 'usage_update'
  | 'available_commands_update'
  | 'current_mode_update'
  | 'config_option_update'
  | 'session_info_update'
  | 'plan'
  | string;
```

字段/类型说明：

| tag | 说明 |
| --- | --- |
| `agent_message_chunk` | agent 输出正文增量，通常带 `content`。会追加到 `SessionAgentMessage.content[].Text`。 |
| `agent_thought_chunk` | agent thinking/reasoning 增量。会追加到 `SessionAgentMessage.content[].Thinking.text`。 |
| `tool_call` | 新工具调用。会创建或更新 `ToolUse` block。 |
| `tool_call_update` | 工具调用状态、输入、输出更新。会更新 `ToolUse` 与 `tool_results`。 |
| `usage_update` | token usage 更新。写入 `cumulative_token_usage` 和最近 user message 的 `request_token_usage`。 |
| `available_commands_update` | 可用命令列表更新。写入 `acpx.available_commands`。 |
| `current_mode_update` | 当前 mode 更新。写入 `acpx.current_mode_id`。 |
| `config_option_update` | 配置项更新。写入 `acpx.config_options`。 |
| `session_info_update` | session 标题/更新时间更新。写入 `title`、`updated_at`。 |
| `plan` | 计划更新。runtime event 层可转为 status 文本；conversation model 当前不将其存为 message content。 |
| `string` | 保留扩展标签，未知标签不会破坏协议。 |

### `AcpRuntimeEvent`

acpx 对外运行时 stream 是 `AcpRuntimeEvent`：

```ts
type AcpRuntimeEvent =
  | { type: 'text_delta'; text: string; stream?: 'output' | 'thought'; tag?: AcpSessionUpdateTag }
  | { type: 'status'; text: string; tag?: AcpSessionUpdateTag; used?: number; size?: number }
  | { type: 'tool_call'; text: string; tag?: AcpSessionUpdateTag; toolCallId?: string; status?: string; title?: string; kind?: ToolKind; locations?: ToolCallLocation[]; rawInput?: unknown; rawOutput?: unknown; content?: ToolCallContent[] }
  | { type: 'done'; stopReason?: string }
  | { type: 'error'; message: string; code?: string; detailCode?: string; retryable?: boolean };
```

#### `text_delta`

| 字段 | 说明 |
| --- | --- |
| `type` | 固定为 `text_delta`。 |
| `text` | 本次新增文本。来自 ACP text content 或兼容字段 `text`。 |
| `stream` | `output` 表示普通回答，`thought` 表示 thinking。 |
| `tag` | 原始 ACP update tag，便于追踪来源。 |

#### `status`

| 字段 | 说明 |
| --- | --- |
| `type` | 固定为 `status`。 |
| `text` | 状态文本，如 available commands 更新、mode 更新、config 更新、plan 首项。 |
| `tag` | 来源 update tag。 |
| `used` | 可选，用量类状态的已使用量。 |
| `size` | 可选，用量类状态的总量或大小。 |

#### `tool_call`

| 字段 | 说明 |
| --- | --- |
| `type` | 固定为 `tool_call`。 |
| `text` | 工具调用摘要。acpx 会从 command/path/query/content 等字段提取可读摘要。 |
| `tag` | `tool_call` 或 `tool_call_update`。 |
| `toolCallId` | ACP tool call id。 |
| `status` | 工具状态。`complete/done/success/failed/error/cancel` 会影响持久化中的 `is_input_complete` 与 `is_error`。 |
| `title` | 工具标题。会优先作为 `ToolUse.name`。 |
| `kind` | ACP `ToolKind`。当没有 title 时可作为工具名。 |
| `locations` | 工具涉及的位置，如文件位置。 |
| `rawInput` | 原始输入。会保存到 `ToolUse.input`，并序列化为 `raw_input`。 |
| `rawOutput` | 原始输出。会保存到 `SessionToolResult.output`，并转换为 `content.Text`。 |
| `content` | ACP 工具内容数组。用于摘要/展示。 |

#### `done`

| 字段 | 说明 |
| --- | --- |
| `type` | 固定为 `done`。 |
| `stopReason` | 停止原因。`startTurn().events` 不发 terminal event，`runTurn()` 为兼容旧消费者会发。 |

#### `error`

| 字段 | 说明 |
| --- | --- |
| `type` | 固定为 `error`。 |
| `message` | 错误消息。 |
| `code` | 错误代码。 |
| `detailCode` | 更细的错误分类，如 queue/protocol 相关 detail。 |
| `retryable` | 是否可重试。 |

## 持久化 Message 格式

### Session record 文件

路径：`~/.acpx/sessions/{encodeURIComponent(acpxRecordId)}.json`

`serializeSessionRecordForDisk()` 写入字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `schema` | string | 固定为 `acpx.session.v1`。用于版本识别。 |
| `acpx_record_id` | string | acpx 自己的 record id，也是 session 文件主键。 |
| `acp_session_id` | string | ACP session id。 |
| `agent_session_id` | string? | agent 原生 session id，经过 `normalizeRuntimeSessionId()`。 |
| `agent_command` | string | 启动 agent 的命令，例如 Codex/Claude ACP adapter 命令。 |
| `cwd` | string | session 工作目录。 |
| `name` | string? | 用户命名的 session 名。 |
| `created_at` | string | 创建时间，ISO 字符串。 |
| `last_used_at` | string | 最近使用时间。event writer 每写一条 JSON-RPC 消息会更新。 |
| `last_seq` | number | 写入 event log 的递增序号。 |
| `last_request_id` | string? | 最近 JSON-RPC request id。 |
| `event_log` | `SessionEventLog` | 原始 ACP event log 文件位置和轮转状态。 |
| `closed` | boolean? | session 是否关闭。 |
| `closed_at` | string? | 关闭时间。 |
| `pid` | number? | agent 进程 pid。 |
| `agent_started_at` | string? | agent 启动时间。 |
| `last_prompt_at` | string? | 最近 prompt 发送时间。 |
| `last_agent_exit_code` | number/null? | 最近 agent 退出码。 |
| `last_agent_exit_signal` | string? | 最近退出 signal。 |
| `last_agent_exit_at` | string? | 最近退出时间。 |
| `last_agent_disconnect_reason` | string? | 最近断开原因。 |
| `protocol_version` | string? | ACP protocol version。 |
| `agent_capabilities` | object? | agent 声明的能力。 |
| `title` | string/null? | conversation 标题。来自 `session_info_update.title`。 |
| `messages` | `SessionMessage[]` | 归一化后的会话消息。详见下节。 |
| `updated_at` | string | conversation 更新时间。 |
| `cumulative_token_usage` | `SessionTokenUsage` | 当前累计 token usage。 |
| `request_token_usage` | `Record<userMessageId, SessionTokenUsage>` | 按 user message id 记录的 request usage。 |
| `acpx` | `SessionAcpxState` | acpx UI/控制状态，如 mode、commands、config options。 |

### `SessionConversation`

```ts
type SessionConversation = {
  title?: string | null;
  messages: SessionMessage[];
  updated_at: string;
  cumulative_token_usage: SessionTokenUsage;
  request_token_usage: Record<string, SessionTokenUsage>;
};
```

| 字段 | 说明 |
| --- | --- |
| `title` | 会话标题，可为空。 |
| `messages` | 归一化消息数组，最多保留最近 200 条运行时消息。 |
| `updated_at` | 最近更新时间，每次 prompt/session update/client operation 都会刷新。 |
| `cumulative_token_usage` | ACP usage update 投影后的累计 token。 |
| `request_token_usage` | 最近 user message id 到 token usage 的映射，最多保留 100 项。 |

### `SessionMessage`

```ts
type SessionMessage =
  | { User: SessionUserMessage }
  | { Agent: SessionAgentMessage }
  | 'Resume';
```

| 变体 | 说明 |
| --- | --- |
| `{ User }` | 用户消息，来自 prompt submission 或 ACP `user_message_chunk`。 |
| `{ Agent }` | agent 消息，包含文本、thinking、工具调用与工具结果。 |
| `"Resume"` | 恢复标记，用于表达历史恢复边界。 |

### `SessionUserMessage`

```ts
type SessionUserMessage = {
  id: string;
  content: SessionUserContent[];
};
```

| 字段 | 说明 |
| --- | --- |
| `id` | 用户消息 id，通常为 `randomUUID()`。用于关联 request token usage。 |
| `content` | 用户内容数组，可包含文本、mention、图片。 |

`SessionUserContent`：

| 变体 | 字段 | 说明 |
| --- | --- | --- |
| `Text` | `Text: string` | 普通文本 prompt。会被截断到运行时最大文本长度。 |
| `Mention` | `{ uri, content }` | resource/resource_link 转成 mention。`uri` 是资源地址，`content` 是显示文本。 |
| `Image` | `{ source, size }` | 图片输入。`source` 是数据，`size` 可为宽高或 null。 |

### `SessionAgentMessage`

```ts
type SessionAgentMessage = {
  content: SessionAgentContent[];
  tool_results: Record<string, SessionToolResult>;
  reasoning_details?: unknown;
};
```

| 字段 | 说明 |
| --- | --- |
| `content` | agent 内容块。文本、thinking、redacted thinking、tool use 都在这里顺序保存。 |
| `tool_results` | tool call id 到结果的映射。acpx 把结果与 tool use 分离，避免丢失更新。 |
| `reasoning_details` | provider/adapter 额外 reasoning 细节。 |

`SessionAgentContent`：

| 变体 | 字段 | 说明 |
| --- | --- | --- |
| `Text` | `Text: string` | agent 普通输出。`agent_message_chunk` 会追加到最后一个 Text，否则新建。 |
| `Thinking` | `{ text, signature? }` | reasoning 文本。`agent_thought_chunk` 会追加，`signature` 可存 provider thinking 签名。 |
| `RedactedThinking` | `string` | 被隐藏/脱敏的 thinking。 |
| `ToolUse` | `SessionToolUse` | 工具调用结构。`tool_call` / `tool_call_update` 会 upsert。 |

### `SessionToolUse`

```ts
type SessionToolUse = {
  id: string;
  name: string;
  raw_input: string;
  input: unknown;
  is_input_complete: boolean;
  thought_signature?: string | null;
};
```

| 字段 | 说明 |
| --- | --- |
| `id` | ACP `toolCallId`。 |
| `name` | 工具名。优先来自 update `title`，其次来自 `kind`，默认 `tool_call`。 |
| `raw_input` | 输入的字符串形式。对象会 JSON.stringify，长度会截断。 |
| `input` | 原始输入对象或值。 |
| `is_input_complete` | 输入是否完成。由 status 是否包含 complete/done/success/failed/error/cancel 推断。 |
| `thought_signature` | 可选 thinking 签名。 |

### `SessionToolResult`

```ts
type SessionToolResult = {
  tool_use_id: string;
  tool_name: string;
  is_error: boolean;
  content: SessionToolResultContent;
  output?: unknown;
};
```

| 字段 | 说明 |
| --- | --- |
| `tool_use_id` | 对应 `SessionToolUse.id`。 |
| `tool_name` | 工具名，来自 tool use 或 patch。 |
| `is_error` | 工具是否失败，由 status 是否含 fail/error 推断。 |
| `content` | 展示用结果内容。rawOutput 会转成 Text；图片可用 Image。 |
| `output` | 原始输出，保留结构化值。字符串会按运行时上限截断。 |

`SessionToolResultContent`：

| 变体 | 说明 |
| --- | --- |
| `{ Text: string }` | 文本结果。 |
| `{ Image: { source, size } }` | 图片结果。 |

### `SessionTokenUsage`

```ts
type SessionTokenUsage = {
  input_tokens?: number;
  output_tokens?: number;
  cache_creation_input_tokens?: number;
  cache_read_input_tokens?: number;
};
```

| 字段 | 说明 |
| --- | --- |
| `input_tokens` | 输入 token。兼容 snake_case/camelCase 来源。 |
| `output_tokens` | 输出 token。 |
| `cache_creation_input_tokens` | 写入缓存 token。 |
| `cache_read_input_tokens` | 读取缓存 token。 |

### `SessionAcpxState`

| 字段 | 说明 |
| --- | --- |
| `reset_on_next_ensure` | 下次 ensureSession 是否重置。 |
| `current_mode_id` | 当前 mode。 |
| `desired_mode_id` | 希望切换到的 mode。 |
| `desired_config_options` | 希望设置的配置项键值。 |
| `current_model_id` | 当前模型 id。 |
| `available_models` | 可用模型列表。 |
| `available_commands` | 可用命令列表。 |
| `config_options` | ACP session config options。 |
| `session_options.model` | session 指定模型。 |
| `session_options.allowed_tools` | 允许工具列表。 |
| `session_options.max_turns` | 最大 turn 数。 |
| `session_options.system_prompt` | system prompt 或 append patch。 |

### Event log

路径由 `event_log.active_path` 指向，默认在 `~/.acpx/sessions` 下的 active/segment 文件。每行是一条 ACP JSON-RPC message。

`SessionEventLog` 字段：

| 字段 | 说明 |
| --- | --- |
| `active_path` | 当前 active event log 文件路径。 |
| `segment_count` | 当前保留的 segment 数量。 |
| `max_segment_bytes` | 单个 segment 最大字节数，超过会 rotate。 |
| `max_segments` | 最多保留 segment 数。 |
| `last_write_at` | 最近写入时间。 |
| `last_write_error` | 最近写入错误。正常为 null。 |

event writer 语义：

- 每写一条 JSON-RPC message 都追加一行 JSON。
- 超过 `max_segment_bytes` 时把 active 文件轮转成 segment。
- 写入后更新 `lastSeq`、`lastRequestId`、`lastUsedAt`、`eventLog`。
- `checkpoint` 或 close 时把 session record JSON 一起写回。

