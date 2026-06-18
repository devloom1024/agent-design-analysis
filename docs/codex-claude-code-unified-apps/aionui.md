# AionUi：Codex JSON-RPC 与通用 Chat Message

## 统一处理

AionUi 的统一层是 `src/common/chat/chatLib.ts` 中的 `TMessage` 家族。Claude/ACP 侧和 Codex 侧都被转换成同一组 chat message 类型，并写入 SQLite `messages` 表。

Codex 侧有独立的强类型事件定义：

- `src/common/types/codex/types/eventTypes.ts`
- `src/common/types/codex/types/eventData.ts`
- `src/common/types/codex/types/permissionTypes.ts`

持久化在：

- `src/process/services/database/schema.ts`
- `src/process/services/database/types.ts`

## Stream Protocol

### Codex JSON-RPC envelope

```ts
type CodexJsonRpcEvent = {
  jsonrpc: '2.0';
  method: 'codex/event';
  params: {
    _meta: {
      requestId: string;
      timestamp?: string;
      source?: string;
    };
    id: string;
    msg: CodexEventMsg;
  };
}
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `jsonrpc` | 固定 `2.0`，表示 JSON-RPC 消息。 |
| `method` | 固定 `codex/event`，用于区分 Codex event notification。 |
| `params._meta.requestId` | 当前请求/turn id。 |
| `params._meta.timestamp` | 事件产生时间，可选。 |
| `params._meta.source` | 事件来源，可选。 |
| `params.id` | Codex event id。 |
| `params.msg` | 具体事件 payload，见下表。 |

### Codex event type

| type | 数据字段 | 说明 |
| --- | --- | --- |
| `session_configured` | session/config/model 相关字段 | Codex session 初始化完成。 |
| `task_started` | task id/description | turn 或任务开始。 |
| `task_complete` | result/status | turn 或任务完成。 |
| `agent_message_delta` | `delta`/`text` | assistant 文本增量。 |
| `agent_message` | `message`/`text` | 完整 assistant 文本。 |
| `user_message` | `message`/`text` | 用户消息。 |
| `agent_reasoning` | reasoning text/details | 完整 reasoning。 |
| `agent_reasoning_delta` | reasoning delta | reasoning 增量。 |
| `agent_reasoning_raw_content` | raw content | 原始 reasoning 内容。 |
| `agent_reasoning_raw_content_delta` | raw delta | 原始 reasoning 增量。 |
| `agent_reasoning_section_break` | section metadata | reasoning 分段。 |
| `token_count` | usage info | token 使用量。 |
| `exec_command_begin` | command/cwd/parsed command | shell 命令开始。 |
| `exec_command_output_delta` | stdout/stderr delta | 命令输出增量。 |
| `exec_command_end` | exit code/status/output | 命令结束。 |
| `exec_approval_request` | command + approval options | 执行命令审批请求。 |
| `apply_patch_approval_request` | patch + approval options | patch 审批请求。 |
| `patch_apply_begin` | patch/files | patch 开始应用。 |
| `patch_apply_end` | status/summary | patch 应用结束。 |
| `mcp_tool_call_begin` | server/tool/arguments | MCP 工具开始。 |
| `mcp_tool_call_end` | result/error/status | MCP 工具结束。 |
| `mcp_list_tools_response` | tools | MCP 工具列表返回。 |
| `web_search_begin` | query | web search 开始。 |
| `web_search_end` | results/status | web search 结束。 |
| `turn_diff` | file changes/diff | turn 产生的 diff。 |
| `get_history_entry_response` | history entry | 历史消息返回。 |
| `list_custom_prompts_response` | prompts | 自定义 prompt 列表。 |
| `conversation_path` | path | Codex conversation 文件路径。 |
| `background_event` | message/level | 后台事件。 |
| `turn_aborted` | reason | turn 被中止。 |

### AionUi `TMessageType`

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
```

| type | 说明 |
| --- | --- |
| `text` | 普通用户/assistant 文本。 |
| `tips` | 提示类消息。 |
| `tool_call` | 通用工具调用。 |
| `tool_group` | 多个工具调用聚合展示。 |
| `agent_status` | agent 状态更新。 |
| `acp_permission` | ACP 权限请求。 |
| `acp_tool_call` | ACP 工具调用。 |
| `codex_permission` | Codex 权限请求。 |
| `codex_tool_call` | Codex 工具调用/进度更新。 |
| `plan` | 计划列表。 |
| `thinking` | 推理内容。 |
| `available_commands` | 可用命令列表。 |
| `skill_suggest` | skill 建议。 |
| `cron_trigger` | 定时任务触发消息。 |

### Codex tool call update

`codex_tool_call` 的更新子类型包括：

| subtype | 关键字段 | 说明 |
| --- | --- | --- |
| `exec_command_begin` | command/cwd | 命令开始。 |
| `exec_command_output_delta` | stream/delta | 命令 stdout/stderr 增量。 |
| `exec_command_end` | exitCode/status | 命令结束。 |
| `patch_apply_begin` | patch/files | patch 应用开始。 |
| `patch_apply_end` | success/error | patch 应用结束。 |
| `mcp_tool_call_begin` | server/tool/arguments | MCP 调用开始。 |
| `mcp_tool_call_end` | result/error | MCP 调用结束。 |
| `web_search_begin` | query | 搜索开始。 |
| `web_search_end` | results | 搜索结束。 |
| `turn_diff` | diff/files | turn diff。 |
| `generic` | raw payload | 未专门建模的 Codex 更新。 |

## Message 持久化

### 应用级 message

基础 message：

```ts
interface IMessage {
  id: string;
  msg_id?: string;
  conversation_id: string;
  type: TMessageType;
  content: unknown;
  createdAt?: string | number;
  position?: number;
  status?: string;
  hidden?: boolean;
}
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `id` | AionUi 内部消息 id，SQLite 主键。 |
| `msg_id` | provider/runtime 原生消息 id，可用于和 Codex event id 或 ACP id 对齐。 |
| `conversation_id` | 会话 id。 |
| `type` | 上文 `TMessageType`。 |
| `content` | JSON 内容，实际结构由 `type` 决定。Codex tool、ACP tool、thinking、plan 都放这里。 |
| `createdAt` | 创建时间；入库时映射到 `created_at`。 |
| `position` | 会话内排序位置。 |
| `status` | 消息状态，如 running/completed/error。 |
| `hidden` | 是否隐藏，不参与普通渲染。 |

### SQLite `messages` 表

```ts
messages: {
  id,
  conversation_id,
  msg_id,
  type,
  content,
  position,
  status,
  created_at
}
```

`messageToRow()` / `rowToMessage()` 负责转换：

| 列 | 说明 |
| --- | --- |
| `id` | 主键，对应 `IMessage.id`。 |
| `conversation_id` | 会话外键/分组字段。 |
| `msg_id` | provider 原生 id。 |
| `type` | message type 字符串。 |
| `content` | JSON string，保存 `IMessage.content`。 |
| `position` | 排序字段。 |
| `status` | 状态字段。 |
| `hidden` | 类型定义中存在，用于隐藏消息；schema 版本需以实际迁移为准。 |
| `created_at` | 创建时间。 |

### 设计评价

AionUi 把 Codex stream 事件拆得比较细，尤其保留命令输出 delta、patch、MCP、web search、diff 等事件。这对 UI 渲染进度很友好；代价是 `content` 是按 type 分派的 JSON，长期演进需要版本化或迁移策略。
