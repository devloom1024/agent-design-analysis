# Stream 与入库消息形态调研

本文调研当前仓库内 7 个应用对“消息目录/消息列表”的设计方式，重点回答两个问题：

1. 消息在实时生成过程中以什么形态流动。
2. 消息在入库、恢复、分页、搜索、上下文组装时以什么形态保存。

结论先说清楚：不要把 stream chunk 直接等同于入库消息。主流实现都会拆成两层，复杂项目还会再增加运行时聚合态和原始事件日志。

## 术语

本文使用以下术语，避免把不同层级混在一起：

| 术语 | 定义 | 生命周期 | 是否适合直接渲染 | 是否适合入库为主表 |
| --- | --- | --- | --- | --- |
| Raw Provider Event | 模型 SDK、ACP、PTY、SSE 产生的原始事件 | 极短，随 provider 变化 | 通常不适合 | 不适合，除非作为审计日志 |
| Stream Event | 应用归一化后的实时事件，如 text delta、tool update、done | 一轮生成期间 | 适合实时渲染 | 不适合作为最终消息 |
| Runtime Aggregate State | 前端或服务端把 stream event 折叠后的运行中状态 | 一轮生成期间到服务端追平 | 适合渲染 | 可定期更新同一条 DB row |
| Persisted Message | 入库后的稳定消息，如 UIChatMessage、TMessage、ConversationMessage | 长期 | 适合历史渲染 | 适合 |
| Event Log | 原始或标准化事件的追加日志 | 长期或调试期 | 不直接渲染 | 适合作为回放/审计辅助表 |

用户说的“两种形态”，对应的是 Stream Event 和 Persisted Message。实际设计时建议至少显式区分这两层；如果要支持断线恢复、回放、复杂工具调用，再增加 Runtime Aggregate State 和 Event Log。

## 总体对比

| 应用 | Stream 形态 | 入库/历史形态 | 折叠策略 | 适合借鉴点 |
| --- | --- | --- | --- | --- |
| acpx | `AcpRuntimeEvent`，通过 `AsyncIterable` 输出 | `SessionRecord.messages` + `eventLog` | ACP `session/update` 转 runtime event，再由 session record 保存最终会话 | 协议事件和持久会话记录分层很清晰 |
| Claude Code UI | WebSocket `NormalizedMessage`，含 `stream_delta`、`stream_end` | 后端 JSONL 历史；前端 `serverMessages` | `realtimeMessages` 和 `serverMessages` 合并去重 | 前端“双轨合并”适合断线与服务端追平 |
| AionUi | ACP `sessionUpdate` 转 `TMessage` | 数据库 `TMessage` | 通过 `msg_id`、`toolCallId`、`sessionId` 合并 | `composeMessage` 对文本、工具、计划的合并规则明确 |
| LobeHub | `StreamingHandler` 累积 text、reasoning、tools | PostgreSQL `messages`、`message_plugins` 等 | stream 更新前端，finish 时写回消息行并刷新 | 乐观更新 + 完成落库 + 完整刷新 |
| Proma | Chat SSE；Agent `AgentEventBus` + IPC `STREAM_EVENT` | Chat/Agent JSONL | 前端 Jotai atom 保存运行中状态，最终 append JSONL | 本地应用使用 JSONL 简洁可靠 |
| AgentAPI | PTY screen diff 或 ACP chunk | `ConversationMessage`；PTY 可保存 `AgentState` JSON | 最后一条 agent 消息原地覆盖 | 终端类应用的替换式流式模型 |
| JetBrains CC GUI | `MessageCallback.onMessage(type, content)` 字符串事件 | Provider history 转前端消息；`SDKResult.messages` | 上层 handler 识别字符串 type | 简单接入多 provider，但类型安全弱 |

## 关键结论

### 1. Stream 不是消息，stream 是“消息变更事件”

Stream 事件通常只有以下职责：

- 把新 token 追加到某个输出流。
- 更新某个工具调用的状态。
- 通知 usage、permission、error、done 等控制信息。
- 给 UI 提供低延迟反馈。

它不天然具备最终消息需要的属性：稳定内容、完整 metadata、完整 tool result、最终 usage、可查询外键、可恢复排序。

### 2. 入库消息是“折叠后的事实快照”

入库消息至少应该回答这些问题：

- 谁说的：`role`。
- 属于哪里：`sessionId`、`topicId`、`threadId`、`agentId`。
- 说了什么：`content` 或 `blocks`。
- 关联什么：`parentId`、`toolCallId`、`messageGroupId`。
- 当前状态：`status`、`error`、`interrupted`。
- 可审计字段：`createdAt`、`updatedAt`、`provider`、`model`、`usage`。

入库消息是历史、搜索、恢复、上下文拼装的 source of truth。

### 3. 大多数实现中间还有“运行中聚合态”

典型例子：

