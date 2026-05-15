# acpx — 消息格式

## 概述

acpx 有两层消息模型：底层 ACP 协议的 JSON-RPC 消息，以及高层 `AcpRuntimeEvent` 运行时事件。所有消息通过 TypeScript 联合类型严格定义。

## AcpRuntimeEvent 运行时事件（5 种）

```typescript
export type AcpRuntimeEvent =
  | {
      type: 'text_delta';         // 文本增量
      text: string;
      stream?: 'output' | 'thought';
      tag?: AcpSessionUpdateTag;
    }
  | {
      type: 'status';             // 状态更新
      text: string;
      tag?: AcpSessionUpdateTag;
      used?: number;              // 已用 token
      size?: number;              // 上下文窗口大小
    }
  | {
      type: 'tool_call';          // 工具调用
      text: string;
      tag?: AcpSessionUpdateTag;
      toolCallId?: string;
      status?: string;
      title?: string;
      kind?: ToolKind;
      locations?: ToolCallLocation[];
      rawInput?: unknown;
      rawOutput?: unknown;
      content?: ToolCallContent[];
    }
  | {
      type: 'done';               // 完成
      stopReason?: string;
    }
  | {
      type: 'error';              // 错误
      message: string;
      code?: string;
      detailCode?: string;
      retryable?: boolean;
    };
```

## AcpSessionUpdateTag（10 种 ACP 标签）

| 标签 | 说明 |
|------|------|
| `agent_message_chunk` | Agent 消息文本块（流式） |
| `agent_thought_chunk` | Agent 思考块（流式） |
| `tool_call` | 工具调用开始 |
| `tool_call_update` | 工具调用状态更新 |
| `usage_update` | token 用量更新 |
| `available_commands_update` | 可用命令列表更新 |
| `current_mode_update` | 当前模式更新 |
| `config_option_update` | 配置选项更新 |
| `session_info_update` | 会话信息更新 |
| `plan` | 计划条目更新 |

## SessionMessage 持久化消息（3 种变体）

```typescript
export type SessionMessage =
  | { User: SessionUserMessage }
  | { Agent: SessionAgentMessage }
  | 'Resume';                          // 恢复标记
```

### SessionUserMessage

```typescript
export type SessionUserMessage = {
  id: string;
  content: SessionUserContent[];       // Text | Mention | Image 联合数组
};
```

### SessionAgentMessage

```typescript
export type SessionAgentMessage = {
  content: SessionAgentContent[];      // Text | Thinking | RedactedThinking | ToolUse
  tool_results: Record<string, SessionToolResult>;
  reasoning_details?: unknown;
};

export type SessionToolUse = {
  id: string;
  name: string;
  raw_input: string;
  input: unknown;
  is_input_complete: boolean;
  thought_signature?: string | null;
};
```

## SessionRecord 完整会话记录

```typescript
export type SessionRecord = {
  schema: 'acpx.session.v1';
  acpxRecordId: string;
  acpSessionId: string;
  agentSessionId?: string;
  agentCommand: string;
  cwd: string;
  name?: string;
  createdAt: string;
  lastUsedAt: string;
  lastSeq: number;
  eventLog: SessionEventLog;
  closed?: boolean;
  closedAt?: string;
  pid?: number;
  messages: SessionMessage[];
  cumulative_token_usage: SessionTokenUsage;
  request_token_usage: Record<string, SessionTokenUsage>;
};

export type SessionTokenUsage = {
  input_tokens?: number;
  output_tokens?: number;
  cache_creation_input_tokens?: number;
  cache_read_input_tokens?: number;
};
```

## 流式处理

通过 `AcpRuntimeTurn.events: AsyncIterable<AcpRuntimeEvent>` 异步迭代器提供，每个 turn 产生多个事件流。

## ACP JSON-RPC 方法（协议层）

| 方法 | 方向 | 说明 |
|------|------|------|
| `initialize` | 请求/响应 | 协商协议版本和能力 |
| `session/new` | 请求/响应 | 创建新会话 |
| `session/load` | 请求/响应 | 恢复已有会话 |
| `session/prompt` | 请求/响应 | 发送用户提示 |
| `session/cancel` | 请求/响应 | 取消当前回合 |
| `session/update` | 通知 | Agent 推送流式更新 |
