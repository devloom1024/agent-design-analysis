# LobeHub：Heterogeneous Agent Adapter 与数据库消息持久化

## 结论

LobeHub 是最典型的“adapter 归一化”实现。Claude Code 与 Codex 都通过 CLI 输出 NDJSON，但进入应用后不再暴露原生事件，而是统一成 `HeterogeneousAgentEvent`，再加上 `operationId` 变成网关/服务端消费的 `AgentStreamEvent`。服务端持久化时只认这个统一事件，并写入 LobeHub 原有聊天表 `messages`、工具表 `message_plugins` 和子任务线程表 `threads`。

关键源码：

- `codes/lobehub/packages/heterogeneous-agents/src/types.ts`
- `codes/lobehub/packages/heterogeneous-agents/src/adapters/claudeCode.ts`
- `codes/lobehub/packages/heterogeneous-agents/src/adapters/codex.ts`
- `codes/lobehub/packages/heterogeneous-agents/src/spawn/agentStreamPipeline.ts`
- `codes/lobehub/src/server/services/heterogeneousAgent/HeterogeneousPersistenceHandler.ts`
- `codes/lobehub/packages/database/src/schemas/message.ts`
- `codes/lobehub/packages/types/src/message/common/tools.ts`
- `codes/lobehub/packages/types/src/message/common/base.ts`
- `codes/lobehub/packages/types/src/message/common/metadata.ts`
- `codes/lobehub/packages/types/src/message/db/item.ts`

## 统一处理 Claude Code 与 Codex

统一入口是 `AgentStreamPipeline`：

```text
CLI stdout chunk
  -> JsonlStreamProcessor
  -> CodexFileChangeTracker（仅 Codex）
  -> ClaudeCodeAdapter / CodexAdapter
  -> toStreamEvent(event, operationId)
  -> AgentStreamEvent[]
```

Claude Code 适配器读取 `claude -p --input-format stream-json --output-format stream-json --verbose --include-partial-messages` 的 NDJSON。它处理 `system:init`、`assistant`、`user.tool_result`、`stream_event`、`result`、`rate_limit_event`，并维护 message id、已流式输出文本、thinking、tool_use 列表和 subagent 上下文。

Codex 适配器读取 `codex exec --json --skip-git-repo-check --full-auto` 的 NDJSON。它处理 `thread.started`、`turn.started`、`session.configured`、`item.started`、`item.completed`、`turn.completed`、`turn.failed`、`error`，并把 Codex 的 `agent_message`、`reasoning`、`command_execution`、`file_change`、`todo_list` 等 item 统一成文本、reasoning 或工具事件。

二者汇合点是 `HeterogeneousAgentEvent`：

```ts
type HeterogeneousAgentEvent = {
  type: HeterogeneousEventType;
  data: any;
  stepIndex: number;
  timestamp: number;
};
```

字段说明：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `type` | `HeterogeneousEventType` | 统一事件类型。消费者只根据这个字段分发，不关心原始 provider。 |
| `data` | `any` | 按事件类型变化的 payload。适配器已把 provider 原生字段映射为统一结构。 |
| `stepIndex` | `number` | 当前 agent turn / step 的序号。Claude Code 用 message id 变化和 step boundary 推进；Codex 用 `turn.started` 或 agent message/tool 交替推进。 |
| `timestamp` | `number` | 事件产生时间，毫秒时间戳。服务端用它参与事件幂等 key。 |

再通过 `toStreamEvent()` 变成：

```ts
type AgentStreamEvent = {
  operationId: string;
  type: HeterogeneousEventType;
  data: any;
  stepIndex: number;
  timestamp: number;
};
```

`operationId` 是一次异构 agent 执行的全局 id，用于服务端 `operationStates` 聚合跨批次事件。

## Stream Protocol

### `stream_start`

