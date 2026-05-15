# LobeHub — 工具调用与权限

## 架构特点

**指令系统模式**——Agent（大脑）产生指令，Runtime（引擎）执行。权限通过 `request_human_approve` 指令实现人在回路。

## ChatToolPayload 核心类型

```typescript
interface ChatToolPayload {
  apiName: string;               // 工具 API 名（如 'Bash', 'Read'）
  arguments: string;             // JSON 字符串参数
  id: string;                    // 工具调用 ID
  identifier: string;            // 提供者标识（如 'claude-code'）
  type: LobeToolRenderType;      // 'mcp' | 'default' | 'markdown' | 'standalone'
  executor?: 'client' | 'server';
  source?: ToolSource;           // 'builtin' | 'client' | 'mcp' | 'klavis' | 'lobehubSkill'
  intervention?: ToolIntervention;
  thoughtSignature?: string;
}

interface ToolIntervention {
  status?: 'pending' | 'approved' | 'rejected' | 'aborted' | 'none';
  rejectedReason?: string;
}
```

## 工具调用完整流转路径

```
Agent.runner() → 返回 AgentInstruction
  → call_tool / call_tools_batch 指令
    → InstructionExecutor 执行
      → 发射 tool_pending 事件
      → ToolRegistry 查找 handler
      → handler.exec(args)
      → 发射 tool_result 事件
  → request_human_approve 指令
    → 发射 human_approve_required 事件
    → status = 'waiting_for_human'
    → 用户审批 → human_approved_tool phase → 继续 step()
```

## 人机交互指令

### request_human_approve

```typescript
AgentInstructionRequestHumanApprove = {
  type: 'request_human_approve',
  payload: {
    pendingToolsCalling: ChatToolPayload[],  // 待审批工具列表
    reason?: string,
    skipCreateToolMessage?: boolean
  }
}
```

**执行流程**：
1. `status = 'waiting_for_human'`
2. `pendingToolsCalling` 存入 AgentState
3. 发射 `human_approve_required` 事件
4. 用户调用 `approveToolCall()` → `human_approved_tool` phase → 继续 step()

### request_human_prompt

```typescript
AgentInstructionRequestHumanPrompt = {
  type: 'request_human_prompt',
  payload: { prompt: string, metadata?: Record<string, unknown> }
}
```

用户输入后进入 `human_response` phase 继续。

### request_human_select

```typescript
AgentInstructionRequestHumanSelect = {
  type: 'request_human_select',
  payload: {
    options: Array<{ label: string; value: string }>,
    multi?: boolean,
    prompt?: string
  }
}
```

## 安全配置

```typescript
interface UserInterventionConfig {
  // 工具审批策略配置
}

interface SecurityBlacklistConfig {
  // 工具/命令黑名单
}
```

在 AgentState 中：

```typescript
AgentState.securityBlacklist?: SecurityBlacklistConfig;
AgentState.userInterventionConfig?: UserInterventionConfig;
```

## Claude Code 工具集成

### 工具标识

```typescript
const ClaudeCodeIdentifier = 'claude-code';
```

### 支持的 Claude Code 工具 API

```typescript
enum ClaudeCodeApiName {
  Agent = 'Agent',              // 子 Agent → exec_sub_agent 指令
  AskUserQuestion = 'askUserQuestion',  // 人工提问（特殊路由到干预 UI）
  Bash = 'Bash',
  Edit = 'Edit',
  Glob = 'Glob',
  Grep = 'Grep',
  Read = 'Read',
  ScheduleWakeup = 'ScheduleWakeup',
  Skill = 'Skill',
  TaskOutput = 'TaskOutput',
  TaskStop = 'TaskStop',
  TodoWrite = 'TodoWrite',
  ToolSearch = 'ToolSearch',
  Write = 'Write',
}
```

### 集成方式

```
Claude API tool_use block → ClaudeCodeAdapter
  → 转换为 ChatToolPayload (identifier = 'claude-code')
  → builtin-tool-claude-code 包处理

特殊处理:
  - Agent tool_use → exec_client_sub_agent / exec_sub_agent 指令
  - AskUserQuestion → 重写为专用 apiName → 路由到干预 UI
```

### 三个子系统

1. **Inspectors**: 每个 `ClaudeCodeApiName` 对应一个 React 组件，展示工具调用详情
2. **Interventions**: `AskUserQuestionIntervention` 组件处理澄清问题流程
3. **Renders**: 工具执行结果的可视化渲染（含 Streaming 变体）

## Agent 事件中的权限/工具类型

| AgentEvent.type | 说明 |
|----------------|------|
| `tool_pending` | 工具调用待处理 |
| `tool_result` | 工具执行结果 |
| `human_approve_required` | 需要人工审批 |
| `human_prompt_required` | 需要人工输入 |
| `human_select_required` | 需要人工选择 |

## CompressedGroup 角色

在 UIChatMessage 中，`compressedGroup` 角色用于将上下文压缩后的消息折叠显示，不影响权限和工具调用流程。