- Claude Code UI 用 `realtimeMessages` 保存实时消息，用 `serverMessages` 保存历史消息，再计算 `merged`。
- AionUi 用 `composeMessage` 把相同 `msg_id` 的文本 chunk 合并成一条消息。
- LobeHub 用 `StreamingHandler.output`、`thinkingContent`、`tools` 保存当前轮聚合状态。
- Proma 用 Jotai `streamingStatesAtom` / `agentStreamingStatesAtom` 保存运行中状态。

这层是 stream 与入库之间的缓冲区，能减少 DB 写压力，也能避免 chunk 级数据污染历史记录。

## 各应用实现细节

### acpx

acpx 的设计最接近协议型架构，分为 runtime event 和 session record。

Stream 形态：

```ts
type AcpRuntimeEvent =
  | { type: 'text_delta'; text: string; stream?: 'output' | 'thought'; tag?: AcpSessionUpdateTag }
  | { type: 'status'; text: string; used?: number; size?: number }
  | { type: 'tool_call'; text: string; toolCallId?: string; status?: string; rawInput?: unknown }
  | { type: 'done'; stopReason?: string }
  | { type: 'error'; message: string; code?: string; retryable?: boolean };
```

入库形态：

```ts
type SessionMessage =
  | { User: SessionUserMessage }
  | { Agent: SessionAgentMessage }
  | 'Resume';

type SessionRecord = {
  schema: 'acpx.session.v1';
  acpxRecordId: string;
  acpSessionId: string;
  eventLog: SessionEventLog;
  messages: SessionMessage[];
  cumulative_token_usage: SessionTokenUsage;
  request_token_usage: Record<string, SessionTokenUsage>;
};
```

转换逻辑：

- ACP `agent_message_chunk` 转为 `text_delta`，`stream='output'`。
- ACP `agent_thought_chunk` 转为 `text_delta`，`stream='thought'`。
- ACP `tool_call` 和 `tool_call_update` 转为 `tool_call` runtime event。
- session record 保存折叠后的 `User/Agent/Resume`。
- `eventLog` 另存 JSON-RPC 事件，可用于审计和回放。

设计判断：

- acpx 明确区分“协议事件”和“会话记录”。
- 如果后续系统需要回放 agent 过程，应该参考 acpx 保留 event log。
- 如果只关心聊天历史，主查询应使用 `SessionRecord.messages`，不要从 event log 即时重放。

相关代码：

- `codes/acpx/src/runtime/public/contract.ts`
- `codes/acpx/src/types.ts`
- `codes/acpx/src/session/persistence/repository.ts`

### Claude Code UI

Claude Code UI 的特点是 provider-neutral：所有 provider 的实时和历史消息都转为 `NormalizedMessage`。

统一消息类型：

```ts
type MessageKind =
  | 'text'
  | 'tool_use'
  | 'tool_result'
  | 'thinking'
  | 'stream_delta'
  | 'stream_end'
  | 'error'
  | 'complete'
  | 'status'
  | 'permission_request'
  | 'permission_cancelled'
  | 'session_created'
  | 'interactive_prompt'
  | 'task_notification';

type NormalizedMessage = {
  id: string;
  sessionId: string;
  timestamp: string;
  provider: LLMProvider;
  kind: MessageKind;
  role?: 'user' | 'assistant';
  content?: string;
  toolName?: string;
  toolInput?: unknown;
  toolId?: string;
  toolResult?: unknown;
};
```

前端状态：

```ts
type SessionSlot = {
  serverMessages: NormalizedMessage[];
  realtimeMessages: NormalizedMessage[];
  merged: NormalizedMessage[];
  status: 'idle' | 'loading' | 'streaming' | 'error';
  total: number;
  hasMore: boolean;
  offset: number;
};
```

转换和合并流程：

1. WebSocket 收到 `stream_delta`。
2. 前端把 delta 追加到 `accumulatedStreamRef`。
3. 每 100ms 更新一次 `__streaming_<sessionId>` 临时消息。
4. 收到 `stream_end` 后，把临时消息转成普通 `text` 消息。
5. 后端 JSONL 历史通过 REST 拉取，进入 `serverMessages`。
6. `computeMerged(server, realtime)` 去重，服务端追平后清空 `realtimeMessages`。

设计判断：

- 这是非常值得参考的前端模型。
- 它允许实时消息和历史消息同时存在，避免刷新时 UI 闪烁。
- 它承认“服务端最终消息 ID”和“前端临时流式 ID”可能不同，通过去重处理最终一致性。

相关代码：

- `codes/claudecodeui/src/stores/useSessionStore.ts`
- `codes/claudecodeui/src/components/chat/hooks/useChatRealtimeHandlers.ts`
- `codes/claudecodeui/server/shared/types.ts`

### AionUi

