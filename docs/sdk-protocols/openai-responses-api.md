# OpenAI Responses API 设计与使用指南

本文基于 OpenAI 官方文档重新整理 Responses API，重点服务两类目标：

1. 作为 API 使用参考：请求参数、响应对象、streaming、工具调用、结构化输出、状态管理。
2. 作为 Agent 架构参考：如何设计 durable item、stream event、tool loop、conversation persistence 和 compact。

官方定位上，Responses API 是 OpenAI 当前面向生成、状态、多模态与工具使用的统一接口。它支持文本/图像/文件输入、文本或结构化 JSON 输出、内置工具、自定义 function calling、MCP 工具、streaming、background mode、conversation state 和 response compact。

文档校准时间：2026-05-25。示例中的 `gpt-5.4` 代表当前官方文档中用于 Responses API 示例和工具能力说明的模型族；实际工程中应把模型 ID 作为配置项，并在升级模型时重新核对工具支持、reasoning 参数、token 上限和成本。

## 1. 基本模型

### Endpoint

```http
POST https://api.openai.com/v1/responses
```

一个最小请求：

```ts
import OpenAI from "openai";

const client = new OpenAI();

const response = await client.responses.create({
  model: "gpt-5.4",
  input: "Explain event sourcing in one paragraph.",
});

console.log(response.output_text);
```

Responses API 的核心抽象不是 Chat Completions 的 `choices[0].message`，而是：

- `Response`：一次模型运行的整体对象。
- `input`：本轮输入，可以是字符串，也可以是 typed input item 数组。
- `output`：本轮输出 item 数组，可能包含 assistant message、reasoning、tool call、built-in tool call 等。
- `output_text`：SDK/响应对象上常用的便捷字段，把 assistant 文本输出提取成字符串。
- `previous_response_id` / `conversation`：管理跨轮上下文。
- `tools` / `tool_choice`：给模型可调用工具。
- `stream`：用 SSE 发送 semantic streaming events。

## 2. 请求参数总览

下面按设计意图分组，而不是照 API reference 的字段顺序逐项复制。

### 2.1 模型与输入

```ts
await client.responses.create({
  model: "gpt-5.4",
  input: [
    {
      role: "user",
      content: "Summarize this repository's architecture.",
    },
  ],
});
```

关键字段：

- `model`: 使用的模型 ID。
- `input`: 字符串或 input item 数组。可以承载文本、图像、文件以及工具结果。
- `instructions`: 插入到模型上下文中的 system/developer message。
- `prompt`: 引用 Prompt 模板及变量。
- `max_output_tokens`: 限制生成 token 上限，包含可见输出和 reasoning token。
- `temperature` / `top_p`: 采样控制，一般不要同时调。
- `reasoning`: reasoning 模型配置，例如 effort、summary 等。
- `text`: 文本输出配置，包含 plain text 与 structured JSON。

`instructions` 有一个重要语义：和 `previous_response_id` 一起使用时，前一个 response 的 instructions 不会自动继承到下一轮。这使你可以在新 response 中替换 developer/system 指令，而不用担心旧指令隐式叠加。

### 2.2 状态与持久化

```ts
const first = await client.responses.create({
  model: "gpt-5.4",
  input: "Tell me a joke.",
  store: true,
});

const second = await client.responses.create({
  model: "gpt-5.4",
  previous_response_id: first.id,
  input: "Explain why it is funny.",
  store: true,
});
```

关键字段：

- `store`: 是否存储生成的 response，默认 `true`。如果要用 `previous_response_id` 延续状态，需要确保对应 response 可被引用。
- `previous_response_id`: 链接到前一个 response，创建 threaded conversation。不能和 `conversation` 同时使用。
- `conversation`: 指定一个 long-running conversation object。该 conversation 的 items 会被 prepended 到本次 input items；本次 input/output items 完成后也会自动加入 conversation。
- `truncation`: 超上下文时的截断策略。`disabled` 默认会在超出 context window 时报错；`auto` 会从 conversation 开头丢弃 items 以适配窗口。

