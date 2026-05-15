# AionUi — 工具调用与权限

## 架构特点

ACP 协议回调 + ApprovalStore 记忆缓存 + 前端弹窗。

## 工具调用完整生命周期

```
声明 → 执行 → 状态更新 → 完成/失败 → 清理
```

### 阶段一：ACP 消息接收

```
Agent Stdio → JSON-RPC 解析 → 按 method 分发:
  - session/update → sessionUpdate 联合类型匹配:
    → 'tool_call'         → AcpAdapter.createOrUpdateAcpToolCall()
    → 'tool_call_update'  → AcpAdapter.updateAcpToolCall()
  - session/request_permission → handlePermissionRequest()
```

### 阶段二：消息转换（AcpAdapter）

| sessionUpdate 值 | 方法 | 行为 |
|-----------------|------|------|
| `tool_call` | `createOrUpdateAcpToolCall()` | 创建 `IMessageAcpToolCall`，存入 `activeToolCalls` Map |
| `tool_call_update` | `updateAcpToolCall()` | 按 toolCallId 查找，更新 status/content/rawInput |
| `plan` | `createOrUpdatePlan()` | 按 sessionId 合并到已有 plan 消息 |

### 阶段三：状态跟踪

```typescript
activeToolCalls: Map<string, IMessageAcpToolCall>
// 按 toolCallId 索引，60s 后清理 completed/failed 的工具
```

### 阶段四：UI 合并渲染

`composeMessage()` 按 `msg_id` / `toolCallId` / `sessionId` 将同一工具的多次更新合并到单条 UI 消息。

## 权限生命周期

```
1. ACP 'session/request_permission' → AcpConnection.handlePermissionRequest()
2. 暂停超时计时器
3. AcpAgent.handlePermissionRequest():
   a. 确保 toolCallId 存在（不存在则生成 UUID）
   b. 检查 ApprovalStore 缓存 (isApprovedForSession) → 自动 allow_always
   c. 存储 permissionRequestMeta (kind, title, rawInput)
   d. 检查 NavigationInterceptor (chrome-devtools 导航工具)
   e. pendingPermissions Map 注册 Promise
   f. 发射 permissionRequest 事件到 UI
   g. 超时: 团队模式无限 / 独立模式 30 分钟
4. UI 用户选择 optionId → Promise resolve
5. 返回 ACP { outcome: 'selected'|'rejected', optionId }
6. 恢复超时计时器
```

## ApprovalStore 记忆缓存

```typescript
class AcpApprovalStore {
  private map: Map<string, string> = new Map();
  
  get(key): optionId | undefined;
  put(key, optionId): void;  // 只缓存 'allow_always'
  isApprovedForSession(key): boolean;
}
```

### 缓存键序列化

```typescript
function serializeKey(key: AcpApprovalKey): string {
  // 只取操作标识字段（command, path, file_path）
  // 忽略描述等元数据
  normalizedInput = { command, path, file_path }
  return JSON.stringify({ kind, title, rawInput: normalizedInput });
}
```

## 权限选项类型

```typescript
interface AcpPermissionOption {
  optionId: string;
  name: string;
  kind: 'allow_once' | 'allow_always' | 'reject_once' | 'reject_always';
}
```

## 工具调用状态枚举

```typescript
status: 'pending' | 'in_progress' | 'completed' | 'failed'
```

## 工具种类

```typescript
kind: 'read' | 'edit' | 'execute'
```

## ToolCallUpdate 结构

```typescript
interface ToolCallUpdate {
  toolCallId: string;
  status: 'pending' | 'in_progress' | 'completed' | 'failed';
  title: string;
  kind: 'read' | 'edit' | 'execute';
  rawInput?: Record<string, unknown>;
  content?: ToolCallContentItem[];
  locations?: ToolCallLocationItem[];
}
```

## YOLO 模式

```typescript
// yoloMode → 自动设置全自动批准模式
// 在 applySessionResult() 中处理
```

## 权限消息的前端类型

| TMessage type | 说明 |
|--------------|------|
| `acp_permission` | ACP 权限请求弹窗 |
| `acp_tool_call` | ACP 工具调用卡片（含状态更新） |
| `codex_permission` | Codex 特有权限请求 |
| `codex_tool_call` | Codex 特有工具调用 |
| `tool_group` | 多个工具聚合到一个 UI 分组 |