AionUi 的核心是 `TMessage`，同一个结构既用于 UI，又会被写入本地数据库。

消息类型：

```ts
type TMessageType =
  | 'text'
  | 'tips'
  | 'tool_call'
  | 'tool_group'
  | 'agent_status'
  | 'acp_permission'
  | 'acp_tool_call'
  | 'codex_permission'
  | 'codex_tool_call'
  | 'plan'
  | 'thinking'
  | 'available_commands'
  | 'skill_suggest'
  | 'cron_trigger';

interface IMessage<T extends TMessageType, Content> {
  id: string;
  msg_id?: string;
  conversation_id: string;
  type: T;
  content: Content;
  createdAt?: number;
  position?: 'left' | 'right' | 'center' | 'pop';
  status?: 'finish' | 'pending' | 'error' | 'work';
}
```

Stream 转换：

- `agent_message_chunk` 转为 `text`。
- 同一段 assistant 输出复用同一个 `msg_id`。
- 每个 chunk 可以有独立 `id`，但 `msg_id` 相同，用于合并。
- `tool_call` / `tool_call_update` 转为 `acp_tool_call`，以 `toolCallId` 作为 `msg_id`。
- `plan` 复用当前 turn 的 `currentPlanMsgId`。

合并规则：

| 类型 | 合并键 | 合并行为 |
| --- | --- | --- |
| `text` | `msg_id` + 相邻同 type | 追加 `content.content` |
| `thinking` | `msg_id` | 追加 thinking 文本，done 时只更新状态 |
| `tool_call` | `content.callId` | 更新已有 tool call |
| `codex_tool_call` | `content.toolCallId` | 更新已有 tool call |
| `acp_tool_call` | `content.update.toolCallId` | 更新状态、content、rawInput |
| `tool_group` | 每个工具的 `callId` | 合并已有工具，剩余工具新建 |
| `plan` | `content.sessionId` | 合并同 session 的 plan |

入库策略：

- `addMessage` 直接插入。
- `addOrUpdateMessage` 走 `accumulate`。
- `ConversationManageWithDB` 先读取最近消息，再用同一套 `composeMessage` 计算 insert/update。
- `insert` 立即 flush；`accumulate` 延迟 2 秒 flush，降低 DB 写压力。

设计判断：

- AionUi 的优势是 UI 合并和 DB 合并使用同一套逻辑，前后表现一致。
- 风险是 `TMessage` 同时承载 UI 和持久层，长期会让 UI 字段进入数据库。
- 后续设计可保留 `msg_id` 思路，但最好区分 `StreamMessagePatch` 和 `PersistedMessage`。

相关代码：

- `codes/AionUi/src/common/chat/chatLib.ts`
- `codes/AionUi/src/process/agent/acp/AcpAdapter.ts`
- `codes/AionUi/src/process/utils/message.ts`
- `codes/AionUi/src/process/bridge/databaseBridge.ts`

### LobeHub

LobeHub 的消息模型最像完整产品形态：DB schema、service、store、stream handler 分层较全。

持久层：

- `messages` 表保存 role、content、reasoning、search、metadata、model、provider、tools、traceId、sessionId、topicId、threadId、parentId、agentId、groupId 等。
- `message_plugins` 表保存 tool call 相关字段，如 `toolCallId`、`apiName`、`arguments`、`state`、`error`、`intervention`。
- `message_groups` 表支持多模型并行和压缩组。

前端状态：

```ts
type ChatMessageState = {
  dbMessagesMap: Record<string, UIChatMessage[]>;
  messagesMap: Record<string, UIChatMessage[]>;
  messagesInit: boolean;
};
```

其中：

- `dbMessagesMap` 是原始数据库消息。
- `messagesMap` 是经过 `conversation-flow` parse 后的展示消息。
- 修改时先更新 `dbMessagesMap`，再 parse 成 `messagesMap`。

Stream 处理：

```ts
class StreamingHandler {
  private output = '';
  private thinkingContent = '';
  private tools?: ChatToolPayload[];

  handleChunk(chunk: StreamChunk): void;
  handleFinish(finishData: FinishData): Promise<StreamingResult>;
}
```

关键流程：

1. 发送前创建用户消息和 assistant placeholder。
2. LLM 回调 `onMessageHandle(chunk)` 时进入 `StreamingHandler.handleChunk`。
3. 文本 chunk 更新 `output`，reasoning chunk 更新 `thinkingContent`，tool chunk 更新 `tools`。
4. 每次更新通过 `internal_dispatchMessage` 乐观更新前端 store。
5. `onFinish` 调用 `handler.handleFinish` 得到完整结果。
6. `optimisticUpdateMessageContent` 调用 `messageService.updateMessage` 写入数据库。
7. 服务端返回完整 messages，前端 `replaceMessages` 刷新 DB 消息列表。

设计判断：

