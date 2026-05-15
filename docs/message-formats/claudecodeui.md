# Claude Code UI — 消息格式

## 概述

NormalizedMessage 是 Claude Code UI 的核心概念——所有 provider 的原始输出都归一化到此格式。

## NormalizedMessage 完整定义

```typescript
export type NormalizedMessage = {
  id: string;                        // 唯一 ID
  sessionId: string;                 // 会话 ID
  timestamp: string;                 // ISO 时间戳
  provider: LLMProvider;             // 'claude' | 'codex' | 'gemini' | 'cursor'

  kind: MessageKind;                 // 核心判别字段（14 种）
  role?: 'user' | 'assistant';

  content?: string;                  // 文本内容
  text?: string;                     // 备用文本字段
  displayText?: string;              // UI 显示文本

  // 工具调用相关
  toolName?: string;
  toolInput?: unknown;
  toolId?: string;
  toolResult?: { content?: string; isError?: boolean; toolUseResult?: unknown };

  // 权限/交互
  requestId?: string;
  input?: unknown;
  context?: unknown;
  reason?: string;
  canInterrupt?: boolean;

  // 会话
  newSessionId?: string;
  status?: string;
  summary?: string;
  tokenBudget?: unknown;
  subagentTools?: unknown;

  // 本地命令
  commandName?: string;
  commandMessage?: string;
  commandArgs?: string;
  isLocalCommand?: boolean;
  isLocalCommandStdout?: boolean;
  isCompactSummary?: boolean;

  tokens?: number;
  isError?: boolean;
  images?: unknown;
  sequence?: number;
  rowid?: number;

  [key: string]: unknown;            // 可扩展
};
```

## MessageKind 枚举（14 种）

| 值 | 说明 |
|---|------|
| `text` | 纯文本消息块 |
| `tool_use` | 工具调用（assistant 发起） |
| `tool_result` | 工具执行结果 |
| `thinking` | 推理/思考内容块 |
| `stream_delta` | 流式增量文本 |
| `stream_end` | 流式块结束 |
| `error` | 错误事件 |
| `complete` | 会话/请求完成 |
| `status` | 状态更新（如 token 预算） |
| `permission_request` | 权限请求 |
| `permission_cancelled` | 权限请求取消/超时 |
| `session_created` | 新会话创建 |
| `interactive_prompt` | 交互式提示（等待用户输入） |
| `task_notification` | 任务通知 |

## 流式消息时序

```
session_created (首次)
  → stream_delta (0..N 次)
  → stream_end (0..N 次, content_block_stop)
  → [status: token_budget]
  → complete 或 error
```

## 消息转换规则（ClaudeSessionsProvider）

| 输入 raw.type | 输出 MessageKind |
|--------------|-----------------|
| `content_block_delta` + `delta.text` | `stream_delta` |
| `content_block_stop` | `stream_end` |
| user role + tool_result 内容 | `tool_result` |
| user role + text 内容 | `text` (role: 'user') |
| assistant role + text block | `text` (role: 'assistant') |
| assistant role + tool_use block | `tool_use` |
| assistant role + thinking block | `thinking` |
| SDK permission_request 事件 | `permission_request` |
| SDK 超时取消 | `permission_cancelled` |
| SDK result 事件 | `status` (token_budget) |
| 流结束 | `complete` |

## FetchHistoryResult

```typescript
export type FetchHistoryResult = {
  messages: NormalizedMessage[];
  total: number;
  hasMore: boolean;
  offset: number;
  limit: number | null;
  tokenUsage?: unknown;
};
```
