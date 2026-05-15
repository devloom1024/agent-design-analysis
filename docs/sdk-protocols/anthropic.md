# Anthropic Messages API — 消息接口定义

## 概述

Anthropic Messages API 是 Anthropic Claude 系列模型的统一消息接口。与 OpenAI 的核心差异在于：Content 始终是强类型 `ContentBlock[]` 数组、内置 Extended Thinking 机制、流式事件使用命名 SSE 事件类型而非 `[DONE]` 哨兵。

## Messages API

### 请求

```
POST https://api.anthropic.com/v1/messages
```

#### MessageCreateParamsBase

```typescript
interface MessageCreateParamsBase {
  // === 核心参数 ===
  model: Model;                            // 模型 ID
  messages: MessageParam[];                // 对话历史
  max_tokens: number;                      // 最大输出 token 数（必填）

  // === 系统提示 ===
  system?: string | TextBlockParam[];       // 顶层参数，非消息角色

  // === 工具 ===
  tools?: ToolUnion[];                      // 可用工具列表
  tool_choice?: ToolChoice;                 // 工具选择策略

  // === 思考 ===
  thinking?: ThinkingConfigParam;           // Extended Thinking 配置

  // === 输出控制 ===
  stop_sequences?: string[];               // 自定义停止序列
  output_config?: OutputConfig;             // 结构化输出配置

  // === 流式 ===
  stream?: boolean;                        // 是否流式

  // === 缓存 ===
  cache_control?: CacheControlEphemeral;   // 提示缓存控制

  // === 服务 ===
  service_tier?: 'auto' | 'standard_only';
  inference_geo?: string;                  // 推理区域
  container?: string;                      // 容器标识（代码执行复用）
  metadata?: Metadata;

  // === 已废弃 ===
  temperature?: number;                    // @deprecated
  top_k?: number;                          // @deprecated
  top_p?: number;                          // @deprecated
}
```

### 消息参数（MessageParam）

```typescript
interface MessageParam {
  role: 'user' | 'assistant';
  content: string | ContentBlockParam[];
}
```

**特点**：只有 `user` 和 `assistant` 两种角色。System prompt 作为顶层参数传递，不在 messages 数组中。`content` 可以是纯字符串简写（等价于 `[{ type: 'text', text: '...' }]`）。

### ContentBlockParam（输入侧）

```typescript
type ContentBlockParam =
  | TextBlockParam            // { type: 'text', text: string, cache_control? }
  | ImageBlockParam           // { type: 'image', source: Base64ImageSource | URLImageSource }
  | DocumentBlockParam        // { type: 'document', source: PDF | PlainText | URL }
  | ToolUseBlockParam         // { type: 'tool_use', id, name, input }
  | ToolResultBlockParam      // { type: 'tool_result', tool_use_id, content?, is_error? }
  | ThinkingBlockParam        // { type: 'thinking', thinking, signature }
  | RedactedThinkingBlockParam
  | SearchResultBlockParam
  | ServerToolUseBlockParam
  | WebSearchToolResultBlockParam
  | WebFetchToolResultBlockParam
  | CodeExecutionToolResultBlockParam
  | BashCodeExecutionToolResultBlockParam
  | TextEditorCodeExecutionToolResultBlockParam
  | ToolSearchToolResultBlockParam
  | ContainerUploadBlockParam;
```

#### 关键 Block 类型详情

**TextBlockParam**：
```typescript
interface TextBlockParam {
  type: 'text';
  text: string;
  cache_control?: CacheControlEphemeral | null;
  citations?: TextCitationParam[] | null;
}
```

**ImageBlockParam**：
```typescript
interface ImageBlockParam {
  type: 'image';
  source: Base64ImageSource | URLImageSource;
  cache_control?: CacheControlEphemeral | null;
}
// Base64ImageSource: { data: string; media_type: 'image/jpeg'|'image/png'|...; type: 'base64' }
// URLImageSource: { url: string; type: 'url' }
```

**DocumentBlockParam**：
```typescript
interface DocumentBlockParam {
  type: 'document';
  source: Base64PDFSource | PlainTextSource | ContentBlockSource | URLPDFSource;
  title?: string;
  context?: string;
  citations?: CitationsConfigParam;
  cache_control?: CacheControlEphemeral | null;
}
```

**ToolUseBlockParam**（工具调用出现在对话中）：
```typescript
interface ToolUseBlockParam {
  type: 'tool_use';
  id: string;
  name: string;
  input: unknown;
  cache_control?: CacheControlEphemeral | null;
}
```

**ToolResultBlockParam**（工具执行结果）：
```typescript
interface ToolResultBlockParam {
  type: 'tool_result';
  tool_use_id: string;
  content?: string | Array<TextBlockParam | ImageBlockParam | SearchResultBlockParam | DocumentBlockParam>;
  is_error?: boolean;
  cache_control?: CacheControlEphemeral | null;
}
```

### 工具定义（ToolUnion）