- LobeHub 的设计适合有服务端 DB、复杂工具调用、多 agent、多话题的应用。
- 它把 stream 聚合逻辑收在 `StreamingHandler`，避免分散在 UI 组件里。
- 它使用“乐观前端更新 + 完成后权威刷新”的最终一致性策略。

相关代码：

- `codes/lobehub/src/store/chat/agents/StreamingHandler.ts`
- `codes/lobehub/src/store/chat/agents/createAgentExecutors.ts`
- `codes/lobehub/src/store/chat/slices/message/actions/query.ts`
- `codes/lobehub/packages/database/src/schemas/message.ts`
- `codes/lobehub/packages/database/src/models/message.ts`

### Proma

Proma 同时有 Chat 模式和 Agent 模式，两套流式通道相互独立。

Chat 模式：

- 请求使用 SSE。
- `streamSSE` 负责通用 SSE 读取。
- provider adapter 解析各家格式。
- 通过 IPC 推送：
  - `STREAM_CHUNK`
  - `STREAM_REASONING`
  - `STREAM_TOOL_ACTIVITY`
  - `STREAM_COMPLETE`
  - `STREAM_ERROR`
- 前端 `streamingStatesAtom` 以 `conversationId` 为 key 存运行中状态。

Chat 入库：

- 会话索引在 `~/.proma/conversations.json`。
- 消息逐行追加到 `~/.proma/conversations/{id}.jsonl`。
- 用户消息发送前 append。
- assistant 完成后 append。

Agent 模式：

- SDK 消息进入 `AgentEventBus`。
- `AgentEventBus.emit(sessionId, payload)` 经中间件后分发。
- IPC `AGENT_IPC_CHANNELS.STREAM_EVENT` 推给渲染进程。
- 前端把 payload 转成 legacy event，再折叠进 `agentStreamingStatesAtom`。

Agent 入库：

- 会话索引在 `~/.proma/agent-sessions.json`。
- 消息逐行追加到 `~/.proma/agent-sessions/{id}.jsonl`。
- 新格式直接追加 `SDKMessage`，读取时通过 `type` 区分新旧格式。

设计判断：

- JSONL 非常适合桌面本地应用：简单、可 append、可恢复、可人工排查。
- 如果需要复杂查询和多维筛选，JSONL 后期会吃力，需要额外索引或迁移到 DB。
- Chat 与 Agent 通道拆开后，两个模式可以同时运行，互不污染。

相关代码：

- `codes/Proma/packages/core/src/providers/sse-reader.ts`
- `codes/Proma/apps/electron/src/main/lib/chat-service.ts`
- `codes/Proma/apps/electron/src/main/lib/conversation-manager.ts`
- `codes/Proma/apps/electron/src/main/lib/agent-session-manager.ts`
- `codes/Proma/apps/electron/src/main/lib/agent-event-bus.ts`
- `codes/Proma/apps/electron/src/renderer/atoms/chat-atoms.ts`
- `codes/Proma/apps/electron/src/renderer/atoms/agent-atoms.ts`

### AgentAPI

AgentAPI 是终端/PTY 风格应用，核心目标不是保留所有 chunk，而是从屏幕变化中抽取最新对话。

API 消息：

```go
type Message struct {
    Id      int
    Content string
    Role    ConversationRole
    Time    time.Time
}

type MessageRequestBody struct {
    Content string
    Type    MessageType // "user" | "raw"
}
```

内部消息：

```go
type ConversationMessage struct {
    Id      int
    Message string
    Role    ConversationRole // "user" | "agent"
    Time    time.Time
}
```

PTY stream 策略：

1. 定时读取 terminal screen。
2. `screenDiff(oldScreen, newScreen)` 算 agent 最新输出。
3. 如果最后一条是 user，则追加新的 agent 消息。
4. 如果最后一条是 agent，则原地覆盖最后一条 agent 消息。
5. EventEmitter 假设只有最后一条消息会变化，向订阅者发 `message_update`。

持久化：

```go
type AgentState struct {
    Version           int
    Messages          []ConversationMessage
    InitialPrompt     string
    InitialPromptSent bool
}
```

`SaveState` 全量写一个 JSON 文件，使用 temp file + rename 原子替换。

特殊点：

- `MessageTypeRaw` 直接写入 PTY，不保存到历史。
- ACP 模式也使用 placeholder agent 消息，chunk 到来时覆盖最后一条 agent 消息。

设计判断：

- 替换式策略适合终端代理，因为屏幕本身就是一个不断变化的 snapshot。
- 不适合需要审计每个工具步骤的应用。
- 如果采用该策略，应明确约束：只能追加新消息或更新最后一条消息。

相关代码：

- `codes/agentapi/lib/httpapi/models.go`
- `codes/agentapi/lib/screentracker/conversation.go`
- `codes/agentapi/lib/screentracker/pty_conversation.go`
- `codes/agentapi/x/acpio/acp_conversation.go`
- `codes/agentapi/lib/httpapi/events.go`

