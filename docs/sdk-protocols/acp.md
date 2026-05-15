# ACP — Agent Client Protocol 接口定义

## 概述

ACP (Agent Client Protocol) 是基于 **JSON-RPC 2.0 over NDJSON** 的双向通信协议，专为 AI Agent 的客户端-服务端交互设计。与 OpenAI/Anthropic 的 HTTP API 不同，ACP 是**有状态、双向、长连接**的协议。

## 传输层

### JSON-RPC 2.0 基座

```typescript
// 请求（期望响应）
interface AnyRequest {
  jsonrpc: '2.0';
  id: string | number;
  method: string;
  params?: Record<string, unknown>;
}

// 响应
interface AnyResponse {
  jsonrpc: '2.0';
  id: string | number;
  // 成功: { result: unknown }
  // 失败: { error: { code: number; message: string; data?: unknown } }
}

// 通知（无 id，无需响应）
interface AnyNotification {
  jsonrpc: '2.0';
  method: string;
  params?: Record<string, unknown>;
}
```

### NDJSON 传输

每条完整的 JSON 消息占一行，每条消息以换行符 `\n` 分隔。

```
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{...}}\n
{"jsonrpc":"2.0","id":1,"result":{...}}\n
{"jsonrpc":"2.0","method":"session/update","params":{...}}\n
```

---

## 连接生命周期

### 初始化（initialize）

Client → Agent:

```typescript
interface InitializeRequest {
  protocolVersion: number;                      // 客户端支持的最新协议版本
  clientCapabilities?: ClientCapabilities;       // 客户端能力声明
  clientInfo?: { name: string; title?: string; version: string };
}
```

Agent → Client:

```typescript
interface InitializeResponse {
  protocolVersion: number;                      // 协商后的协议版本
  agentCapabilities?: AgentCapabilities;         // Agent 能力声明
  agentInfo?: { name: string; title?: string; version: string };
  authMethods?: AuthMethod[];                    // 支持的认证方式
}
```

**版本协商**：客户端声明支持的版本，Agent 返回双方兼容的版本。如果客户端不支持返回的版本，应断开连接。

---

## Session 管理

### 创建会话（session/new）

```typescript
interface NewSessionRequest {
  workingDirectory?: string;       // 工作目录
  mcpServers?: McpServerConfig[];  // MCP Server 配置
  context?: SessionContext;         // 上下文信息
}

interface NewSessionResponse {
  sessionId: string;
}
```

### 加载会话（session/load）

```typescript
interface LoadSessionRequest {
  sessionId: string;
}

interface LoadSessionResponse {
  sessionId: string;
  // Agent 通过 session/update 通知回放完整历史
}
```

### 分叉会话（session/fork）— 实验性

```typescript
interface ForkSessionRequest {
  sessionId: string;
  messageId?: string;  // 从哪条消息处开始分叉
}

interface ForkSessionResponse {
  sessionId: string;   // 新会话 ID
}
```

### 恢复会话（session/resume）

```typescript
interface ResumeSessionRequest {
  sessionId: string;
}

interface ResumeSessionResponse {
  sessionId: string;
}
```

### 关闭会话（session/close）

```typescript
interface CloseSessionRequest {
  sessionId: string;
}
// 返回 void
```

---

## Prompt 交互（核心流程）

### 发送 Prompt（session/prompt）

```typescript
interface PromptRequest {
  sessionId: string;
  prompt: ContentBlock[];        // 用户消息内容
  messageId?: string;            // 客户端分配的消息 ID（UUID）
}

interface PromptResponse {
  stopReason: StopReason;        // 停止原因
  usage?: Usage | null;          // Token 用量
  userMessageId?: string;        // 回声确认消息 ID
}

type StopReason =
  | 'end_turn'          // 自然完成
  | 'max_tokens'        // 达到 token 上限
  | 'max_turn_requests' // 达到工具调用轮次上限
  | 'refusal'           // 安全拒绝
  | 'cancelled';        // 被取消
```

### 取消 Prompt（session/cancel）

```typescript
// 通知（无需响应）
interface CancelNotification {
  sessionId: string;
}
```

取消后 Agent 应：
1. 停止所有 LLM 请求
2. 中止正在执行的工具调用
3. 发送最终的更新通知
4. 返回 `stopReason: 'cancelled'`

---

## 流式更新（session/update）

`session/update` 是 Agent → Client 的**通知**，承载全部流式输出。

### SessionUpdate 联合类型（11 种变体）