```ts
type StreamStartData = {
  assistantMessage?: { id: string };
  model?: string;
  provider?: string;
  newStep?: boolean; // Codex/服务端逻辑实际使用
};
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `assistantMessage.id` | 可选，指定已有 assistant message id。异构 CLI 常由 orchestrator 预建消息，因此这个字段可不存在。 |
| `model` | 当前 turn 模型。Claude Code 可从 `message_start` / assistant message 得到，Codex 从 session/turn metadata 得到。 |
| `provider` | 适配器名称，如 `claude-code` 或 `codex`。 |
| `newStep` | 表示同一个 operation 内开启新的 assistant step。服务端收到后创建新的 assistant message，并把 parent 指向上一个工具结果或上一个 assistant。 |

### `stream_chunk`

```ts
type StreamChunkData = {
  chunkType: 'text' | 'reasoning' | 'tools_calling';
  content?: string;
  reasoning?: string;
  toolsCalling?: ToolCallPayload[];
  subagent?: SubagentEventContext;
};
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `chunkType` | `text` 表示回答文本，`reasoning` 表示思考内容，`tools_calling` 表示本 step 当前完整工具调用数组。 |
| `content` | `chunkType='text'` 时使用，追加到 assistant `content`。 |
| `reasoning` | `chunkType='reasoning'` 时使用，追加到 assistant `reasoning.content`。 |
| `toolsCalling` | `chunkType='tools_calling'` 时使用。注意适配器传的是当前 step 累积后的完整工具列表，不只是最新工具，防止 DB 或 UI 覆盖旧工具。 |
| `subagent` | 如果事件来自子 agent，则携带 parent tool call id、spawn 元数据和子 agent turn id。主 agent 事件不带。 |

`ToolCallPayload`：

```ts
type ToolCallPayload = {
  id: string;
  apiName: string;
  arguments: string;
  identifier: string;
  type: string;
  result_msg_id?: string; // 持久化后回填
};
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `id` | provider 原生 tool call id。用于把 tool_use 与 tool_result 关联。 |
| `apiName` | 工具 API 名。Claude Code 使用 tool_use.name 或改写后的 `askUserQuestion`；Codex 使用 item type，如 `command_execution`、`file_change`、`todo_list`。 |
| `arguments` | JSON 字符串形式的工具参数。Claude Code 来自 `tool_use.input`；Codex 对 command/file/todo item 做序列化。 |
| `identifier` | 工具来源标识，Codex 固定为 `codex`，Claude Code 对应 Claude Code 工具来源。 |
| `type` | LobeHub tool render type，异构 adapter 中通常是 `default`。 |
| `result_msg_id` | DB 创建 tool message 后回填到 assistant `tools[]`，指向结果消息 id。 |

`SubagentEventContext`：

```ts
type SubagentEventContext = {
  parentToolCallId: string;
  spawnMetadata?: {
    description?: string;
    prompt?: string;
    subagentType?: string;
  };
  subagentMessageId?: string;
};
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `parentToolCallId` | 主 agent 中触发子 agent 的 tool_use id。服务端用它找到或创建对应 thread。 |
| `spawnMetadata.description` | 子 agent 短标题，Claude Code 从 Task/Agent tool input 中提取。 |
| `spawnMetadata.prompt` | 子 agent 初始 prompt。服务端会写成 thread 内第一条 `role='user'` message。 |
| `spawnMetadata.subagentType` | 子 agent 模板/类型。用于 thread title 和 metadata。 |
| `subagentMessageId` | 子 agent CLI 当前 turn 的 message id。变化时服务端在 thread 中创建新的 assistant message。 |

### `tool_start`

```ts
type ToolStartData = {
  toolCallId: string;
  subagent?: SubagentEventContext;
};
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `toolCallId` | 开始执行的工具 id。服务端持久化层当前把它视作生命周期事件，不直接写 DB。 |
| `subagent` | 子 agent 工具时存在。 |

### `tool_result`

```ts
type ToolResultData = {
  toolCallId: string;
  content: string;
  isError?: boolean;
  pluginState?: Record<string, any>;
  subagent?: SubagentEventContext;
};
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `toolCallId` | 与 `ToolCallPayload.id` 对应。 |
| `content` | 工具结果展示文本。Codex command 使用 `aggregated_output`；Codex file/todo/collab 会合成摘要；Claude Code 来自 `tool_result.content`。 |
| `isError` | 工具是否失败。Codex 根据 `status` 和 `exit_code` 推断。 |
| `pluginState` | 结构化渲染状态。Codex command 会放 `stdout`、`output`、`exitCode`、`success`；Codex todo/file 会放 todo 或 change summary；Claude TodoWrite 也会合成共享 todo 状态。 |
| `subagent` | 如果结果属于子 agent 工具则存在。 |

### `tool_end`

```ts
type ToolEndData = {
  toolCallId: string;
  isSuccess: boolean;
  subagent?: SubagentEventContext;
};
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `toolCallId` | 结束的工具 id。 |
| `isSuccess` | 是否成功结束。Codex 从 item completion 推断；flush 时未完成工具会以 `false` 结束。 |
| `subagent` | 子 agent 工具时存在。 |

### `step_complete`

```ts
type StepCompleteData = {
  phase: 'turn_metadata' | 'result_usage';
  provider?: string;
  model?: string;
  usage?: UsageData;
  costUsd?: number;
};
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `phase` | `turn_metadata` 表示单个 LLM turn 的模型/usage；`result_usage` 表示整个会话结束时的总 usage。 |
| `provider` | CLI adapter 名称，如 `claude-code`、`codex`。 |
| `model` | 当前 turn 使用的模型。 |
| `usage` | 归一化 token usage。服务端写入 assistant message 的 `metadata.usage`。 |
| `costUsd` | CLI 结果中如果有成本，放在这里。 |

