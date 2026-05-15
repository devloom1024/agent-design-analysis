# SDK 协议层分析

对三种底层 AI 通信协议的详细对比研究。7 个应用项目最终都收敛到这三种消息格式之一。

## 三种协议全景对比

| 维度 | OpenAI | Anthropic | ACP |
|------|--------|-----------|-----|
| **API 端点** | Chat Completions + Responses（两套） | Messages（单一） | JSON-RPC 2.0 双向 |
| **传输协议** | HTTP SSE (`data: [DONE]`) | HTTP SSE（命名事件） | NDJSON over stdio |
| **Content 结构** | 简单字符串 `content: string` + 可选 ContentPart 数组 | `ContentBlock[]` 强类型数组 | `ContentBlock[]` 联合类型（5 种） |
| **角色体系** | 6 种：system/user/assistant/tool/function/developer | 2 种：user/assistant（system 是顶层参数） | 不区分角色（通过消息方向区分） |
| **流式事件** | `data:` 行 + `[DONE]` 哨兵 | 6 种命名 SSE 事件（message_start/content_block_delta 等） | 11 种 `session/update` 变体 |
| **流式增量** | `choices[0].delta.content` 字符串 | `content_block_delta` → `TextDelta`/`InputJSONDelta` 等 | `agent_message_chunk` → `ContentChunk` → `ContentBlock` |
| **工具调用** | `tool_calls[]` + `function`/`custom` 类型 | `ToolUseBlock` + 15 个版本化 Tool 定义 | `tool_call` + `tool_call_update` 增量更新 |
| **思考/推理** | `reasoning_effort` 参数（仅推理模型） | `thinking` 配置 + `ThinkingBlock`（含 `signature`） | `agent_thought_chunk` SessionUpdate |
| **权限模型** | 无内置 | 无内置 | 内置 `session/request_permission` 双向流程 |
| **Session 管理** | 无状态 HTTP | 无状态 HTTP | 有状态：`session/new` → `session/prompt` → `session/close` |
| **类型生成** | OpenAPI → Stainless 自动生成 | 手写 + 部分自动生成 | `schema.json` → Zod + TS 自动生成 |
| **代码仓库** | openai/openai-node | anthropics/anthropic-sdk-typescript | agentclientprotocol/typescript-sdk |

## 协议选择决策树

```
需要哪个协议？
├── 构建 Web 应用、直接调 LLM API
│   ├── 需要 Extended Thinking、签名验证 → Anthropic Messages API
│   └── 需要广泛模型兼容、Responses API 新特性 → OpenAI API
└── 构建 Agent 客户端/服务端、需要双向通信
    └── 需要 Session 管理、权限流程、工具生命周期 → ACP
```

## Content 结构对比

### OpenAI（简单优先）

```
content: string | ContentPart[]
ContentPart = TextPart | ImagePart | AudioPart | FilePart
```

默认纯字符串，多模态场景使用 `ContentPart[]` 数组。结构简单，适合大多数场景。

### Anthropic（强类型数组）

```
content: ContentBlock[]
ContentBlock = TextBlock | ImageBlock | DocumentBlock | ToolUseBlock |
               ToolResultBlock | ThinkingBlock | ServerToolBlock | ...
```

始终使用 `ContentBlock[]` 数组，每个 Block 有明确的 `type` 字段。工具调用、思考过程都作为 ContentBlock 的一部分。

### ACP（协议双向）

```
ContentBlock = TextContent | ImageContent | AudioContent |
               ResourceLink | EmbeddedResource
```

涵盖文本/图片/音频/资源链接/嵌入资源五种类型。与 MCP 协议的类型体系兼容，Agent 可以直接转发 MCP 工具输出。

## 流式事件对比

```
OpenAI SSE:
  data: {"choices":[{"delta":{"content":"Hello"}}],"object":"chat.completion.chunk"}
  data: [DONE]

Anthropic SSE:
  event: message_start      → 完整 Message 对象
  event: content_block_start → content_block 对象 + index
  event: content_block_delta → delta (text_delta / input_json_delta / thinking_delta)
  event: content_block_stop  → index
  event: message_delta       → stop_reason + usage
  event: message_stop        → 结束

ACP NDJSON:
  → {"method":"session/update","params":{"sessionUpdate":"agent_message_chunk",...}}
  → {"method":"session/update","params":{"sessionUpdate":"tool_call",...}}
  → {"method":"session/update","params":{"sessionUpdate":"tool_call_update",...}}
  → PromptResponse: { stopReason: "end_turn", usage: {...} }
```

## 工具调用对比

| 维度 | OpenAI | Anthropic | ACP |
|------|--------|-----------|-----|
| 工具定义 | `tools: [{type: 'function'\|'custom', function/ custom: {...}}]` | `tools: [Tool \| ToolBash \| CodeExecutionTool \| ...]` | Agent 自行实现 |
| 调用格式 | `tool_calls: [{id, type:'function', function: {name, arguments}}]` | `content: [{type:'tool_use', id, name, input}]` | `session/update` 通知 → `ToolCall` + `ToolCallUpdate` |
| 结果格式 | `role: 'tool', tool_call_id, content` | `role: 'user', content: [{type:'tool_result', tool_use_id, content}]` | `ClientRequest → fs/*` 或 `terminal/*` 方法 |
| 状态跟踪 | 无内置 | 无内置 | `status: 'pending' \| 'in_progress' \| 'completed' \| 'failed'` |
| 增量更新 | 流式 delta 累积 | 流式 `input_json_delta` 累积 | `ToolCallUpdate` 任意字段可选更新 |

## 权限机制对比

| 维度 | OpenAI | Anthropic | ACP |
|------|--------|-----------|-----|
| 内置权限 | 无 | 无 | `session/request_permission` |
| 请求结构 | — | — | `{ sessionId, toolCall, options[] }` |
| 选项类型 | — | — | `allow_once \| allow_always \| reject_once \| reject_always` |
| 响应结构 | — | — | `{ outcome: 'cancelled' \| 'selected' }` |
| 超时 | — | — | 由 Client 控制（Agent 暂停等待） |

## 各协议详情

| 协议 | 文档 |
|------|------|
| OpenAI Chat Completions + Responses API | [openai.md](./openai.md) |
| Anthropic Messages API | [anthropic.md](./anthropic.md) |
| ACP (Agent Client Protocol) | [acp.md](./acp.md) |