三种状态模式：

| 模式 | 适用场景 | 你负责什么 |
| --- | --- | --- |
| 手动拼 history，`store: false` | 数据保留要求严格、自研持久化、ZDR 场景 | 每轮把历史 input/output 自己传回 |
| `previous_response_id` | 线性多轮对话、轻量状态链 | 保存上一轮 response id |
| `conversation` | 跨设备、跨 job、长生命周期线程 | 保存 conversation id，必要时管理 conversation items |

### 2.3 工具

```ts
const response = await client.responses.create({
  model: "gpt-5.4",
  input: "What changed in AI news today?",
  tools: [{ type: "web_search" }],
  tool_choice: "auto",
});
```

关键字段：

- `tools`: 模型可调用工具列表。
- `tool_choice`: 控制模型如何选择工具。
- `parallel_tool_calls`: 是否允许并行 tool calls，默认 `true`。
- `max_tool_calls`: 限制 built-in tool call 总数。
- `include`: 请求 response 额外包含某些工具结果细节，例如 web search sources、file search results、code interpreter outputs、image URLs、output logprobs、reasoning encrypted content。

工具类别：

- Built-in tools: web search、file search、computer use、code interpreter 等。
- MCP tools: 远程 MCP server 或预定义 connector。
- Function calls: 你定义的自定义代码工具。
- Tool search: 运行时按需加载 deferred tool definitions。官方文档说明只有 `gpt-5.4` 及之后模型支持 `tool_search`。

### 2.4 Streaming 与异步运行

```ts
const stream = await client.responses.create({
  model: "gpt-5.4",
  input: "Say 'double bubble bath' ten times fast.",
  stream: true,
});

for await (const event of stream) {
  console.log(event.type, event);
}
```

关键字段：

- `stream`: `true` 时通过 SSE 返回语义化 streaming events。
- `stream_options`: 仅在 `stream: true` 时使用。
- `background`: `true` 时让 response 在后台运行，可后续 retrieve/cancel。

Responses API 的 streaming 不只是文本 chunk。它是 semantic events，比如：

- `response.created`
- `response.output_text.delta`
- `response.completed`
- `response.function_call_arguments.delta`
- `response.function_call_arguments.done`
- file search / code interpreter / tool call 相关事件
- `error`

这对 Agent UI 很重要：你不需要从纯文本 delta 反推“工具是否开始/结束”，而是可以用 typed event 驱动状态机。

## 3. Response 对象

典型 Response 包含：

- `id`: response id，可用于 retrieve、delete、cancel、previous_response_id。
- `object`: response object 类型。
- `created_at`: 创建时间。
- `status`: `completed`、`failed`、`in_progress`、`cancelled`、`queued`、`incomplete` 等。
- `model`: 实际使用模型。
- `input`: 输入。
- `output`: 输出 item 数组。
- `output_text`: SDK/响应便捷字段。
- `usage`: token usage。
- `error`: 失败时的错误信息。
- `incomplete_details`: incomplete 时的细节。
- `parallel_tool_calls`
- `previous_response_id`
- `reasoning`
- `store`
- `text`
- `tool_choice`
- `tools`
- `truncation`
- `metadata`

Agent 设计上应特别关注 `output`。它不是“一个字符串”，而是一组 typed items。每个 item 都应该被视作可持久化的事件/事实。

## 4. Input 与 Output Item 思维

Responses API 更接近 event/item log，而不是单条 assistant message。

### 4.1 输入

最简单输入可以是字符串：

```ts
await client.responses.create({
  model: "gpt-5.4",
  input: "Write a haiku about distributed systems.",
});
```

复杂输入应使用 typed items：

```ts
await client.responses.create({
  model: "gpt-5.4",
  input: [
    {
      role: "user",
      content: [
        { type: "input_text", text: "What is in this image?" },
        { type: "input_image", image_url: "https://example.com/image.png" },
      ],
    },
  ],
});
```

