# Proma — 消息格式

## 概述

Proma 拥有**两套并行的消息体系**：Chat 模式的 SSE 增量消息和 Agent 模式的 `SDKMessage` 流，通过 `AgentEventBus` 统一为 `AgentEvent` 事件类型发送到渲染进程。

## Agent 消息体系

### SDKMessage 原始类型（Claude Agent SDK）

```typescript
// 来自 @anthropic-ai/claude-agent-sdk
type SDKMessage =
  | SDKAssistantMessage   // assistant 完整回复 + content blocks
  | SDKUserMessage        // user 输入 + tool_result blocks
  | SDKResultMessage      // 最终结果 (subtype: success/error/max_turns/etc.)
  | SDKSystemMessage      // 系统通知 (init, resume 状态等)
  | SDKStreamEvent;       // 流式增量事件
```

### convertSDKMessage() — SDK → AgentEvent

```typescript
// agent-orchestrator.ts
function convertSDKMessage(msg: SDKMessage): AgentEvent[] {
  switch (msg.type) {
    case 'assistant':
      // 提取 text/reasoning/tool_use content blocks
      // → AgentTextEvent[] + AgentToolStartEvent[]
    case 'user':
      // 提取 tool_result blocks
      // → AgentToolResultEvent[]
    case 'result':
      // → AgentCompleteEvent (包含终止原因和统计)
    case 'system':
      // → AgentSystemEvent (初始化/恢复状态)
    case 'stream_event':
      // 流式增量: text_delta, reasoning_delta, tool_use_partial
      // → AgentStreamEvent (partial: true)
  }
}
```

### AgentEvent 事件类型

```typescript
type AgentEvent =
  | AgentTextEvent           // 文本增量或完整内容
  | AgentReasoningEvent      // 思考内容（含 signature）
  | AgentToolStartEvent      // tool_use 开始 { toolUseId, toolName, input }
  | AgentToolProgressEvent   // 工具执行进度更新
  | AgentToolResultEvent     // tool_result { toolUseId, content, isError }
  | AgentCompleteEvent       // 完成 { subtype, tokenUsage, cost }
  | AgentSystemEvent         // 系统（初始化/恢复状态）
  | AgentErrorEvent          // 错误 { code, message, retryable }
  | AgentPermissionEvent     // 权限请求（从 canUseTool 回调触发）
  | AgentAskUserEvent        // AskUserQuestion
  | AgentExitPlanEvent       // ExitPlanMode
  | AgentCompactingEvent;    // 上下文压缩通知
```

### AgentStreamState — 前端累积状态

```typescript
// agent-atoms.ts
interface AgentStreamState {
  running: boolean;
  content: string;                    // 累积文本
  reasoningContent: string;           // 累积思考内容
  thinkingBlocks: ThinkingBlock[];    // 带签名和结构化信息
  toolActivities: ToolActivity[];     // 所有工具调用（含时间/状态）
  model: string;
  tokenUsage: { input: number; output: number };
  cost: { total: number; currency: string };
  contextWindow: { used: number; total: number };
  compacting: boolean;
  retryState: { count: number; maxRetries: number };
}
```

## Chat 消息体系

### SSE 原始格式

```
data: {"type":"content_block_delta","delta":{"text":"Hello"}}

data: {"type":"content_block_start","content_block":{"type":"tool_use",...}}

data: [DONE]
```

### Chat Stream State

```typescript
// chat-atoms.ts
interface ChatStreamState {
  content: string;                // 累积文本
  reasoning: string;              // 思考文本
  thinkingBlocks: ThinkingBlock[];
  toolCalls: ChatToolCall[];
  stopReason: string | null;
  // 每个 conversation 独立的 AbortController
}
```

### SSE Line 解析（`sse-reader.ts`）

```typescript
// 通用 SSE 读取器
async function* readSSE(response: Response): AsyncGenerator<string> {
  const reader = response.body!.getReader();
  const decoder = new TextDecoder();
  let buffer = '';

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    buffer += decoder.decode(value, { stream: true });

    const lines = buffer.split('\n');
    buffer = lines.pop() || '';  // 保留未完成行

    for (const line of lines) {
      if (line.startsWith('data: ')) {
        const data = line.slice(6);
        if (data === '[DONE]') return;
        yield data;
      }
    }
  }
}
```

### Provider 适配器解析

```typescript
// parseSSELine() — 各适配器的差异化解析
AnthropicAdapter.parseSSELine(data):
  // content_block_start / content_block_delta / content_block_stop
  // message_start / message_delta / message_stop
  // → { type, delta?, content_block?, thinking?, signature? }

OpenAIAdapter.parseSSELine(data):
  // OpenAI chat.completion.chunk 格式
  // → { type, delta?, tool_calls? }

GoogleAdapter.parseSSELine(data):
  // Gemini GenerateContent 流式格式
  // → 转换为统一增量
```

## 两套消息体系的关系

```
Chat 模式:
  fetch SSE → sse-reader.ts → adapter.parseSSELine()
    → webContents.send(CHAT_IPC_CHANNELS.STREAM_EVENT)
      → useGlobalChatListeners → chat-atoms

Agent 模式:
  SDK query() → MessageStream (AsyncGenerator)
    → convertSDKMessage() → AgentEventBus middleware chain
      → webContents.send(AGENT_IPC_CHANNELS.STREAM_EVENT)
        → useGlobalAgentListeners → agent-atoms
```

**Chat 和 Agent 事件通道完全独立**，互不干扰，可同时运行。

## 流式策略

| 模式 | 流式策略 | 合并方式 |
|------|---------|---------|
| Chat | SSE 逐行解析 | 内容累积到 `ChatStreamState`，React state 更新触发重渲染 |
| Agent | SDK 异步迭代器 | `convertSDKMessage()` 拆分为独立事件 → Jotai `useStore()` 直接写入原子 |
| Agent 文本 | text_delta 增量 | `content += delta`，`unstable_batchedUpdates` 批量更新 |
| Agent 工具 | tool_start → tool_progress → tool_result | 按 toolUseId 更新 `ToolActivity[]` |

## 消息持久化格式

### Agent Session — JSONL

```jsonl
{"type":"user","message":{"role":"user","content":[{"type":"text","text":"..."}]}}
{"type":"assistant","message":{"role":"assistant","content":[{"type":"text","text":"..."},{"type":"tool_use",...}]}}
{"type":"result","subtype":"success","tokenUsage":{...}}
```

### Chat Conversation — JSONL

```jsonl
{"role":"user","content":"..."}
{"role":"assistant","content":"...","toolCalls":[...]}
```
