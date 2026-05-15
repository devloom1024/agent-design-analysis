# OpenAI API — 消息接口定义

## 概述

OpenAI Node.js SDK 提供两套核心消息 API：**Chat Completions**（v1/chat/completions）和 **Responses**（新 API）。Chat Completions 是目前应用最广泛的 LLM API 格式，被大量第三方项目作为标准格式参考。

## Chat Completions API

### 请求

```
POST https://api.openai.com/v1/chat/completions
```

#### ChatCompletionCreateParamsBase

```typescript
interface ChatCompletionCreateParamsBase {
  messages: ChatCompletionMessageParam[];  // 对话消息数组
  model: string;                           // 模型 ID，如 'gpt-4o'、'o3'

  // 输出控制
  max_completion_tokens?: number;          // 最大生成 token 数
  temperature?: number;                    // 0-2，采样温度
  top_p?: number;                          // 核采样
  n?: number;                              // 生成 N 条回复（默认 1）
  stop?: string | string[];                // 停止序列（最多 4 个）

  // 工具调用
  tools?: ChatCompletionTool[];            // 可用工具列表
  tool_choice?: ChatCompletionToolChoiceOption;  // 'none' | 'auto' | 'required' | 指定工具
  parallel_tool_calls?: boolean;           // 是否允许并行调用

  // 结构化输出
  response_format?: ResponseFormatText | ResponseFormatJSONSchema | ResponseFormatJSONObject;

  // 推理模型
  reasoning_effort?: 'none' | 'minimal' | 'low' | 'medium' | 'high' | 'xhigh';

  // 多模态
  modalities?: ('text' | 'audio')[];       // 输出模态
  audio?: ChatCompletionAudioParam;        // 音频输出配置

  // 流式
  stream?: boolean;                        // 是否流式
  stream_options?: ChatCompletionStreamOptions;  // { include_usage: boolean }

  // 缓存
  prompt_cache_key?: string;
  prompt_cache_retention?: 'in_memory' | '24h';

  // 其他
  frequency_penalty?: number;              // -2.0 ~ 2.0
  presence_penalty?: number;               // -2.0 ~ 2.0
  logit_bias?: Record<string, number>;      // token 偏置
  logprobs?: boolean;
  top_logprobs?: number;                   // 0-20
  seed?: number;                           // 确定性采样
  service_tier?: 'auto' | 'default' | 'flex' | 'scale' | 'priority';
  safety_identifier?: string;
  metadata?: Record<string, string>;       // 最多 16 个键值对
  web_search_options?: WebSearchOptions;
  verbosity?: 'low' | 'medium' | 'high';
}
```

**流式与非流式**通过 `stream` 字段区分：
- `stream?: false | null` → `ChatCompletionCreateParamsNonStreaming`
- `stream: true` → `ChatCompletionCreateParamsStreaming`

### 消息角色（ChatCompletionMessageParam）

```typescript
type ChatCompletionMessageParam =
  | ChatCompletionDeveloperMessageParam   // role: 'developer'
  | ChatCompletionSystemMessageParam      // role: 'system'
  | ChatCompletionUserMessageParam        // role: 'user'
  | ChatCompletionAssistantMessageParam   // role: 'assistant'
  | ChatCompletionToolMessageParam        // role: 'tool'
  | ChatCompletionFunctionMessageParam;   // role: 'function' (已废弃)
```

#### System / Developer Message

```typescript
interface ChatCompletionSystemMessageParam {
  role: 'system';
  content: string | ChatCompletionContentPartText[];  // 仅文本
  name?: string;
}
// Developer message 与 system 完全相同的结构，区别在于语义定位
```

#### User Message

```typescript
interface ChatCompletionUserMessageParam {
  role: 'user';
  content: string | ChatCompletionContentPart[];  // 支持多模态
  name?: string;
}
```

#### Assistant Message

```typescript
interface ChatCompletionAssistantMessageParam {
  role: 'assistant';
  content?: string | Array<ChatCompletionContentPartText | ChatCompletionContentPartRefusal> | null;
  tool_calls?: ChatCompletionMessageToolCall[];
  refusal?: string | null;
  name?: string;
  audio?: { id: string };
}
```

#### Tool Message

```typescript
interface ChatCompletionToolMessageParam {
  role: 'tool';
  content: string | ChatCompletionContentPartText[];
  tool_call_id: string;  // 关联哪个 tool_call
}
```

### Content Part 类型

```typescript
type ChatCompletionContentPart =
  | { type: 'text'; text: string }
  | { type: 'image_url'; image_url: { url: string; detail?: 'auto' | 'low' | 'high' } }
  | { type: 'input_audio'; input_audio: { data: string; format: 'wav' | 'mp3' } }
  | { type: 'file'; file: { file_data?: string; file_id?: string; filename?: string } };
```

### 工具定义

```typescript
// 函数工具
interface ChatCompletionFunctionTool {
  type: 'function';
  function: {
    name: string;
    description?: string;
    parameters?: Record<string, unknown>;  // JSON Schema
    strict?: boolean;
  };
}

// 自定义工具
interface ChatCompletionCustomTool {
  type: 'custom';
  custom: {
    name: string;
    description?: string;
    format?: { type: 'text' } | { type: 'grammar'; grammar: { definition: string; syntax: 'lark' | 'regex' } };
  };
}

type ChatCompletionTool = ChatCompletionFunctionTool | ChatCompletionCustomTool;
```