### JetBrains CC GUI

CC GUI 是最轻量的回调式设计。

Stream 接口：

```java
public interface MessageCallback {
    void onMessage(String type, String content);
    void onError(String error);
    void onComplete(SDKResult result);
}
```

常见 type：

| type | 含义 |
| --- | --- |
| `content` | 完整内容 |
| `content_delta` | 文本增量 |
| `thinking_delta` | thinking 增量 |
| `stream_start` | 流开始 |
| `stream_end` | 流结束 |
| `message_end` | 消息结束 |
| `session_id` | 会话 ID |
| `tool_result` | 工具结果 |
| `usage` | token usage |

结果对象：

```java
public class SDKResult {
    public boolean success;
    public String error;
    public int messageCount;
    public List<Object> messages;
    public String rawOutput;
    public String finalResult;
}
```

历史恢复：

- `SessionMessageOrchestrator` 从 provider 读取 session history。
- 清空当前 state。
- 将 provider history parse 成 `ClaudeSession.Message`。
- 通知前端刷新。

设计判断：

- 字符串 type 接入快，但后续维护成本高。
- 如果后续要统一多 provider，建议尽快升级为 discriminated union，而不是继续扩展字符串。

相关代码：

- `codes/jetbrains-cc-gui/src/main/java/com/github/claudecodegui/provider/common/MessageCallback.java`
- `codes/jetbrains-cc-gui/src/main/java/com/github/claudecodegui/provider/common/SDKResult.java`
- `codes/jetbrains-cc-gui/src/main/java/com/github/claudecodegui/session/ClaudeMessageHandler.java`
- `codes/jetbrains-cc-gui/src/main/java/com/github/claudecodegui/session/SessionMessageOrchestrator.java`

## 格式设计参考

下面给出一个可落地的格式设计。它不完全照搬任何一个项目，而是提取共同模式。

### 第一层：StreamEvent

StreamEvent 是实时事件，不是最终消息。

```ts
type StreamEvent =
  | StreamTextDelta
  | StreamReasoningDelta
  | StreamContentPart
  | StreamToolCallStarted
  | StreamToolCallDelta
  | StreamToolCallCompleted
  | StreamToolResult
  | StreamPermissionRequest
  | StreamUsageUpdate
  | StreamMessageCompleted
  | StreamTurnCompleted
  | StreamError;

type StreamEventBase = {
  eventId: string;
  sessionId: string;
  turnId: string;
  seq: number;
  at: string;
  provider?: string;
  model?: string;
  rawType?: string;
};

type StreamTextDelta = StreamEventBase & {
  type: 'assistant_text_delta';
  streamId: string;
  messageId?: string;
  delta: string;
};

type StreamReasoningDelta = StreamEventBase & {
  type: 'assistant_reasoning_delta';
  streamId: string;
  messageId?: string;
  delta: string;
  signature?: string;
};

type StreamToolCallStarted = StreamEventBase & {
  type: 'tool_call_started';
  messageId?: string;
  toolCallId: string;
  name: string;
  title?: string;
  kind?: 'read' | 'edit' | 'execute' | 'search' | 'mcp' | 'unknown';
  input?: unknown;
  inputText?: string;
};

type StreamToolCallDelta = StreamEventBase & {
  type: 'tool_call_delta';
  toolCallId: string;
  inputDelta?: string;
  status?: 'pending' | 'in_progress';
};

type StreamToolCallCompleted = StreamEventBase & {
  type: 'tool_call_completed';
  toolCallId: string;
  status: 'completed' | 'failed' | 'cancelled';
  input?: unknown;
  output?: unknown;
  error?: MessageError;
};

type StreamUsageUpdate = StreamEventBase & {
  type: 'usage_update';
  usage: TokenUsage;
};

type StreamMessageCompleted = StreamEventBase & {
  type: 'message_completed';
  messageId: string;
  finishReason?: string;
};

type StreamTurnCompleted = StreamEventBase & {
  type: 'turn_completed';
  stopReason?: string;
};

type StreamError = StreamEventBase & {
  type: 'error';
  messageId?: string;
  error: MessageError;
};
```

字段说明：

| 字段 | 必要性 | 说明 |
| --- | --- | --- |
| `eventId` | 必须 | 去重、审计、重放 |
| `sessionId` | 必须 | 会话归属 |
| `turnId` | 必须 | 一次用户输入到模型完成的完整回合 |
| `seq` | 必须 | 流内排序；断线重连后避免乱序 |
| `streamId` | 文本/思考必需 | 同一段文本流的聚合键，类似 AionUi 的 `msg_id` |
| `messageId` | 推荐 | 如果 assistant placeholder 已创建，则绑定最终消息 |
| `toolCallId` | 工具事件必需 | 工具调用聚合键 |
| `rawType` | 可选 | 记录 provider 原始事件名，方便调试 |