对于自研 Agent，建议内部统一为 item 数组，即使用户只是发一段文本。这样后续加入图片、文件、tool output、系统上下文、compact item 时不会改整体数据模型。

### 4.2 输出

`response.output` 可能包含：

- assistant message item
- reasoning item
- function call item
- built-in tool call item
- tool result 相关 item
- structured output message

不要只保存 `output_text`。`output_text` 适合 UI 快速展示，但无法表达完整历史。持久化应保存 `output` 中的 typed item。

## 5. Conversation State

官方文档提供三种管理方式。

### 5.1 手动管理

每次请求都传完整 history：

```ts
let history = [
  { role: "user", content: "tell me a joke" },
];

const first = await client.responses.create({
  model: "gpt-5.4",
  input: history,
  store: false,
});

history = [
  ...history,
  ...first.output.map((item) => ({
    role: item.role,
    content: item.content,
  })),
  { role: "user", content: "tell me another" },
];

const second = await client.responses.create({
  model: "gpt-5.4",
  input: history,
  store: false,
});
```

优点：

- 你完全控制持久化、裁剪、脱敏、加密、审计。
- 适合自研 Agent 的 durable log。

缺点：

- 每轮都要传上下文。
- 要自己处理 token budget、compact、tool output 连接。

### 5.2 `previous_response_id`

适合线性线程：

```ts
const first = await client.responses.create({
  model: "gpt-5.4",
  input: "tell me a joke",
  store: true,
});

const second = await client.responses.create({
  model: "gpt-5.4",
  previous_response_id: first.id,
  input: [{ role: "user", content: "explain why this is funny." }],
});
```

优点：

- 简化多轮。
- 不需要每次传完整历史。

注意：

- 这仍会按链上的历史 input tokens 计费。
- 如果你要 fork、rollback 或跨供应商持久化，仍建议自己保存 item log。
- `previous_response_id` 不能和 `conversation` 一起用。

### 5.3 Conversations API

Conversations API 与 Responses API 配合使用，提供 long-running object：

```ts
const conversation = await client.conversations.create();

await client.responses.create({
  model: "gpt-5.4",
  conversation: conversation.id,
  input: [{ role: "user", content: "What are the 5 Ds of dodgeball?" }],
});
```

优点：

- conversation 拥有 durable id。
- 可以跨 session、device、job 使用。
- conversation 存储 messages、tool calls、tool outputs 等 items。

适合产品级 Chat/Agent 线程；但如果你要实现完全可控的 agent replay、compact、debug trace，仍建议在自己系统里镜像一份 durable log。

## 6. Function Calling

Function calling 是模型提出“我要调用某个工具”，你的系统执行工具并把结果返回给模型。

### 6.1 定义工具

```ts
const tools = [
  {
    type: "function",
    name: "get_weather",
    description: "Get current weather for a city.",
    parameters: {
      type: "object",
      properties: {
        city: { type: "string" },
      },
      required: ["city"],
      additionalProperties: false,
    },
    strict: true,
  },
];
```

字段要点：

- `type`: function tool 固定为 `function`。
- `name`: 工具名。
- `description`: 何时/如何使用。
- `parameters`: JSON Schema。
- `strict`: 是否启用 strict schema。

### 6.2 执行 tool loop

```ts
let input = [
  { role: "user", content: "What's the weather in Paris?" },
];

let response = await client.responses.create({
  model: "gpt-5.4",
  input,
  tools,
});

input = [...input, ...response.output];

for (const item of response.output) {
  if (item.type !== "function_call") continue;

  const args = JSON.parse(item.arguments);
  const result = await getWeather(args.city);

  input.push({
    type: "function_call_output",
    call_id: item.call_id,
    output: JSON.stringify(result),
  });
}

response = await client.responses.create({
  model: "gpt-5.4",
  input,
  tools,
});
```

关键不变量：

