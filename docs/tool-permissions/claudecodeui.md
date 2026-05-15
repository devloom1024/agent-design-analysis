# Claude Code UI — 工具调用与权限

## 架构特点

使用 **WebSocket 直连模式**——权限请求从 SDK 回调 → WebSocket → 前端审批 → WebSocket → SDK。

```javascript
canUseTool(toolName, input, context) → Promise<{ behavior, updatedInput?, message? }>
```

## 完整流转路径（Claude SDK）

```
1. queryClaudeSDK() 设置 canUseTool 回调
2. SDK 触发 canUseTool(toolName, input, context):
   a. 检查 disallowedTools → 匹配则 deny
   b. 检查 allowedTools → 匹配则 allow
   c. 交互工具(AskUserQuestion/ExitPlanMode) → waitForToolApproval(0) 无限超时
   d. 其他 → waitForToolApproval(TOOL_APPROVAL_TIMEOUT_MS)
3. 发送 permission_request WebSocket 消息到前端
4. 前端用户审批 → 发送 claude-permission-response
5. resolveToolApproval(requestId, decision) → Promise resolved
6. 返回 { behavior: 'allow', updatedInput }
```

## 权限匹配逻辑

```javascript
function matchesToolPermission(entry, toolName, input) {
  // 精确匹配工具名
  if (entry === toolName) return true;
  // Bash 命令通配符：Bash(command:*)
  const bashMatch = entry.match(/^Bash\((.+):\*\)$/);
  if (bashMatch) return input.command?.startsWith(bashMatch[1]);
  return false;
}
```

## pendingToolApprovals 机制

```javascript
const pendingToolApprovals = new Map();  // requestId → { resolve, timeout }

function waitForToolApproval(requestId, options):
  return new Promise(resolve => {
    if (timeoutMs > 0) {
      // 设置超时 → 自动 deny + permission_cancelled 事件
    }
    pendingToolApprovals.set(requestId, { resolve, timeout });
  });
```

## 四种 Provider 权限模式

### Claude SDK

| 模式 | 配置 | 行为 |
|------|------|------|
| default | `permissionMode: 'default'` | canUseTool 正常回调 |
| bypassPermissions | `permissionMode: 'bypassPermissions'` | 跳过所有权限 |
| plan | `permissionMode: 'plan'` | 只允许 Read/Task/exit_plan_mode 等 |
| acceptEdits | `permissionMode: 'acceptEdits'` | 允许编辑，其他需审批 |
| 白名单 | `allowedTools: [...]` | 列表内自动允许 |
| 黑名单 | `disallowedTools: [...]` | 列表内自动拒绝 |

### Codex

```javascript
// 映射关系
'default'            → { sandboxMode: 'workspace-write', approvalPolicy: 'untrusted' }
'acceptEdits'        → { sandboxMode: 'workspace-write', approvalPolicy: 'never' }
'bypassPermissions'  → { sandboxMode: 'danger-full-access', approvalPolicy: 'never' }
```

### Gemini

```
--approval-mode 参数控制
--yolo → 跳过所有权限
```

### Cursor

```
--approval-mode 参数
-f / --force 跳过权限
```

## 交互工具特殊处理

```javascript
const INTERACTIVE_TOOLS = ['AskUserQuestion', 'ExitPlanMode'];
// timeoutMs = 0 → 无限等待用户响应
```

- **AskUserQuestion**: 显示问题表单，返回用户答案
- **ExitPlanMode**: 等待用户批准计划

## 权限消息类型

| WebSocket kind | 说明 |
|---------------|------|
| `permission_request` | 发起权限请求 |
| `permission_cancelled` | 权限请求取消/超时 |
| `permission_response` | 用户决策（allow/deny + updatedInput） |