```typescript
type ToolUnion =
  | Tool                          // 自定义工具
  | ToolBash20250124              // Bash 执行
  | CodeExecutionTool20250522     // 代码执行
  | CodeExecutionTool20250825
  | CodeExecutionTool20260120
  | MemoryTool20250818            // 记忆
  | ToolTextEditor20250124        // 文本编辑器
  | ToolTextEditor20250429
  | ToolTextEditor20250728
  | WebSearchTool20250305         // 网页搜索
  | WebFetchTool20250910          // 网页抓取
  | WebSearchTool20260209
  | WebFetchTool20260209
  | WebFetchTool20260309
  | ToolSearchToolBm25_20251119   // 工具搜索
  | ToolSearchToolRegex20251119;
```

**注意**：Anthropic 的工具类型带有版本后缀（日期），反映了其 API 的阶段性演进方式。

#### 自定义工具（Tool）

```typescript
interface Tool {
  name: string;
  type?: 'custom';
  description?: string;
  input_schema: {
    type: 'object';
    properties?: unknown;
    required?: string[];
    [k: string]: unknown;
  };
  strict?: boolean;
  defer_loading?: boolean;          // 延迟加载，不出现在初始 system prompt
  eager_input_streaming?: boolean;  // 启用工具的输入流式传输
  input_examples?: Array<Record<string, unknown>>;
  allowed_callers?: Array<'direct' | 'code_execution_20250825' | 'code_execution_20260120'>;
  cache_control?: CacheControlEphemeral | null;
}
```

### 工具选择策略（ToolChoice）

```typescript
type ToolChoice =
  | { type: 'auto'; disable_parallel_tool_use?: boolean }    // 自动决定
  | { type: 'any'; disable_parallel_tool_use?: boolean }     // 必须调用（任意工具）
  | { type: 'tool'; name: string; disable_parallel_tool_use?: boolean }  // 指定工具
  | { type: 'none' };  // 不调用工具
```

### Extended Thinking 配置

```typescript
type ThinkingConfigParam =
  | {
      type: 'enabled';
      budget_tokens: number;       // 思考预算（≥1024）
      display?: 'summarized' | 'omitted';
    }
  | {
      type: 'disabled';
    }
  | {
      type: 'adaptive';
      display?: 'summarized' | 'omitted';
    };
```

**关键机制**：`budget_tokens` 要求 ≥1024 且 < `max_tokens`。思考内容以 `ThinkingBlock`（含加密 `signature`）形式返回，确保思考链的完整性和可验证性。

### 结构化输出

```typescript
interface OutputConfig {
  effort?: 'low' | 'medium' | 'high' | 'xhigh' | 'max' | null;
  format?: {
    type: 'json_schema';
    json_schema: unknown;
  } | null;
}
```

### 提示缓存

```typescript
interface CacheControlEphemeral {
  type: 'ephemeral';
  ttl?: '5m' | '1h';  // 默认 5 分钟
}
```

在 `ContentBlock` 上标记 `cache_control` 断点，实现多轮对话间的缓存复用。

---

### 响应（Message）

```typescript
interface Message {
  id: string;
  type: 'message';
  role: 'assistant';
  model: string;
  content: ContentBlock[];           // 核心输出
  stop_reason: StopReason | null;    // 停止原因
  stop_sequence: string | null;      // 命中的自定义停止序列
  stop_details: RefusalStopDetails | null;  // 安全拒绝详情
  container: Container | null;       // 代码执行容器信息
  usage: Usage;                      // Token 用量
}
```

### ContentBlock（输出侧）

```typescript
type ContentBlock =
  | TextBlock                    // { type: 'text', text: string, citations?: TextCitation[] }
  | ThinkingBlock                // { type: 'thinking', thinking: string, signature: string }
  | RedactedThinkingBlock        // { type: 'redacted_thinking', data: string }
  | ToolUseBlock                 // { type: 'tool_use', id: string, name: string, input: unknown }
  | ServerToolUseBlock           // 服务端工具调用（web_search/code_execution 等）
  | WebSearchToolResultBlock     // 网页搜索结果
  | WebFetchToolResultBlock      // 网页抓取结果
  | CodeExecutionToolResultBlock // 代码执行结果
  | BashCodeExecutionToolResultBlock
  | TextEditorCodeExecutionToolResultBlock
  | ToolSearchToolResultBlock
  | ContainerUploadBlock;        // 容器文件上传
```

#### 核心 Block 详情

**TextBlock** — 带引用的文本输出：
```typescript
interface TextBlock {
  type: 'text';
  text: string;
  citations: TextCitation[] | null;
}
```

**ThinkingBlock** — 思考过程（带签名验证）：
```typescript
interface ThinkingBlock {
  type: 'thinking';
  thinking: string;
  signature: string;     // 思考内容的加密签名，确保完整性
}
```

**RedactedThinkingBlock** — 因安全策略被隐藏的思考：
```typescript
interface RedactedThinkingBlock {
  type: 'redacted_thinking';
  data: string;
}
```