- tool output 必须引用对应 tool call 的 `call_id`。
- 模型输出的 `function_call` item 应保存进 history。
- 你的 `function_call_output` 也应保存进 history。
- 如果支持并行工具，必须能同时处理多个 tool call。

### 6.3 Streaming function calls

Streaming 下 function call arguments 不是一次性返回，常见事件包括：

- `response.output_item.added`: 新 function call item 出现。
- `response.function_call_arguments.delta`: arguments 的 JSON 字符串增量。
- `response.function_call_arguments.done`: 完整 function call arguments 就绪。

因此 streaming tool parser 应该：

- 以 `item_id` 或 `output_index` 建立 accumulator。
- 对 arguments delta 做字符串拼接。
- 只在 done 后 JSON parse 并执行工具。
- 不要把未完成 arguments 当成 durable tool call。

## 7. Structured Outputs

Structured Outputs 有两种形式：

1. Function calling strict schema：适合模型调用你的工具。
2. `text.format` 的 `json_schema`：适合模型最终回答要符合某个 JSON Schema。

### 7.1 最终回答结构化

```ts
const response = await client.responses.create({
  model: "gpt-5.4",
  input: "Extract the action items from this meeting note...",
  text: {
    format: {
      type: "json_schema",
      name: "action_items",
      strict: true,
      schema: {
        type: "object",
        properties: {
          items: {
            type: "array",
            items: {
              type: "object",
              properties: {
                owner: { type: "string" },
                task: { type: "string" },
                due_date: { type: "string" },
              },
              required: ["owner", "task", "due_date"],
              additionalProperties: false,
            },
          },
        },
        required: ["items"],
        additionalProperties: false,
      },
    },
  },
});
```

### 7.2 选择原则

用 function calling，当：

- 模型需要调用你的系统能力。
- 你要执行数据库查询、RPC、文件操作、UI action。
- 输出不是给用户看的最终答案，而是工具参数。

用 `text.format`，当：

- 模型最终回答需要结构化。
- 你要生成 UI 可直接消费的 JSON。
- 不涉及执行外部工具。

Structured Outputs 优先于旧 JSON mode，因为它不仅保证 JSON 可解析，还要求符合 schema。

## 8. Built-in Tools 与 MCP

Responses API 支持多类工具：

### Web search

```ts
const response = await client.responses.create({
  model: "gpt-5.4",
  tools: [{ type: "web_search" }],
  input: "What was a positive news story from today?",
});
```

适合需要当前公开互联网信息的回答。可以通过 `include` 请求 sources。

### File search / retrieval

适合把私有文件或知识库接入模型上下文。通常和 vector stores 或文件检索配置一起使用。

### Code interpreter

适合让模型执行 Python、分析数据、生成中间文件。可通过 `include` 请求 code interpreter outputs。

### Computer use

适合屏幕级操作任务。输出中可能包含 computer call output，可通过 `include` 请求 image URL。

### MCP tools

适合接入第三方服务或你自己的工具服务器。对自研 Agent 来说，MCP 是一个很自然的工具注册与执行边界。

### Tool search

适合工具数量很多、需要运行时按需发现工具定义的场景。注意模型支持限制。

## 9. Streaming 事件设计

Responses API streaming 是 semantic SSE，而不是单纯 token delta。

常见文本流事件：

- `response.created`
- `response.output_text.delta`
- `response.completed`
- `error`

工具相关可能有：

- `response.output_item.added`
- `response.function_call_arguments.delta`
- `response.function_call_arguments.done`
- file search 状态事件
- code interpreter 状态和代码 delta

Agent UI 推荐做法：

- `response.created`: 创建 turn runtime state。
- `response.output_item.added`: 创建 UI item placeholder。
- `response.output_text.delta`: 追加到 streaming text buffer。
- `response.function_call_arguments.delta`: 追加到 tool argument buffer。
- `response.function_call_arguments.done`: 标记 tool call 参数完整，可进入 approval/execution。
- `response.completed`: 用完整 response/output 替换 streaming buffer，写入 durable log。
- `error`: 标记本轮失败，不要把半成品当完成态历史。