```typescript
type SessionUpdate =
  | ContentChunk & { sessionUpdate: 'user_message_chunk' }
  | ContentChunk & { sessionUpdate: 'agent_message_chunk' }
  | ContentChunk & { sessionUpdate: 'agent_thought_chunk' }
  | ToolCall & { sessionUpdate: 'tool_call' }
  | ToolCallUpdate & { sessionUpdate: 'tool_call_update' }
  | Plan & { sessionUpdate: 'plan' }
  | AvailableCommandsUpdate & { sessionUpdate: 'available_commands_update' }
  | CurrentModeUpdate & { sessionUpdate: 'current_mode_update' }
  | ConfigOptionUpdate & { sessionUpdate: 'config_option_update' }
  | SessionInfoUpdate & { sessionUpdate: 'session_info_update' }
  | UsageUpdate & { sessionUpdate: 'usage_update' };
```

### ContentChunk — 消息增量

```typescript
interface ContentChunk {
  content: ContentBlock;          // 内容块
  messageId?: string;             // 所属消息 UUID
}
```

### ContentBlock — 内容类型

```typescript
type ContentBlock =
  | { type: 'text'; text: string; annotations?: unknown }
  | { type: 'image'; data: string; mimeType: string; uri?: string }
  | { type: 'audio'; data: string; mimeType: string }
  | { type: 'resource_link'; name: string; uri: string; title?: string; description?: string; mimeType?: string; size?: number }
  | { type: 'resource'; resource: TextResourceContents | BlobResourceContents };
```

**与 MCP 协议的兼容性**：ContentBlock 类型体系与 MCP（Model Context Protocol）保持一致，Agent 可以直接转发 MCP 工具输出而不需要类型转换。

---

## 工具调用

### ToolCall — 工具调用开始

```typescript
interface ToolCall {
  toolCallId: string;                        // 唯一 ID
  title: string;                             // 人类可读标题
  kind?: ToolKind;                           // 工具种类
  status?: ToolCallStatus;                    // 执行状态
  rawInput?: unknown;                        // 原始输入
  rawOutput?: unknown;                       // 原始输出
  locations?: ToolCallLocation[];             // 影响的文件位置
  content?: ToolCallContent[];               // 显示内容
}

type ToolKind =
  | 'read' | 'edit' | 'delete' | 'move'
  | 'search' | 'execute' | 'think'
  | 'fetch' | 'switch_mode' | 'other';

type ToolCallStatus =
  | 'pending' | 'in_progress' | 'completed' | 'failed';

interface ToolCallLocation {
  path: string;
  line?: number | null;
}

type ToolCallContent =
  | { type: 'content'; ... }    // 通用内容
  | { type: 'diff'; ... }      // Diff 视图数据
  | { type: 'terminal'; ... }; // 终端输出
```

### ToolCallUpdate — 工具调用增量更新

```typescript
interface ToolCallUpdate {
  toolCallId: string;                        // 必填：要更新的工具 ID
  // 以下全部可选，只传变更的字段：
  title?: string | null;
  kind?: ToolKind | null;
  status?: ToolCallStatus | null;
  rawInput?: unknown;
  rawOutput?: unknown;
  locations?: ToolCallLocation[] | null;
  content?: ToolCallContent[] | null;
}
```

**特征**：除了 `toolCallId` 外全部可选，实现"只更新变化字段"的增量模式。

---

## 权限机制

### 请求权限（session/request_permission）

Agent → Client:

```typescript
interface RequestPermissionRequest {
  sessionId: string;
  toolCall: ToolCallUpdate;          // 需要授权的工具调用
  options: PermissionOption[];       // 可供选择的权限选项
}

interface PermissionOption {
  optionId: string;                  // 唯一 ID
  name: string;                      // 显示标签（如 "Allow once"）
  kind: PermissionOptionKind;        // 选项性质
}

type PermissionOptionKind =
  | 'allow_once'
  | 'allow_always'
  | 'reject_once'
  | 'reject_always';
```

Client → Agent:

```typescript
interface RequestPermissionResponse {
  outcome: RequestPermissionOutcome;
}

type RequestPermissionOutcome =
  | { outcome: 'cancelled' }                          // 用户取消
  | { outcome: 'selected'; optionId: string; kind?: PermissionOptionKind };  // 用户选择
```

**执行流程**：
1. Agent 在工具执行前发送 `session/request_permission`
2. Agent 暂停执行，等待 Client 响应
3. Client 展示 UI，用户选择
4. Client 返回 `RequestPermissionResponse`
5. Agent 继续执行（批准 → 执行工具 / 拒绝 → 跳过 / 取消 → 中止）

