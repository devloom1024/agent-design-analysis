# Agent Spaces 统一 Agent Runtime 架构分析

源码位置: `codes/agent-spaces`  
分析版本: `eaca371` (`fix`)

## 项目定位

Agent Spaces 是一个本地多 Agent 协同编程平台，核心目标不是只封装一个 CLI，而是在 Workspace、Channel、Issue、Workflow、Agent Preset 之上统一调度多种 Agent 运行时。

它支持的 runtime 包括:

- `open-agent-sdk`: 基于 `@codeany/open-agent-sdk`
- `claude-code`: 基于 `@anthropic-ai/claude-agent-sdk`
- `codex`: 基于 `@openai/codex-sdk`
- `langchain`: 基于 LangChain.js
- `hermes`: 外部 Hermes CLI 适配

与 ACP/PTY 型项目不同，Agent Spaces 的统一层是一个应用私有的 `AgentRuntime` 接口。Claude Code 和 Codex 都先被适配成统一事件，再由 WebSocket/SSE 和结构化 `MessagePart` 负责展示。

## 统一管理入口

统一入口在:

- `packages/server/src/adapters/agent-runtime-types.ts`
- `packages/server/src/adapters/agent-runtime.ts`
- `packages/shared/src/types/workspace.ts`

核心接口:

```ts
export interface AgentRuntime {
  execute(prompt: string, workingDir: string, options?: AgentRunOptions): Promise<AgentRunResult>;
  stop(): void;
}
```

所有 runtime 都返回统一的 `AgentRunResult`，并通过 `AgentRunOptions.onEvent` 发出统一流式事件:

```ts
export type AgentRuntimeEvent =
  | { type: 'output'; line: string }
  | { type: 'session'; sessionId: string }
  | { type: 'reasoning'; text: string; status?: 'streaming' | 'completed' }
  | { type: 'tool_use'; id: string; name: string; input?: unknown; line: string }
  | { type: 'tool_result'; toolUseId?: string; result: unknown }
  | { type: 'hook_event'; event: ClaudeHookEventName; matcher?: string; payload?: unknown };
```

工厂函数 `createAgentRuntime(config)` 根据 `config.kind` 创建不同实现:

| `runtimeKind` | Runtime class | 统一方式 |
| --- | --- | --- |
| `open-agent-sdk` | `OpenAgentSdkRuntime` | SDK final output/usage |
| `claude-code` | `ClaudeCodeRuntime` | Claude Agent SDK async iterator -> `AgentRuntimeEvent` |
| `codex` | `CodexRuntime` | Codex SDK `runStreamed()` -> `AgentRuntimeEvent` |
| `langchain` | `LangChainRuntime` | LangChain agent result -> unified output |
| `hermes` | `HermesRuntime` | 外部 CLI 进程输出适配 |

Agent Preset 的配置类型是 `AgentConfig`，其中 `runtimeKind` 决定运行时，`modelProvider` 决定底层 API 协议。一个特殊点是如果 `modelProvider` 是:

- `openai-responses-to-anthropic-messages`
- `openai-chat-completions-to-anthropic-messages`

服务层会把 runtime 固定成 `claude-code`，再通过 Anthropic Bridge 让 Claude Code SDK 调 OpenAI 兼容模型。

## 聊天触发链路

频道聊天入口在 `packages/server/src/ws/handler.ts`。

用户发送 `channel.message` 后:

1. 服务端创建用户消息并广播 `channel.message`。
2. 从 HTML 内容和 `mentions` 字段提取 agent id。
3. 对每个 agent 调用 `runMentionedAgent()`。
4. `runMentionedAgent()` 读取 Agent Preset、MCP、Skills、内置 function tools、工作目录和历史消息。
5. 创建 pending assistant message，状态为 `streaming`。
6. 调用 `runtime.execute()`。
7. runtime 事件持续更新 pending message，并广播 `channel.message.updated`。
8. 完成后写入最终 `content/status/metadata/parts`。

简化链路:

```text
channel.message
  -> runMentionedAgent()
  -> createAgentRuntime()
  -> runtime.execute(prompt, workingDir, { onEvent })
  -> AgentRuntimeEvent
  -> liveOutput / toolDetails / reasoning
  -> buildAgentMessageParts()
  -> updateMessage()
  -> channel.message.updated
```

实时更新做了约 120ms 的节流，避免工具调用事件过密时频繁写文件和广播。

## Claude Code Stream 设计

Claude Code runtime 在:

- `packages/server/src/adapters/claude-code-runtime/index.ts`
- `packages/server/src/adapters/claude-code-runtime/message-format.ts`

它直接调用 `@anthropic-ai/claude-agent-sdk` 的 `query({ prompt, options })`，并遍历 SDK 返回的 async iterator。