关键原则：

- delta 是 transient state。
- completed response/output 是 durable fact。
- UI 可用 delta 提前渲染，但持久化应以完整 item 为准。

## 10. Background Mode

`background: true` 让 response 在后台运行，适合长任务：

```ts
const response = await client.responses.create({
  model: "gpt-5.4",
  input: "Run a deep analysis over these files.",
  background: true,
});

// later
const current = await client.responses.retrieve(response.id);
```

设计建议：

- 后台任务应把 `response.id` 保存到本地 job table。
- UI 状态以 `status` 字段驱动：`queued`、`in_progress`、`completed`、`failed`、`cancelled`、`incomplete`。
- 支持 cancel 时调用 response cancel endpoint。
- 后台任务完成后，把完整 `output` 合并进你的 durable thread。

## 11. Compact

Responses API 提供 `/v1/responses/compact`：

```http
POST https://api.openai.com/v1/responses/compact
```

它对 conversation 运行 compact pass，返回 encrypted/opaque compacted items。官方提示 compact 逻辑可能随时间演进。

对自研 Agent 的含义：

- 不要假设 compact item 是你能解析的摘要文本。
- 如果你用 OpenAI compact，应把 compacted response/items 当作模型可继续消费的 opaque durable item。
- 如果你要跨模型/跨供应商恢复，仍建议自己保存一份可解释摘要和原始历史边界。
- compact 之后要明确记录：compact 前历史边界、compact item、compact 后 reference context。

## 12. 与 Chat Completions 的差异

| 维度 | Chat Completions | Responses API |
| --- | --- | --- |
| 核心输入 | `messages` | `input` typed items |
| 核心输出 | `choices[].message` | `output` typed items + `output_text` |
| 多轮状态 | 手动传 messages | `previous_response_id` / `conversation` / 手动 |
| 工具调用 | `tool_calls` + tool message | `function_call` item + `function_call_output` item |
| Streaming | chunk/delta 为主 | semantic events |
| Built-in tools | 有限/分散 | web/file/computer/code/MCP/tool search 等统一在 `tools` |
| Agent 友好度 | 需要自建状态机 | 更接近 event/item log |

迁移时最常见的心智切换：

- 不要只看 `output_text`，要处理 `output`。
- 不要把 tool call 当 assistant message 字段，要当 output item。
- 不要把 stream 当纯文本流，要当 typed event stream。
- 不要只用 role/content 设计 history，要支持 typed items。

## 13. 自研 Agent 的推荐数据模型

### 13.1 Durable log

```ts
type DurableThreadItem =
  | {
      type: "user_message";
      id: string;
      content: UserContentPart[];
      created_at: string;
    }
  | {
      type: "assistant_message";
      id: string;
      response_id: string;
      content: AssistantContentPart[];
      created_at: string;
    }
  | {
      type: "reasoning";
      id: string;
      response_id: string;
      summary?: string;
      encrypted_content?: string;
    }
  | {
      type: "function_call";
      id: string;
      response_id: string;
      call_id: string;
      name: string;
      arguments: string;
    }
  | {
      type: "function_call_output";
      id: string;
      call_id: string;
      output: string;
    }
  | {
      type: "builtin_tool_call";
      id: string;
      response_id: string;
      tool_type: string;
      status: "in_progress" | "completed" | "failed";
      payload: unknown;
    }
  | {
      type: "compaction";
      id: string;
      response_id: string;
      compacted_items: unknown[];
      boundary: { from_item_id: string; to_item_id: string };
    };
```

### 13.2 Runtime event

```ts
type RuntimeEvent =
  | { type: "turn.started"; turn_id: string }
  | { type: "item.started"; turn_id: string; item_id: string; item_type: string }
  | { type: "item.delta"; turn_id: string; item_id: string; delta: unknown }
  | { type: "item.completed"; turn_id: string; item: DurableThreadItem }
  | { type: "tool.approval_requested"; call_id: string; reason: string }
  | { type: "tool.completed"; call_id: string; output_item: DurableThreadItem }
  | { type: "turn.completed"; turn_id: string; response_id: string; usage: unknown }
  | { type: "turn.failed"; turn_id: string; error: unknown };
```

