# 消息格式标准化分析

对 7 个项目归一化消息格式的详细对比研究。每个项目都定义了自己的统一消息模型，将不同 AI 工具的原始输出转换为结构化数据。

## 消息格式全景对比

| 项目 | 核心消息类型 | 类型枚举数 | Role 字段 | 流式方式 | 类型安全 |
|------|------------|----------|----------|---------|---------|
| CC GUI | `SDKResult` | ~4 种字符串 type | 无 | 回调函数 | 低 |
| Claude Code UI | `NormalizedMessage` | 14 种 MessageKind | `'user' \| 'assistant'` | stream_delta + stream_end | 中 |
| acpx | `AcpRuntimeEvent` | 5 种事件 + 10 种 tag | User/Agent tagged union | AsyncIterable | 高 |
| AgentAPI | `ConversationMessage` | 2 种角色 + 3 种状态 | `'user' \| 'agent'` | PTY 轮询覆盖 | 高(Go) |
| AionUi | `TMessage` 联合类型 | 14 种 + 9 种 ACP | `position` (left/right/center) | msg_id 合并 | 高 |
| LobeHub | `UIChatMessage` | 13 种 UI 角色 + 15 种 Agent 事件 | `UIMessageRoleType` (12种) | ChatStreamCallbacks | 高 |
| Proma | `AgentEvent` + `ChatStreamState` | 11 种 Agent 事件 + SSE 增量 | `'user' \| 'assistant'` | IPC 推送 + Jotai atom 写入 | 高 |

## 消息类别覆盖对比

| 消息类别 | CC GUI | Claude Code UI | acpx | AgentAPI | AionUi | LobeHub | Proma |
|---------|--------|---------------|------|----------|--------|---------|-------|
| 文本消息 | ✓ | ✓ text | ✓ text_delta | ✓ | ✓ text | ✓ | ✓ Agent/Chat |
| 思考/推理 | — | ✓ thinking | ✓ stream=thought | — | ✓ thinking | ✓ onThinking | ✓ reasoning (含 signature) |
| 工具调用 | — | ✓ tool_use | ✓ tool_call | — | ✓ acp_tool_call | ✓ tool_pending | ✓ ToolActivityItem |
| 工具结果 | — | ✓ tool_result | ✓ (含于 tool_call) | — | ✓ (tool_group) | ✓ tool_result | ✓ tool_result |
| 权限请求 | — | ✓ permission_request | ✓ (permissionStats) | — | ✓ acp_permission | ✓ human_approve | ✓ PERMISSION_REQUEST |
| 状态更新 | — | ✓ status | ✓ status | ✓ status_change | ✓ agent_status | ✓ llm_start/result | ✓ compacting/retry |
| 错误 | ✓ error | ✓ error | ✓ error | ✓ error | ✓ tips | ✓ error | ✓ error (含重试) |
| 会话完成 | ✓ complete | ✓ complete | ✓ done | ✓ | ✓ | ✓ done | ✓ complete (subtype) |
| 计划 | — | — | — | — | ✓ plan | — | ✓ exit_plan |
| 人工交互 | — | ✓ interactive_prompt | — | — | — | ✓ human_prompt/select | ✓ ask_user |
| 上下文压缩 | — | — | — | — | — | — | ✓ compacting |

## 流式消息处理策略对比

| 策略 | 采用项目 | 说明 |
|------|---------|------|
| **替换式** (原地覆盖) | AgentAPI | PTY 模式下最后一条 agent 消息被持续覆盖更新 |
| **追加式** (新消息) | CC GUI, LobeHub | 每个 chunk 作为新消息/事件推送 |
| **合并式** (按 ID 聚合) | AionUi, Claude Code UI | 相同 msg_id 的消息合并为一条，工具调用按 toolCallId 聚合 |
| **迭代器式** (AsyncIterable) | acpx | 运行时事件以异步迭代器形式产出 |
| **原子写入式** (Jotai useStore) | Proma | IPC 事件直接 `useStore().set()` 写入 Jotai atom，`unstable_batchedUpdates` 批量渲染 |

## 各项目详情

| 项目 | 文档 |
|------|------|
| Stream 与入库消息形态设计 | [stream-vs-persisted-message-design.md](./stream-vs-persisted-message-design.md) |
| CC GUI (JetBrains 插件) | [jetbrains-cc-gui.md](./jetbrains-cc-gui.md) |
| Claude Code UI | [claudecodeui.md](./claudecodeui.md) |
| acpx | [acpx.md](./acpx.md) |
| AgentAPI | [agentapi.md](./agentapi.md) |
| AionUi | [AionUi.md](./AionUi.md) |
| LobeHub | [lobehub.md](./lobehub.md) |
| Proma | [proma.md](./proma.md) |
