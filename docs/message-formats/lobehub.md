# LobeHub — 消息格式

## 概述

LobeHub 有三层消息模型：LLM API 层（`ChatStreamPayload`）、Agent 运行时层（`AgentEvent`）、UI 渲染层（`UIChatMessage`）。

## ChatStreamPayload（LLM 调用请求）

```typescript
export interface ChatStreamPayload {
  model: string;
  messages: OpenAIChatMessage[];         // 兼容 OpenAI 格式
  provider?: string;
  stream?: boolean;                       // 流式开关
  temperature?: number;
  top_p?: number;
  max_tokens?: number;
  tools?: ChatCompletionTool[];
  tool_choice?: string;
  thinking?: { budget_tokens: number; type?: 'enabled' | 'disabled' | 'adaptive' };
  enabledSearch?: boolean;
  response_format?: ChatResponseFormat;
  frequency_penalty?: number;
  presence_penalty?: number;
  // ... 更多可选字段
}
```

## OpenAIChatMessage（统一消息格式）

```typescript
export interface OpenAIChatMessage {
  content: string | UserMessageContentPart[];
  role: LLMRoleType;           // 'user' | 'system' | 'assistant' | 'function' | 'tool'
  name?: string;
  tool_call_id?: string;
  tool_calls?: MessageToolCall[];
  reasoning?: { content?: string; duration?: number; };
}

// 多模态内容
type UserMessageContentPart =
  | { type: 'text'; text: string }
  | { type: 'image_url'; image_url: { url: string; detail?: 'auto' | 'low' | 'high' } }
  | { type: 'video_url'; video_url: { url: string } };
```

## ChatStreamCallbacks（流式回调）

```typescript
export interface ChatStreamCallbacks {
  onStart?: () => Promise<void> | void;
  onText?: (content: string) => Promise<void> | void;
  onThinking?: (content: string) => Promise<void> | void;
  onToolsCalling?: (data: { chunk; toolsCalling }) => Promise<void> | void;
  onUsage?: (usage: ModelTokensUsage) => Promise<void> | void;
  onError?: (error: any) => Promise<void> | void;
  onFinal?: (data: OnFinishData) => Promise<void> | void;
  onContentPart?: (data: ContentPartData) => Promise<void> | void;
  onReasoningPart?: (data: ContentPartData) => Promise<void> | void;
  onBase64Image?: (data: { image; images }) => Promise<void> | void;
}
```

## AgentEvent（15 种运行时事件）

| type | 说明 |
|------|------|
| `init` | 初始化 |
| `llm_start` | LLM 调用开始 |
| `llm_stream` | LLM 流式块 |
| `llm_result` | LLM 调用结束 |
| `tool_pending` | 工具调用待处理 |
| `tool_result` | 工具执行结果 |
| `done` | 完成 |
| `error` | 错误 |
| `human_approve_required` | 需要人工审批 |
| `human_prompt_required` | 需要人工输入 |
| `human_select_required` | 需要人工选择 |
| `interrupted` | 中断 |
| `resumed` | 恢复 |
| `compression_complete` | 上下文压缩完成 |
| `compression_error` | 上下文压缩失败 |

## UIChatMessage（UI 完整消息）

```typescript
export interface UIChatMessage {
  id: string;
  role: UIMessageRoleType;          // 12 种角色
  content: string;
  createdAt: number;
  updatedAt: number;
  parentId?: string;
  sessionId?: string;
  model?: string | null;
  provider?: string | null;
  error?: ChatMessageError | null;

  // 子结构
  children?: AssistantContentBlock[];  // 子消息块（tool 分组）
  tasks?: UIChatMessage[];             // 子任务
  members?: UIChatMessage[];           // 成员消息（Agent 委员会）

  // 工具
  tools?: ChatToolPayload[];
  tool_call_id?: string;

  // 性能
  performance?: ModelPerformance;
  usage?: ModelUsage;
  reasoning?: ModelReasoning | null;

  // 扩展
  extra?: ChatMessageExtra;
  meta?: MessageMetadata | null;
}
```

## UIMessageRoleType（12 种 UI 角色）

```typescript
export type UIMessageRoleType =
  | 'user' | 'system' | 'assistant' | 'tool'
  | 'task' | 'tasks' | 'groupTasks'
  | 'supervisor' | 'assistantGroup' | 'agentCouncil'
  | 'compressedGroup' | 'compareGroup';
```

## ChatToolPayload（工具调用）

```typescript
interface ChatToolPayload {
  apiName: string;               // 工具 API 名（如 'Bash', 'Read'）
  arguments: string;             // JSON 字符串参数
  id: string;                    // 工具调用 ID
  identifier: string;            // 提供者标识（如 'claude-code'）
  type: LobeToolRenderType;      // 'mcp' | 'default' | 'markdown' | 'standalone'
  executor?: 'client' | 'server';
  source?: ToolSource;           // 'builtin' | 'client' | 'mcp' | 'klavis' | 'lobehubSkill'
  intervention?: ToolIntervention; // { status: 'pending'|'approved'|'rejected'|'aborted'|'none' }
  thoughtSignature?: string;
}
```

## FinishReason（11 种完成原因）

```typescript
export type FinishReason =
  | 'completed' | 'user_requested' | 'user_aborted'
  | 'max_steps_exceeded' | 'max_steps_completed'
  | 'cost_limit_exceeded' | 'timeout'
  | 'agent_decision' | 'queued_message_interrupt'
  | 'error_recovery' | 'system_shutdown';
```