### 13.3 Turn loop

```ts
async function runTurn(thread: Thread, userInput: UserInput) {
  const input = buildResponsesInput(thread.durableItems, userInput);

  const stream = await client.responses.create({
    model: thread.model,
    input,
    tools: thread.tools,
    stream: true,
    store: true,
  });

  const accumulators = new Map<string, unknown>();

  for await (const event of stream) {
    applyRuntimeEvent(accumulators, event);
    publishToUi(event);
  }

  const response = await client.responses.retrieve(/* response id */);
  appendDurableItems(thread, response.output);

  const toolCalls = response.output.filter((item) => item.type === "function_call");
  if (toolCalls.length > 0) {
    const toolOutputs = await executeToolCalls(toolCalls);
    appendDurableItems(thread, toolOutputs);
    return continueTurnWithToolOutputs(thread, toolOutputs);
  }

  return response;
}
```

实际实现中，stream 完成时 SDK 通常已经给你 final response；这里用 retrieve 只是表达“最终持久化以完整 response/output 为准”这个原则。

## 14. 工程实践建议

### 14.1 保存 response id

每个 assistant turn 保存：

- `response.id`
- `previous_response_id`
- `conversation id`，如果使用 Conversations API
- `model`
- `status`
- `usage`
- `created_at`
- `output item ids`

这样可以：

- retrieve 历史 response。
- 继续 `previous_response_id`。
- debug token/cost。
- 支持后台任务恢复。

### 14.2 不要把 `output_text` 当唯一事实

`output_text` 会丢失：

- tool call
- tool result
- reasoning item
- structured output metadata
- annotations/sources
- image/code/file tool outputs

适合 UI summary，不适合 durable history。

### 14.3 Tool output 要和 call id 强绑定

任何工具结果都必须引用原始 `call_id`。没有 call id 的 tool output 不应进入模型历史，否则模型无法把结果对应回工具调用。

### 14.4 Streaming parser 要可恢复

为每个 `response_id + item_id + output_index` 建 accumulator：

- 文本 delta accumulator。
- function arguments accumulator。
- code interpreter code/output accumulator。
- reasoning summary accumulator。

完成事件到达前，只能作为 UI state；完成事件到达后再转 durable item。

### 14.5 自己做长期记忆时优先手动 history

如果你的 Agent 需要：

- 跨供应商迁移
- 可解释 compact
- rollback/fork
- 对每个工具调用做审计
- 加密/脱敏/租户隔离
- 本地优先存储

建议使用 `store: false` 或同步镜像 OpenAI conversation state，把完整 item log 存到自己的 DB。

## 15. 官方来源

- Responses API reference: https://platform.openai.com/docs/api-reference/responses
- Conversation state guide: https://developers.openai.com/api/docs/guides/conversation-state
- Streaming responses guide: https://developers.openai.com/api/docs/guides/streaming-responses
- Function calling guide: https://developers.openai.com/api/docs/guides/function-calling
- Structured Outputs guide: https://developers.openai.com/api/docs/guides/structured-outputs
- Using tools guide: https://developers.openai.com/api/docs/guides/tools

## 16. 结论

Responses API 最适合用“typed item log + semantic runtime events”来理解。

对简单应用：

- 用 `input` + `output_text` 快速生成文本。
- 用 `previous_response_id` 或 `conversation` 简化多轮。

对 Agent 系统：

- 把 `response.output` 当 durable history。
- 把 streaming events 当 runtime UI state。
- 把 function call / function_call_output 当 tool loop 的核心协议。
- 把 `conversation` 或 `previous_response_id` 当 OpenAI 侧状态引用，但本地仍保存可 replay 的 item log。
- compact 时保留边界与可解释摘要，不要只依赖 opaque compact item。
