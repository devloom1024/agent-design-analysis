# AionUi — 消息格式

## 概述

AionUi 使用两套消息系统：前端 `TMessage` 联合类型（14 种）和 ACP 协议 `AcpSessionUpdate` 联合类型（9 种），通过 `AcpAdapter` 进行双向转换。

## TMessage 泛型基接口

```typescript
interface IMessage<T extends TMessageType, Content extends Record<string, any>> {
  id: string;                      // 唯一 ID
  msg_id?: string;                 // 流式合并键
  conversation_id: string;         // 会话 ID
  type: T;                         // 判别字段
  content: Content;                // 泛型内容
  createdAt?: number;              // 创建时间戳
  position?: 'left' | 'right' | 'center' | 'pop';  // UI 位置
  status?: 'finish' | 'pending' | 'error' | 'work';
  hidden?: boolean;
}
```

## TMessageType 枚举（14 种）

| type | content 结构 | 说明 |
|------|-------------|------|
| `text` | `{ content: string; cronMeta?, teammateMessage? }` | 纯文本消息 |
| `tips` | `{ content: string; type: 'error'\|'success'\|'warning' }` | 提示消息 |
| `tool_call` | `{ callId: string; name: string; args; error?; status? }` | 非 ACP 工具调用 |
| `tool_group` | `Array<{ callId, description, name, resultDisplay?, status? }>` | 工具分组 |
| `agent_status` | `{ backend; status: 'connecting'\|'connected'\|'authenticated'\|'session_active'\|'error' }` | Agent 连接状态 |
| `acp_permission` | `AcpPermissionRequest` (含 options 和 toolCall) | ACP 权限请求 |
| `acp_tool_call` | `ToolCallUpdate` (含 status/in_progress/completed/failed) | ACP 工具调用 |
| `codex_permission` | `CodexPermissionRequest` | Codex 权限请求 |
| `codex_tool_call` | `CodexToolCallUpdate` (含 subtype) | Codex 工具调用 |
| `plan` | `{ sessionId; entries: Array<{ content, status, priority }> }` | 计划更新 |
| `thinking` | `{ content: string; subject?; duration?; status: 'thinking'\|'done' }` | 思考内容 |
| `available_commands` | `{ commands: AvailableCommand[] }` | 可用命令列表 |
| `skill_suggest` | `{ cronJobId; name; description; skillContent }` | Skill 建议 |
| `cron_trigger` | `{ cronJobId; cronJobName; triggeredAt }` | Cron 触发 |

## AcpSessionUpdate（9 种 ACP 更新类型）

```typescript
export type AcpSessionUpdate =
  | AgentMessageChunkUpdate      // sessionUpdate = 'agent_message_chunk'
  | AgentThoughtChunkUpdate      // sessionUpdate = 'agent_thought_chunk'
  | ToolCallUpdate               // sessionUpdate = 'tool_call'
  | ToolCallUpdateStatus         // sessionUpdate = 'tool_call_update'
  | PlanUpdate                   // sessionUpdate = 'plan'
  | AvailableCommandsUpdate      // sessionUpdate = 'available_commands_update'
  | UserMessageChunkUpdate       // sessionUpdate = 'user_message_chunk'
  | ConfigOptionsUpdatePayload   // sessionUpdate = 'config_option_update'
  | UsageUpdatePayload;          // sessionUpdate = 'usage_update'
```

## ToolCallUpdate 详析

```typescript
export interface ToolCallUpdate {
  update: {
    sessionUpdate: 'tool_call';
    toolCallId: string;
    status: 'pending' | 'in_progress' | 'completed' | 'failed';
    title: string;
    kind: 'read' | 'edit' | 'execute';
    rawInput?: Record<string, unknown>;
    content?: ToolCallContentItem[];
    locations?: ToolCallLocationItem[];
  };
}
```

## AcpPermissionRequest

```typescript
export interface AcpPermissionRequest {
  sessionId: string;
  options: Array<{
    optionId: string;
    name: string;
    kind: 'allow_once' | 'allow_always' | 'reject_once' | 'reject_always';
  }>;
  toolCall: {
    toolCallId: string;
    rawInput?: { command?; description?; ... };
    title?: string;
    kind?: string;
    content?: ToolCallContentItem[];
    locations?: ToolCallLocationItem[];
  };
}
```

## 流式合并策略（composeMessage）

AionUi 使用 `composeMessage` 实现**按 ID 合并**策略：

| 消息类型 | 合并键 | 合并方式 |
|---------|--------|---------|
| `text` | `msg_id` | 相同 msg_id 追加 content 文本 |
| `tool_group` | `callId` | 合并到已有 group 的 content 数组 |
| `tool_call` | `callId` | 更新已有 tool_call 消息 |
| `acp_tool_call` | `update.toolCallId` | 更新状态/content/rawInput |
| `plan` | `sessionId` | 合并到已有 plan 消息 |
| `thinking` | `msg_id` | 累积 content；status='done' 时更新 |

## AcpAdapter 转换表

| ACP sessionUpdate 值 | 输出 TMessage type | 说明 |
|---------------------|-------------------|------|
| `agent_message_chunk` | `text` | msg_id 流式累积 |
| `agent_thought_chunk` | `tips` (warning) | 居中显示 |
| `tool_call` | `acp_tool_call` | 创建，存 activeToolCalls |
| `tool_call_update` | `acp_tool_call` (更新) | 按 toolCallId 合并 |
| `plan` | `plan` | 同回合内 currentPlanMsgId 复用 |
| `usage_update` | 无 | 由 AcpAgent 直接处理 |
| `config_option_update` | 无 | 由 AcpConnection 处理 |
| `user_message_chunk` | 无 | 回放时忽略 |