设计约束：

- `delta` 必须只表达增量，不应该塞完整消息，除非事件名叫 snapshot。
- 所有事件必须带 `seq`，否则断线重放会很麻烦。
- StreamEvent 可以被丢弃或压缩；最终历史不能依赖前端保留它。
- tool input 可能是 partial JSON，不要强行 parse 成对象。保留 `inputText`，完成时再写 `input`。

### 第二层：RuntimeMessageState

RuntimeMessageState 是内存中的聚合状态，用于实时渲染。

```ts
type RuntimeMessageState = {
  sessionId: string;
  turnId: string;
  messageId: string;
  status: 'creating' | 'streaming' | 'requires_action' | 'completed' | 'error' | 'cancelled';
  role: 'assistant';
  content: string;
  reasoning?: {
    content: string;
    signature?: string;
    durationMs?: number;
  };
  toolCalls: RuntimeToolCallState[];
  usage?: TokenUsage;
  error?: MessageError;
  updatedAt: string;
};

type RuntimeToolCallState = {
  toolCallId: string;
  name: string;
  status: 'pending' | 'in_progress' | 'requires_permission' | 'completed' | 'failed';
  inputText?: string;
  input?: unknown;
  output?: unknown;
  error?: MessageError;
};
```

这层可以参考：

- Claude Code UI 的 `realtimeMessages`。
- Proma 的 `streamingStatesAtom`。
- LobeHub 的 `StreamingHandler` 内部字段。
- AionUi 的 `composeMessage` 聚合结果。

推荐策略：

- 前端 UI 优先读 RuntimeMessageState。
- 服务端可选择每 500ms 到 2s 把 RuntimeMessageState patch 到 DB。
- 完成时必须生成一次权威 PersistedMessage。

### 第三层：PersistedMessage

PersistedMessage 是入库主表或 JSONL 的核心结构。

```ts
type PersistedMessage = {
  id: string;
  sessionId: string;
  topicId?: string | null;
  threadId?: string | null;
  agentId?: string | null;
  groupId?: string | null;
  turnId?: string;

  role: 'system' | 'user' | 'assistant' | 'tool';
  status: 'completed' | 'streaming' | 'requires_action' | 'error' | 'cancelled' | 'interrupted';

  content: string;
  blocks?: MessageBlock[];

  parentId?: string | null;
  toolCallId?: string | null;
  messageGroupId?: string | null;

  provider?: string | null;
  model?: string | null;
  reasoning?: PersistedReasoning | null;
  tools?: PersistedToolCall[];
  usage?: TokenUsage;
  error?: MessageError | null;
  metadata?: Record<string, unknown>;

  createdAt: string;
  updatedAt: string;
};

type MessageBlock =
  | { type: 'text'; text: string }
  | { type: 'image'; fileId?: string; url?: string; mimeType?: string }
  | { type: 'file'; fileId: string; name?: string; mimeType?: string }
  | { type: 'reasoning'; text: string; signature?: string }
  | { type: 'tool_call'; toolCallId: string }
  | { type: 'tool_result'; toolCallId: string; content: string; isError?: boolean };

type PersistedToolCall = {
  toolCallId: string;
  name: string;
  title?: string;
  kind?: string;
  status: 'pending' | 'in_progress' | 'requires_permission' | 'completed' | 'failed' | 'cancelled';
  input?: unknown;
  inputText?: string;
  output?: unknown;
  error?: MessageError | null;
  intervention?: {
    status: 'pending' | 'approved' | 'rejected' | 'aborted' | 'none';
    optionId?: string;
    reason?: string;
  };
};

type PersistedReasoning = {
  content?: string;
  signature?: string;
  durationMs?: number;
  redacted?: boolean;
};

type TokenUsage = {
  inputTokens?: number;
  outputTokens?: number;
  cacheReadTokens?: number;
  cacheCreationTokens?: number;
  totalTokens?: number;
  costUsd?: number;
};

type MessageError = {
  code?: string;
  message: string;
  retryable?: boolean;
  detail?: unknown;
};
```

字段设计理由：

| 字段 | 为什么需要 |
| --- | --- |
| `id` | 入库消息稳定主键 |
| `turnId` | 把一次用户请求内的 assistant、tool、permission、usage 关联起来 |
| `status` | 区分完成、流式中、报错、中断、等待用户操作 |
| `blocks` | 支持多模态、reasoning、tool result，而不把所有内容压成字符串 |
| `parentId` | 支持 assistant 回复关联 user 消息、tool 关联 assistant |
| `toolCallId` | 工具消息和 assistant message 中的 tool call 对齐 |
| `messageGroupId` | 支持多模型并行、压缩摘要、agent group |
| `metadata` | 扩展字段，但不要把关键查询字段只放在 metadata |