SDK message 被拆成几类统一事件:

| Claude SDK message | Agent Spaces 事件/输出 |
| --- | --- |
| `assistant` text block | `output` |
| `assistant` thinking block | `reasoning` |
| `assistant` tool_use block | `tool_use` |
| `user.tool_use_result` / `tool_result` block | `tool_result` |
| `result` | 最终 summary/output/usage/sessionId |
| `system task_*` | hook/subagent 相关事件或展示行 |
| `tool_progress` | 工具进度展示行 |

工具调用会被格式化为统一文本行，例如:

```text
Tool: Read file_path="/path/to/file.ts"
Tool: Bash command="pnpm build"
```

这些行既进入实时输出，也用于构造前端的 chain step。完整工具参数和结果不会直接塞进消息正文，而是保存为 tool detail。

Claude Code runtime 还支持:

- `resumeSessionId`: 传给 SDK 的 `resume`
- `configDir/.claude`: 为每个 Agent 准备独立 Claude 配置目录
- skills / commands / subagents 同步到 `.claude`
- MCP servers
- permission mode / sandbox dirs
- output style
- hook event

## Codex Stream 设计

Codex runtime 在:

- `packages/server/src/adapters/codex-runtime.ts`

它使用 `@openai/codex-sdk`:

```ts
const thread = options?.resumeSessionId
  ? codex.resumeThread(options.resumeSessionId, threadOptions)
  : codex.startThread(threadOptions);

const { events } = await thread.runStreamed(prompt, { signal });
```

Codex 的 `ThreadEvent` 被映射为统一事件:

| Codex event/item | Agent Spaces 事件/输出 |
| --- | --- |
| `thread.started` | `session` |
| `item.started` + `command_execution` | `tool_use` (`Bash`) |
| `item.started` + `mcp_tool_call` | `tool_use` (`server.tool`) |
| `item.started` + `web_search` | `tool_use` (`WebSearch`) |
| `item.completed` + command/MCP | `tool_result` |
| `item.completed` + `agent_message` | final text candidate |
| `turn.completed` | usage line + normalized usage |
| `turn.failed` / `error` | error |

Codex runtime 还为每个 Agent 准备独立 `CODEX_HOME`:

```text
{agentDir}/.codex/
  skills/
    {skillName}/SKILL.md
```

它把 Agent Spaces 的 skill markdown 转为 Codex skill 目录结构，并通过 Codex config 注入:

- `mcp_servers`
- `skills.enabled`
- `model_provider`
- `model_reasoning_effort`
- sandbox/approval policy

## Anthropic Bridge 设计

Bridge 位于:

- `packages/server/src/adapters/claude-code-runtime/anthropic-bridge.ts`
- `packages/server/src/adapters/claude-code-runtime/protocol-converter.ts`

用途: 让 Claude Code SDK 仍然以 Anthropic Messages 协议工作，但实际调用 OpenAI Chat Completions 或 OpenAI Responses。

请求转换:

```text
Claude Code SDK
  -> POST /v1/messages       (Anthropic Messages)
  -> convertAnthropicToOpenAI()
  -> /chat/completions 或 /responses
  -> convert...ToAnthropic()
  -> Anthropic Messages response
```

转换规则包括:

- Anthropic `system` -> OpenAI system message / Responses instructions
- Anthropic user text -> OpenAI user message
- Anthropic `tool_use` -> OpenAI function tool call
- Anthropic `tool_result` -> OpenAI tool / function_call_output
- OpenAI tool calls -> Anthropic `tool_use`
- OpenAI usage -> Anthropic usage 字段
- Responses reasoning -> Anthropic `thinking`

一个重要细节: Bridge 当前不是上游真正流式转发。它先 `await upstream.json()` 得到完整响应，如果原始 Anthropic request 要求 `stream: true`，再把完整响应合成为 Anthropic SSE:

```text
message_start
content_block_start
content_block_delta
content_block_stop
message_delta
message_stop
```

因此这里的 stream 更像兼容 Claude Code SDK 预期的 SSE 外形，而不是 token 级透传。

## HTTP SSE API

除了 WebSocket 聊天，项目还提供 `POST /api/agent-sse/run`。

实现位于 `packages/server/src/routes/agent-sse.ts`。

它复用同一套 `createAgentRuntime()` 和 `AgentRuntimeEvent`，但直接写 HTTP SSE:

```text
event: session
data: {...}

event: status
data: {...}

event: output
data: {...}

event: tool_use
data: {...}

event: tool_result
data: {...}

event: done
data: {...}
```

这条链路适合外部系统以 HTTP stream 方式调用 Agent，而频道聊天则走 WebSocket + message persistence。

## Message 数据结构

共享类型在 `packages/shared/src/types/channel.ts`。