### 工具选择策略

```typescript
type ChatCompletionToolChoiceOption =
  | 'none'       // 不调用工具
  | 'auto'       // 模型自行决定
  | 'required'   // 必须调用工具
  | { type: 'function'; function: { name: string } }    // 指定函数
  | { type: 'custom'; custom: { name: string } }         // 指定自定义工具
  | { type: 'allowed_tools'; allowed_tools: { mode: 'auto' | 'required'; tools: Array<{}> } };
```

### 结构化输出

```typescript
type ResponseFormat =
  | { type: 'text' }                                              // 纯文本
  | { type: 'json_object' }                                       // 任意 JSON
  | { type: 'json_schema'; json_schema: { name: string; description?: string; schema?: Record<string, unknown>; strict?: boolean } };
```

---

### 响应（ChatCompletion）

```typescript
interface ChatCompletion {
  id: string;                              // 唯一 ID
  object: 'chat.completion';               // 固定值
  created: number;                         // Unix 时间戳
  model: string;                           // 实际使用的模型
  choices: ChatCompletion.Choice[];         // 通常 1 条
  usage?: CompletionUsage;                 // Token 用量
  service_tier?: 'auto' | 'default' | 'flex' | 'scale' | 'priority';
}

interface ChatCompletion.Choice {
  index: number;
  message: ChatCompletionMessage;          // 完整消息
  finish_reason: 'stop' | 'length' | 'tool_calls' | 'content_filter' | 'function_call';
  logprobs: { content: ChatCompletionTokenLogprob[] | null } | null;
}

interface ChatCompletionMessage {
  role: 'assistant';
  content: string | null;                  // 文本回复
  tool_calls?: ChatCompletionMessageToolCall[];  // 工具调用
  refusal?: string | null;                 // 安全拒绝内容
  annotations?: Annotation[];              // web_search 引用标注
  audio?: ChatCompletionAudio | null;      // 音频输出
}
```

#### 工具调用结果

```typescript
type ChatCompletionMessageToolCall =
  | {
      id: string;
      type: 'function';
      function: { name: string; arguments: string; };
    }
  | {
      id: string;
      type: 'custom';
      custom: { name: string; input: string; };
    };
```

#### Token 用量

```typescript
interface CompletionUsage {
  prompt_tokens: number;
  completion_tokens: number;
  total_tokens: number;
  prompt_tokens_details?: {
    cached_tokens?: number;
    audio_tokens?: number;
  };
  completion_tokens_details?: {
    reasoning_tokens?: number;
    accepted_prediction_tokens?: number;
    rejected_prediction_tokens?: number;
    audio_tokens?: number;
  };
}
```

---

### 流式响应（ChatCompletionChunk）

流式模式下，服务端以 SSE 格式持续推送 delta 块，以特殊数据行 `[DONE]` 结束。

```
data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":"Hello"}}]}

data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":" World"}}]}

data: {"id":"chatcmpl-xxx","object":"chat.completion.chunk","choices":[{"index":0,"delta":{},"finish_reason":"stop"}],"usage":{"prompt_tokens":10,"completion_tokens":2,"total_tokens":12}}

data: [DONE]
```

#### Chunk 结构

```typescript
interface ChatCompletionChunk {
  id: string;
  object: 'chat.completion.chunk';
  created: number;
  model: string;
  choices: {
    index: number;
    delta: {
      role?: 'developer' | 'system' | 'user' | 'assistant' | 'tool';
      content?: string | null;          // 文本增量
      tool_calls?: {                    // 工具调用增量
        index: number;
        id?: string;
        function?: { name?: string; arguments?: string };
      }[];
      refusal?: string | null;
    };
    finish_reason: 'stop' | 'length' | 'tool_calls' | 'content_filter' | 'function_call' | null;
    logprobs?: { content: ChatCompletionTokenLogprob[] | null } | null;
  }[];
  usage?: CompletionUsage | null;  // 仅最后一块（需 stream_options.include_usage）
}
```

#### 流式累积策略

```
delta.content  → 字符串拼接: accumulatedContent += delta.content
delta.tool_calls[0].function.arguments → JSON 拼接: accumulatedJSON += delta.arguments
finish_reason != null → 当前 turn 完成
```

---

## Responses API（新）

新 API 仍在发展中。其设计理念是将 Chat Completions 的无状态请求-响应模型扩展为更丰富的有状态模型：

- **ResponseInputItem / ResponseOutputItem** — 取代 messages 数组，支持更多类型
- **ResponseStreamEvent** — 新的流式事件类型
- **ParsedResponse** — 内置结构化输出解析
- **WebSocket 支持** — 实时通信

## SSE 流式基础设施

OpenAI SDK 的核心流式抽象：

```typescript
class Stream<Item> implements AsyncIterable<Item> {
  // 从 SSE Response 创建
  static fromSSEResponse(response: Response, controller: AbortController): Stream<Item>;

  // 从 ReadableStream 创建
  static fromReadableStream(stream: ReadableStream, controller: AbortController): Stream<Item>;

  // 分叉流
  tee(): [Stream<Item>, Stream<Item>];

  // 转换为 Web ReadableStream
  toReadableStream(): ReadableStream;
}

// SSE 事件格式
interface ServerSentEvent {
  event: string | null;
  data: string;
  raw: string[];
}
```