**退出模式**：如果 Client 发送了 `session/cancel`，必须用 `outcome: 'cancelled'` 响应。

---

## 计划（Plan）

```typescript
interface Plan {
  entries: PlanEntry[];
}

interface PlanEntry {
  content: string;                              // 任务描述
  priority: 'high' | 'medium' | 'low';         // 优先级
  status: 'pending' | 'in_progress' | 'completed';  // 状态
}
```

**更新规则**：每次发送完整的 `entries` 列表，Client 替换整个计划显示。

---

## 其他 SessionUpdate 类型

### 配置选项更新

```typescript
interface ConfigOptionUpdate {
  configOptions: SessionConfigOption[];
}
```

### 可用命令更新

```typescript
interface AvailableCommandsUpdate {
  availableCommands: AvailableCommand[];
}
```

### 当前模式更新

```typescript
interface CurrentModeUpdate {
  currentModeId: string;
}
```

### 会话信息更新

```typescript
interface SessionInfoUpdate {
  title?: string | null;       // null 表示清空
  updatedAt?: string | null;   // ISO 8601
}
```

### 用量更新

```typescript
interface UsageUpdate {
  used: number;                // 已用 token
  size: number;                // 上下文窗口总大小
  cost?: {
    amount: number;            // 累计费用
    currency: string;          // ISO 4217
  } | null;
}
```

---

## Agent 接口（Agent 实现者视角）

```typescript
interface Agent {
  // === 必需方法 ===
  initialize(params: InitializeRequest): Promise<InitializeResponse>;
  newSession(params: NewSessionRequest): Promise<NewSessionResponse>;
  prompt(params: PromptRequest): Promise<PromptResponse>;
  cancel(params: CancelNotification): Promise<void>;
  authenticate(params: AuthenticateRequest): Promise<AuthenticateResponse | void>;

  // === 可选方法（按能力协商） ===
  loadSession?(params: LoadSessionRequest): Promise<LoadSessionResponse>;
  resumeSession?(params: ResumeSessionRequest): Promise<ResumeSessionResponse>;
  closeSession?(params: CloseSessionRequest): Promise<void>;
  listSessions?(params: ListSessionsRequest): Promise<ListSessionsResponse>;
  setSessionMode?(params: SetSessionModeRequest): Promise<void>;
  setSessionModel?(params: SetSessionModelRequest): Promise<void>;
  setSessionConfigOption?(params: SetSessionConfigOptionRequest): Promise<void>;

  // === 实验性方法（unstable_ 前缀） ===
  unstable_forkSession?(params: ForkSessionRequest): Promise<ForkSessionResponse>;
  unstable_listProviders?(params: ListProvidersRequest): Promise<ListProvidersResponse>;
  unstable_setProvider?(params: SetProvidersRequest): Promise<void>;
  unstable_disableProvider?(params: DisableProvidersRequest): Promise<void>;
  unstable_logout?(params: LogoutRequest): Promise<void>;

  // === NES（Next Edit Suggestions） ===
  unstable_startNes? / unstable_suggestNes? / unstable_closeNes? ...

  // === Document 同步 ===
  unstable_didOpenDocument? / didChangeDocument? / didCloseDocument? / didSaveDocument? ...

  // === 扩展点 ===
  extMethod?(method: string, params: Record<string, unknown>): Promise<Record<string, unknown>>;
  extNotification?(method: string, params: Record<string, unknown>): Promise<void>;
}
```

---

## 与 OpenAI/Anthropic 的关键差异

| 维度 | ACP | OpenAI / Anthropic |
|------|-----|-------------------|
| **协议** | JSON-RPC 2.0 双向 | HTTP 请求-响应 |
| **连接** | 长连接（stdio/WebSocket） | 短连接（每请求一个 HTTP） |
| **方向** | 双向：Agent ↔ Client 互发请求 | 单向：Client → Server |
| **Session** | 显式生命周期（new → prompt → close） | 无状态 |
| **流式输出** | `session/update` 通知（11 种变体） | SSE 文本流 |
| **权限** | 内置 `request_permission` 双向协议 | 无协议层支持 |
| **工具调用** | 完整生命周期：`tool_call` → `tool_call_update`（增量）→ `completed/failed` | 仅 `tool_calls` + `tool_result` |
| **类型生成** | `schema.json` → Zod + TypeScript | 手动 + 部分自动 |
| **能力协商** | `initialize` 阶段交换 `clientCapabilities` / `agentCapabilities` | 无 |