`Message` 保留兼容字段:

- `content`: 最终或兼容文本
- `status`: `pending` / `streaming` / `waiting_for_user` / `completed` / `error`
- `metadata`: runtime、model、session id、summary、duration

同时新增结构化展示字段:

- `attachments`
- `parts`
- `replies`

核心是 `MessagePart`:

| Part type | 用途 |
| --- | --- |
| `text` | 最终 Markdown 答案 |
| `reasoning` | 推理/准备状态 |
| `chain` | AI 中间输出和工具调用链 |
| `terminal` | 命令/错误输出 |
| `confirmation` | 权限确认 |
| `context` | token usage、prompt、runtime context |
| `subagent` | Agent 主动调用子 Agent |
| `ask_user_question` | Agent 中断并向用户提问 |

`buildAgentMessageParts()` 位于 `packages/server/src/ws/message-parts.ts`，负责把 runtime 输出行和 tool details 转成结构化 `parts`。

主要策略:

- `Tool:` / `Read` / `Write` / `Edit` / `Bash` 等行进入 chain tool step
- 普通 AI 输出进入 chain message step 或最终 `text`
- 最后一段连续非工具输出被识别为最终答案
- token usage 进入 `context`
- 工具详情只保存 `detailId`
- 重复最终答案会被归一化去重

## Message 存储

消息正文不存 SQLite，而是 JSON 文件。

实现位于:

- `packages/server/src/services/message.ts`
- `packages/server/src/storage/json-store.ts`

路径:

```text
~/.agent-spaces-data/
  workspaces/
    {workspaceId}/
      channels/
        {channelId}/
          messages.json
          tool-details.json
```

`messages.json` 是 `Message[]`:

- `listMessages()` 默认取最后 50 条
- `createMessage()` 读数组、push、写回
- `updateMessage()` 根据 `messageId` 替换后写回
- `appendMessageReply()` 把 reply 追加到原消息的 `replies`

工具详情单独存 `tool-details.json`，实现位于 `packages/server/src/services/tool-detail.ts`。每条详情包含:

- `id`
- `workspaceId`
- `channelId`
- `messageId`
- `title`
- `raw`
- `input`
- `output`
- `createdAt`
- `updatedAt`

这样设计的好处是消息列表轻量，前端展开工具详情时再懒加载完整 input/output。

## SQLite 存储边界

SQLite 主要用于 Agent session 和 usage 统计，不用于聊天 message 正文。

实现位于 `packages/server/src/storage/agent-store.ts`，数据库路径:

```text
~/.agent-spaces-data/agents/agents.sqlite
```

表:

- `agent_sessions`: session id、workspace、agent config、role、status、current task、时间、error
- `agent_usage`: token、cost、summary、error、duration、runtime、model

这说明项目采取了混合存储:

| 数据 | 存储 |
| --- | --- |
| Workspace / Channel / Message / Tool Detail / Preset 等 | JSON 文件 |
| Agent Session / Usage | SQLite |
| Kanban / Database 文档系统 | SQLite |
| Agent 技能、MCP、commands、subagents | 文件目录 |

## 架构特征

### 优点

- `AgentRuntimeEvent` 把 Claude Code 和 Codex 的 stream 差异压到了适配层。
- WebSocket 聊天和 HTTP SSE API 复用同一套 runtime 抽象。
- `MessagePart` 让前端可以稳定渲染工具链、上下文、最终答案，而不依赖原始 SDK transcript。
- 工具详情拆成懒加载文件，避免消息列表过重。
- Claude Code 和 Codex 都有 per-agent 配置目录，便于隔离 skills/MCP/session。
- Bridge 让 Claude Code SDK 可复用 OpenAI Responses/Chat 兼容模型。

### 局限

- Message JSON 每次整体读写，频道消息规模变大后会有写放大和并发覆盖风险。
- `MessagePart` 是展示结构，不是完整 LLM transcript；历史上下文只回放最近消息的文本结果，不保留原始 tool call/tool result 结构。
- Anthropic Bridge 的 SSE 是合成流，不是上游 token 级 stream。
- Claude/Codex 的 tool 语义仍需格式化成 `Tool:` 文本行后再解析成 chain，存在一定字符串协议耦合。

## 归类

在本研究的架构范式中，Agent Spaces 属于:

**私有 Runtime 抽象 + SDK 事件归一化 + WebSocket/SSE 双输出 + JSON/SQLite 混合持久化。**

它比 Claude Code UI 这类纯 provider adapter 更重，因为它同时管理 Workspace、Issue、Workflow、Agent Preset、Skills、MCP、Hook 和消息持久化；但它没有采用 ACP 作为主协议，而是用自己的 `AgentRuntime` 与 `AgentRuntimeEvent` 作为内部标准。