`UsageData`：

```ts
type UsageData = {
  inputCacheMissTokens: number;
  inputCachedTokens?: number;
  inputWriteCacheTokens?: number;
  totalInputTokens: number;
  totalOutputTokens: number;
  totalTokens: number;
};
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `inputCacheMissTokens` | 未命中缓存的新输入 token。Claude Code 从 `input_tokens` 映射，Codex 从 `input_tokens` 映射。 |
| `inputCachedTokens` | 命中缓存的输入 token。Claude Code 从 `cache_read_input_tokens` 映射，Codex 从 `cached_input_tokens` 映射。 |
| `inputWriteCacheTokens` | 写入缓存的输入 token。Claude Code 从 `cache_creation_input_tokens` 映射；Codex 当前通常没有。 |
| `totalInputTokens` | 输入总 token，包含 cache read/write/miss。 |
| `totalOutputTokens` | 输出 token。 |
| `totalTokens` | 输入与输出合计。 |

### `agent_runtime_end`

```ts
type AgentRuntimeEndData = {
  // adapter 可带 session/result 信息；持久化层主要把它作为终止 flush 信号
};
```

字段说明：该事件在服务端触发 final flush，把 accumulated content/reasoning/model/provider 写回当前 assistant message，并结束 operation state。

### `error`

```ts
type HeterogeneousTerminalErrorData = {
  message: string;
  agentType?: string;
  clearEchoedContent?: boolean;
  code?: string;
  docsUrl?: string;
  error?: string;
  installCommands?: readonly string[];
  rateLimitInfo?: HeterogeneousRateLimitInfo;
  stderr?: string;
};
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `message` | 面向用户的错误信息。 |
| `agentType` | `claude-code` 或 `codex`。 |
| `clearEchoedContent` | 如果 CLI 先把同样错误串流进 content，再发结构化错误，持久化时可清空重复 content。 |
| `code` | 错误分类，如 `auth_required`、`rate_limit`。 |
| `docsUrl` | 安装/认证文档链接。 |
| `error` | 原始错误文本。 |
| `installCommands` | 可选安装命令提示。 |
| `rateLimitInfo` | Claude Code rate limit 结构化信息。 |
| `stderr` | CLI stderr 或可等价展示的原始错误。 |

## 持久化 Message 格式

### `messages` 表

LobeHub 使用 PostgreSQL/Drizzle `messages` 表存聊天消息。异构 agent 持久化主要写 assistant、tool、subagent thread 内 user/assistant 消息。

字段逐项说明：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | `text` | 消息主键，由 `idGenerator('messages')` 生成。 |
| `role` | `varchar255` | 消息角色。异构 agent 常用 `assistant`、`tool`，子 agent thread 初始化会创建 `user`。 |
| `content` | `text` | 消息正文。`stream_chunk.text` 被累加后写入 assistant；tool message 的 content 由 `tool_result.content` 回填。 |
| `editorData` | `jsonb` | 富文本编辑器数据。异构 agent 路径通常不写。 |
| `summary` | `text` | 消息摘要。异构 agent 路径通常不写。 |
| `reasoning` | `jsonb` / `ModelReasoning` | 推理内容。`stream_chunk.reasoning` 被写为 `{ content: accumulatedReasoning }`。还可包含 `duration`、`signature`、`isMultimodal` 等。 |
| `search` | `jsonb` / `GroundingSearch` | grounding/search 结果。异构 agent 当前路径通常不写。 |
| `metadata` | `jsonb` | 消息元数据。`step_complete.phase='turn_metadata'` 将 `usage` 写到 `metadata.usage`。 |
| `model` | `text` | 当前 turn 模型。来自 `step_complete.model` 或 step boundary 前的 `lastModel`。 |
| `provider` | `text` | provider/adapter，如 `claude-code`、`codex`。来自 `step_complete.provider` 或 stream start。 |
| `favorite` | `boolean` | 收藏标记，默认 `false`。异构 agent 不直接使用。 |
| `error` | `jsonb` | 错误对象。`error` stream event 被转换为 `ChatMessageError` 写入当前 assistant。 |
| `tools` | `jsonb` | assistant message 上的工具调用数组，元素为 `ChatToolPayload`。工具创建时先预注册，再回填 `result_msg_id`。 |
| `traceId` | `text` | observability trace id。异构 agent 路径通常不写。 |
| `observationId` | `text` | observability observation id。异构 agent 路径通常不写。 |
| `clientId` | `text` | 客户端侧幂等 id。服务端异构持久化通常不写。 |
| `userId` | `text` | 所属用户，外键到 `users.id`。模型层会补。 |
| `sessionId` | `text` | 旧会话外键。异构 topic/thread 路径通常主要用 `topicId`。 |
| `topicId` | `text` | 所属 topic。`HeterogeneousPersistenceHandler.ingest()` 必须传入。 |
| `threadId` | `text` | 所属 thread。主 agent 消息为空；子 agent thread 内消息写该 thread id。 |
| `parentId` | `text` | 父消息 id。主 agent 新 step 会接到上个 tool result 或 assistant；tool message 父级是 assistant；子 agent 消息按 thread 内链路连接。 |
| `quotaId` | `text` | 引用消息 id。异构 agent 不直接使用。 |
| `agentId` | `text` | topic 关联 agent id。持久化 handler 从 topic 恢复后写入。 |
| `groupId` | `text` | 群组 id。异构 agent 不直接使用。 |
| `targetId` | `text` | 群聊/DM 目标。异构 agent 不直接使用。 |
| `messageGroupId` | `varchar255` | 多模型并行消息组。异构 agent 不直接使用。 |
| `createdAt` | timestamp helper | 创建时间。 |
| `updatedAt` | timestamp helper | 更新时间。 |