**ToolUseBlock** — 模型发起的工具调用：
```typescript
interface ToolUseBlock {
  type: 'tool_use';
  id: string;
  name: string;
  input: unknown;
  caller: DirectCaller | ServerToolCaller | ServerToolCaller20260120;  // 调用者信息
}
```

### StopReason

```typescript
type StopReason =
  | 'end_turn'       // 自然结束
  | 'max_tokens'     // 达到 max_tokens
  | 'stop_sequence'  // 命中自定义停止序列
  | 'tool_use'       // 发起工具调用
  | 'pause_turn'     // 长任务暂停
  | 'refusal';       // 安全策略拦截
```

### Token 用量（Usage）

```typescript
interface Usage {
  input_tokens: number;
  output_tokens: number;
  cache_creation_input_tokens: number | null;  // 新建缓存消耗
  cache_read_input_tokens: number | null;      // 缓存命中节省
  server_tool_use: {
    web_search_requests: number;
    web_fetch_requests: number;
  } | null;
  service_tier: 'standard' | 'priority' | 'batch' | null;
  inference_geo: string | null;
}
```

---

### 流式响应（SSE 命名事件）

Anthropic 流式响应使用 **命名 SSE 事件**，每个事件有独立的 `event:` 类型，不同于 OpenAI 的 `data:` + `[DONE]` 模式。

#### 事件类型总览

```typescript
type RawMessageStreamEvent =
  | RawMessageStartEvent         // event: message_start
  | RawContentBlockStartEvent    // event: content_block_start
  | RawContentBlockDeltaEvent    // event: content_block_delta
  | RawContentBlockStopEvent     // event: content_block_stop
  | RawMessageDeltaEvent         // event: message_delta
  | RawMessageStopEvent;         // event: message_stop
```

#### message_start — 流开始

```typescript
{
  type: 'message_start';
  message: Message;  // 完整 Message 对象（stop_reason 此时为 null）
}
```

#### content_block_start — 内容块开始

```typescript
{
  type: 'content_block_start';
  index: number;
  content_block: ContentBlock;  // 完整内容块（如 ToolUseBlock 在开始时就知道 name）
}
```

#### content_block_delta — 内容增量

```typescript
type RawContentBlockDelta =
  | TextDelta          // { type: 'text_delta', text: string }
  | InputJSONDelta     // { type: 'input_json_delta', partial_json: string }
  | CitationsDelta     // { type: 'citations_delta', citation: Citation* }
  | ThinkingDelta      // { type: 'thinking_delta', thinking: string }
  | SignatureDelta;    // { type: 'signature_delta', signature: string }
```

**五种 Delta 类型及其用途**：

| Delta 类型 | 字段 | 用途 |
|-----------|------|------|
| `text_delta` | `text: string` | 普通文本增量 |
| `input_json_delta` | `partial_json: string` | 工具参数 JSON 增量 |
| `thinking_delta` | `thinking: string` | 思考过程增量 |
| `signature_delta` | `signature: string` | 思考签名增量 |
| `citations_delta` | `citation: Citation*` | 引用标注增量 |

#### content_block_stop — 内容块结束

```typescript
{
  type: 'content_block_stop';
  index: number;
}
```

#### message_delta — 消息级增量

```typescript
{
  type: 'message_delta';
  delta: {
    stop_reason: StopReason | null;
    stop_sequence: string | null;
    stop_details: RefusalStopDetails | null;
    container: Container | null;
  };
  usage: MessageDeltaUsage;  // 增量 token 统计
}
```

#### message_stop — 流结束

```typescript
{
  type: 'message_stop';
}
```

#### 流式时序

```
message_start              → 开始
  content_block_start 0    → 第 1 个 ContentBlock 开始（如 text）
    content_block_delta 0  → text_delta: "Hello"
    content_block_delta 0  → text_delta: " World"
  content_block_stop 0     → 第 1 个 ContentBlock 结束
  content_block_start 1    → 第 2 个 ContentBlock 开始（如 tool_use）
    content_block_delta 1  → input_json_delta: '{"query":'
    content_block_delta 1  → input_json_delta: '"..."}'
  content_block_stop 1     → 第 2 个 ContentBlock 结束
message_delta              → stop_reason + usage
message_stop               → 结束
```

---

## 与 OpenAI 的关键差异

| 特性 | OpenAI | Anthropic |
|------|--------|-----------|
| Content | 简单字符串（默认） | 始终 `ContentBlock[]` 数组 |
| System Prompt | `role: 'system'` 消息 | 顶层 `system` 参数 |
| 工具调用 | `tool_calls[]` 独立字段 | `content` 中的 `ToolUseBlock` |
| Thinking | `reasoning_effort`（推理模型） | `ThinkingBlock` + `signature`（专有机制） |
| 流式标识 | `object: 'chat.completion.chunk'` | `type: 'content_block_delta'` |
| 流式结束 | `[DONE]` 哨兵 | `message_stop` 事件 |
| 工具版本 | 无版本概念 | 每个工具带版本后缀（如 `20260120`） |
| 并行工具 | `parallel_tool_calls: boolean` | `disable_parallel_tool_use: boolean`（每个 tool_choice 级别） |