### 第四层：PersistedEventLog

如果需要回放、审计、debug，建议加事件日志表或 JSONL。

```ts
type PersistedEventLog = {
  id: string;
  sessionId: string;
  turnId?: string;
  seq: number;
  direction: 'inbound' | 'outbound' | 'internal';
  event: StreamEvent | unknown;
  raw?: unknown;
  createdAt: string;
};
```

保存原则：

- EventLog 是辅助数据，不作为默认消息查询源。
- 可以设置 retention，例如只保留最近 30 天或每个 session 最多 N MB。
- 如果保存 raw provider payload，需要注意敏感信息脱敏。

## 推荐数据流

推荐采用以下完整数据流：

```mermaid
flowchart LR
  U["用户输入"] --> PU["写入 user PersistedMessage"]
  PU --> PA["创建 assistant placeholder"]
  PA --> S["模型/Agent stream"]
  S --> SE["标准化 StreamEvent"]
  SE --> R["RuntimeMessageState 聚合"]
  R --> UI["实时 UI 渲染"]
  R --> P1["可选: 节流 patch DB"]
  S --> EL["可选: append EventLog"]
  SE --> Done["message_completed / turn_completed"]
  Done --> PF["最终 upsert PersistedMessage"]
  PF --> Reload["从 DB/JSONL 刷新历史"]
  Reload --> UI
```

明确执行规则：

1. 用户消息可以立即入库，因为它是稳定事实。
2. assistant 消息建议先创建 placeholder，获得稳定 `messageId`。
3. stream event 只更新 RuntimeMessageState。
4. 如果需要刷新后仍看到生成中状态，可以节流更新 placeholder 行。
5. 完成时用完整内容、reasoning、tools、usage、finishReason 更新同一条 assistant 消息。
6. 前端最终以 DB/JSONL 返回的历史为准，清空或去重 realtime 状态。

## 入库策略对比

| 策略 | 做法 | 优点 | 缺点 | 适用场景 |
| --- | --- | --- | --- | --- |
| 完成后一次入库 | stream 只在内存聚合，finish 后写完整消息 | DB 压力最低，历史最干净 | 刷新会丢失生成中内容 | 简单聊天、可接受生成中刷新丢失 |
| placeholder + 节流 update | 先插入 assistant 空消息，stream 中每隔一段时间更新同一行 | 刷新可恢复部分内容 | DB 写更多，需要处理并发覆盖 | 产品级聊天、多端同步 |
| event log + final snapshot | 每个事件 append log，完成后写最终消息 | 可回放、可 debug、审计好 | 存储量大，查询复杂 | Agent、工具链、企业审计 |
| JSONL append | 每条完整消息或 SDK message 一行 | 本地简单、迁移方便 | 查询和索引弱 | 桌面本地应用 |
| 替换式 snapshot | 始终覆盖最后一条 agent 消息或整个 state 文件 | 实现简单，适合终端输出 | 不保留细粒度过程 | PTY/终端类应用 |

推荐默认策略：`placeholder + RuntimeMessageState + finish upsert`。如果需要调试和回放，再加 `event log`。

## 设计时必须明确的边界

### 1. StreamEvent 的 ID 与 PersistedMessage 的 ID 不一定相同

可以有三种 ID：

- `eventId`：每个事件唯一。
- `streamId`：同一段文本或 reasoning 的聚合键。
- `messageId`：最终入库消息 ID。

不要只用一个 `id` 解决全部问题。AionUi 的 `id + msg_id`、Claude Code UI 的 `__streaming_<session>` 都是在处理这个问题。

### 2. 工具调用必须按 toolCallId 聚合

工具调用通常分阶段出现：

1. 模型开始吐出 tool call。
2. tool input 以 partial JSON 形式流式到达。
3. input 完整。
4. 等待权限。
5. 工具执行中。
6. 工具完成或失败。
7. tool result 回写。

所以工具不应该只保存在 assistant 文本里。至少要有：

- `toolCallId`
- `name`
- `status`
- `inputText`
- `input`
- `output`
- `error`
- `intervention`

### 3. Reasoning 要和普通文本分开

reasoning/thinking 可能有：

- 可展示文本。
- signature。
- redacted thinking。
- duration。
- 多模态 thinking block。

不要把 reasoning 混进 `content`。否则后续无法单独隐藏、折叠、回传签名或做安全策略。

### 4. 控制事件不要入库成普通消息

以下事件通常不应该直接进入 `messages` 主表：

- `status`
- `usage_update`
- `stream_start`
- `stream_end`
- `session_created`
- `available_commands_update`
- `config_option_update`

它们可以更新 session state、usage 表、event log，或只影响 UI。

### 5. 中断和错误要有明确状态

不要只在 content 后面拼接错误文本。建议：