### `message_plugins` 表

每个 tool call 都创建一条 `role='tool'` message，并在 `message_plugins` 表保存工具元数据。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | `text` | 主键，同时外键到 `messages.id`。也就是 tool message 的 message id。 |
| `toolCallId` | `text` | provider 原生 tool call id，用于 `tool_result` 回填。 |
| `type` | `text` | 工具渲染类型，默认 `default`。 |
| `intervention` | `jsonb` | 人工介入状态，如 `pending`、`approved`、`rejected`。 |
| `apiName` | `text` | 工具 API 名，对应 `ToolCallPayload.apiName`。 |
| `arguments` | `text` | 工具参数 JSON 字符串，对应 `ToolCallPayload.arguments`。 |
| `identifier` | `text` | 工具来源，对应 `ToolCallPayload.identifier`。 |
| `state` | `jsonb` | 工具结果结构化状态。`tool_result.pluginState` 写到这里。 |
| `error` | `jsonb` | 工具错误。`tool_result.isError` 为真时写 `{ message: content }`。 |
| `clientId` | `text` | 客户端幂等 id。 |
| `userId` | `text` | 所属用户。 |

### `ChatToolPayload`

写入 assistant `tools` JSONB 的结构：

| 字段 | 说明 |
| --- | --- |
| `apiName` | 工具名。 |
| `arguments` | JSON 字符串参数。 |
| `executor` | 执行位置，可选 `client` 或 `server`。异构 adapter 通常不设置。 |
| `id` | tool call id。 |
| `identifier` | 工具来源标识。 |
| `intervention` | 人工介入状态。 |
| `result_msg_id` | 对应 tool message id，phase 2 创建 tool message 后回填。 |
| `source` | 工具来源，如 builtin/client/mcp。异构 adapter 通常不设置。 |
| `thoughtSignature` | thinking/tool 关联签名。 |
| `type` | 渲染类型。 |

### `ChatMessageError`

写入 `messages.error` 的结构：

| 字段 | 说明 |
| --- | --- |
| `type` | 错误类型。异构 runtime error 使用 `AgentRuntimeErrorType.AgentRuntimeError`。 |
| `message` | 展示用错误消息。 |
| `body` | 原始结构化错误体，例如 `HeterogeneousTerminalErrorData`。 |

## 关键持久化流程

1. `stream_chunk.text/reasoning` 先进入 per-operation 内存 accumulator。
2. 每批 ingest 后 `flushBatchContent()` 把 accumulator 写回当前 assistant，防止 serverless 多实例导致内容截断。
3. `tools_calling` 触发三阶段工具持久化：
   - assistant `tools[]` 预注册；
   - 创建 `role='tool'` message 和 `message_plugins`；
   - 把 tool message id 回填为 `result_msg_id` 再写回 assistant。
4. `tool_result` 用 `toolCallId -> tool message id` 映射更新 tool message 的 content/error/state。
5. `stream_start.newStep` 创建新的 assistant message，并链到上一个 tool result 或上个 assistant。
6. 子 agent 通过 `SubagentEventContext.parentToolCallId` 创建 isolation thread，thread 内也按 user/assistant/tool message 链式保存。