```ts
status: 'error' | 'cancelled' | 'interrupted';
error: {
  code: 'MODEL_TIMEOUT',
  message: 'request timed out',
  retryable: true
};
```

这样历史、重试、UI 样式、统计才能准确处理。

### 6. 前端展示态和 DB 原始态要分开

LobeHub 的 `dbMessagesMap` / `messagesMap` 是一个好例子：

- `dbMessagesMap` 保存数据库原始消息。
- `messagesMap` 保存 parse 后的展示结构。

原因是展示层可能会做：

- assistant group。
- compressed group。
- tool group。
- branch/thread 展开。
- 临时 UI 状态。

这些不一定适合作为 DB 主结构。

## 推荐表结构

如果采用关系型数据库，可以这样拆：

```sql
messages (
  id text primary key,
  session_id text not null,
  topic_id text null,
  thread_id text null,
  agent_id text null,
  group_id text null,
  turn_id text null,
  role text not null,
  status text not null,
  content text null,
  blocks jsonb null,
  parent_id text null,
  message_group_id text null,
  provider text null,
  model text null,
  reasoning jsonb null,
  tools jsonb null,
  usage jsonb null,
  error jsonb null,
  metadata jsonb null,
  created_at timestamptz not null,
  updated_at timestamptz not null
);

message_tool_calls (
  id text primary key,
  message_id text not null,
  tool_call_id text not null,
  name text not null,
  status text not null,
  input jsonb null,
  input_text text null,
  output jsonb null,
  error jsonb null,
  intervention jsonb null,
  created_at timestamptz not null,
  updated_at timestamptz not null
);

message_events (
  id text primary key,
  session_id text not null,
  turn_id text null,
  seq integer not null,
  direction text not null,
  event jsonb not null,
  raw jsonb null,
  created_at timestamptz not null
);
```

索引建议：

```sql
create index messages_session_created_idx on messages (session_id, created_at);
create index messages_topic_created_idx on messages (topic_id, created_at);
create index messages_thread_created_idx on messages (thread_id, created_at);
create index messages_parent_idx on messages (parent_id);
create index messages_turn_idx on messages (turn_id);
create unique index message_tool_calls_tool_call_id_idx on message_tool_calls (tool_call_id);
create unique index message_events_session_seq_idx on message_events (session_id, seq);
```

如果用 JSONL：

- 一个 session 一个 JSONL 文件。
- 每行必须有 `type`。
- 完整消息用 `{type:'message', message: PersistedMessage}`。
- 事件日志用 `{type:'event', event: StreamEvent}`。
- meta 更新用 `{type:'meta', patch:{...}}`。

示例：

```jsonl
{"type":"message","message":{"id":"u1","role":"user","content":"hello","status":"completed"}}
{"type":"event","event":{"type":"assistant_text_delta","seq":1,"delta":"hi"}}
{"type":"message","message":{"id":"a1","role":"assistant","content":"hi","status":"completed"}}
```

读取时只把 `message` 类型作为历史主数据，`event` 类型用于调试或重放。

## 后续设计建议

如果我们要设计自己的消息目录，推荐这样定：

1. 明确两个主接口：
   - `appendStreamEvent(event: StreamEvent)`
   - `upsertPersistedMessage(message: PersistedMessage)`
2. 前端不要直接消费 raw provider event，必须先标准化为 StreamEvent。
3. StreamEvent 只负责实时变化，PersistedMessage 负责长期事实。
4. 每个 turn 创建稳定 `turnId`，每个 assistant placeholder 创建稳定 `messageId`。
5. 文本、thinking、tool 都用不同 event type，不要都塞进 `content_delta`。
6. 工具调用用 `toolCallId` 做全链路聚合键。
7. 入库采用 `placeholder + finish upsert`，同时允许节流 patch。
8. 如果需要 replay/debug，单独存 `message_events`，不要污染 `messages` 主表。
9. UI 使用 `persistedMessages + runtimeState` 合并渲染，服务端追平后清理 runtime。
10. DB 原始消息和 UI 展示消息分离，展示层可以派生 group、branch、tool panel。

最推荐的参考组合：

- 协议和事件分层：参考 acpx。
- 前端实时/历史合并：参考 Claude Code UI。
- 按 ID 合并 chunk、tool、plan：参考 AionUi。
- 产品级 DB schema 和乐观更新：参考 LobeHub。
- 本地 JSONL 方案：参考 Proma。
- 终端 screen snapshot 场景：参考 AgentAPI。

## 来源文档

- `docs/message-formats/README.md`
- `docs/message-formats/acpx.md`
- `docs/message-formats/claudecodeui.md`
- `docs/message-formats/AionUi.md`
- `docs/message-formats/lobehub.md`
- `docs/message-formats/proma.md`
- `docs/message-formats/agentapi.md`
- `docs/message-formats/jetbrains-cc-gui.md`
