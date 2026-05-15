# Proma — 工具调用与权限

## 架构特点

Agent 模式使用 **SDK canUseTool 回调 + Promise/Map 暂停机制**，Chat 模式使用 **tool_use 循环（最多 999 轮）**。权限规则基于**结构化命令解析**，不仅匹配命令名，还分析管道、重定向等 Bash 语法结构。

## Agent 工具调用流转路径

```
1. SDK 产生 tool_use → canUseTool(toolName, input, context) 回调触发
2. agent-permission-service.ts — 权限决策:
   a. mode === 'safe' → SAFE_TOOLS 匹配 → allow
   b. mode === 'ask' → 结构化危险检测 → 暂停流
   c. mode === 'allow-all' → 直接 allow
3. 需要用户决策 → Promise 暂存到 Map<toolUseID, PendingPermission>
   → AgentEventBus → IPC → 前端 PermissionRequest 组件
4. 用户选择 allow/deny/allow-all → IPC → resolve Permission Promis
5. tool_use 结果注入 MessageChannel → SDK 继续执行
```

### 关键代码位置

```typescript
// agent-permission-service.ts (13KB)
class AgentPermissionService {
  private pendingPermissions = new Map<string, PendingPermission>();

  async canUseTool(toolName, input, context): Promise<PermissionResult> {
    // 1. 安全工具白名单
    if (SAFE_TOOLS.has(toolName)) return { behavior: 'allow' };

    // 2. 危险命令检测
    if (toolName === 'Bash') {
      const risk = analyzeCommandRisk(input.command);
      if (risk.level === 'dangerous') {
        return await this.requestUserPermission(toolName, input, risk);
      }
    }

    // 3. 用户决策
    return await this.requestUserPermission(toolName, input);
  }
}
```

## 权限规则体系

### 三级权限模式

```typescript
type PermissionMode = 'safe' | 'ask' | 'allow-all';
// 与 Craft Agents OSS 保持一致
```

### 安全工具（始终允许）

```typescript
const SAFE_TOOLS = new Set([
  'Read', 'Glob', 'Grep',
  'WebSearch', 'WebFetch',
  'TodoRead', 'TodoWrite',
  'TaskOutput',
]);
```

### 安全 Bash 命令

```typescript
const SAFE_COMMANDS = [
  'git status', 'git log', 'git diff', 'git show',
  'git branch', 'git remote', 'git tag',
  'ls', 'head', 'tail', 'grep', 'rg',
  'which', 'pwd', 'env', 'whoami', 'uname',
  'tree', 'wc', 'file', 'stat', 'du', 'df',
  // 版本检查 ...
];
```

### 危险命令

```typescript
const DANGEROUS_COMMANDS = [
  'rm', 'rmdir', 'sudo', 'su',
  'chmod', 'chown', 'mv', 'dd', 'kill',
  'git push', 'git reset', 'git rebase',
  'git checkout', 'git clean', 'git branch -D',
  'npm publish', 'curl', 'wget', 'ssh', 'scp',
];
```

### 结构化危险检测（Proma 独有）

```typescript
function analyzeCommandRisk(command: string): RiskAssessment {
  // 不仅匹配命令名，还分析 Bash 语法结构:
  const risks: RiskPattern[] = [];

  // 1. 管道检测
  if (command.includes('|')) {
    risks.push({ pattern: 'pipe', severity: 'medium' });
  }

  // 2. 输出重定向
  if (/>[>\s]/.test(command)) {
    risks.push({ pattern: 'redirect', severity: 'high' });
  }

  // 3. find 危险用法
  if (command.includes('find') && (command.includes('-exec') || command.includes('-delete'))) {
    risks.push({ pattern: 'find_exec', severity: 'critical' });
  }

  // 4. 命令链
  if (command.includes('&&') || command.includes(';')) {
    risks.push({ pattern: 'command_chain', severity: 'medium' });
  }

  // 5. 子shell
  if (/\$\(/.test(command) || /`/.test(command)) {
    risks.push({ pattern: 'subshell', severity: 'high' });
  }

  // 综合评估
  const severity = risks.some(r => r.severity === 'critical') ? 'dangerous'
    : risks.length >= 2 ? 'suspicious'
    : risks.length === 1 ? 'caution'
    : 'safe';

  return { severity, risks };
}
```

## 权限记忆

### Session 级白名单

```typescript
// agent-permission-service.ts
class AgentPermissionService {
  private toolWhitelist = new Map<string, Set<string>>();  // sessionId → toolName[]
  private commandWhitelist = new Map<string, Set<string>>(); // sessionId → command[]

  // 用户选择 "Always Allow" → 加入白名单
  // 同一 session 内后续调用自动通过
  // session 关闭后白名单自动清除
}
```

### 前端权限队列

```typescript
// agent-atoms.ts
// 按 sessionId 分组的权限请求队列
allPendingPermissionRequestsAtom: Map<string, PermissionRequest[]>
allPendingAskUserRequestsAtom: Map<string, AskUserRequest[]>

// 跨 session 权限请求互不干扰
// 切换 tab 时只显示当前 session 的权限请求
```

## Chat 工具调用

### 工具执行循环

```typescript
// chat-service.ts
async function runChatWithTools(conversationId, messages) {
  let round = 0;
  const MAX_ROUNDS = 999;

  while (round < MAX_ROUNDS) {
    const response = await fetchSSE(messages);
    if (response.stopReason !== 'tool_use') break;

    // 执行工具 → 结果加入消息列表 → 继续
    for (const toolCall of response.toolCalls) {
      const result = await chatToolExecutor.execute(toolCall);
      messages.push({ role: 'tool', content: result, toolCallId });
    }
    round++;
  }
}
```

### Chat 内置工具

```typescript
// chat-tool-registry.ts + chat-tool-executor.ts
const builtInTools = {
  memory:       MemOSTool,        // MemOS 记忆集成
  web_search:   WebSearchTool,    // 网页搜索
  recommend:    AgentRecommendTool, // Agent 推荐
  nano_banana:  ImageGenTool,     // 图像生成 (MCP)
};

// 自定义工具 — 用户配置的 HTTP 工具
// 注册为函数定义注入 Chat API 请求
```

## 权限消息类型

| IPC Channel | 说明 |
|-------------|------|
| `AGENT_IPC_CHANNELS.PERMISSION_REQUEST` | 工具权限请求 |
| `AGENT_IPC_CHANNELS.ASK_USER_REQUEST` | AskUserQuestion 请求 |
| `AGENT_IPC_CHANNELS.EXIT_PLAN_REQUEST` | 计划退出请求 |
| `AGENT_IPC_CHANNELS.PERMISSION_RESPONSE` | 用户决策返回 |

## 权限策略配置对比

| 策略 | Agent 模式 | Chat 模式 |
|------|-----------|-----------|
| 全部批准 | `allow-all` 模式 | N/A（Chat 工具由模型决定） |
| 只读 + 安全 Bash | `safe` 模式（默认） | N/A |
| 每次询问 | `ask` 模式 | N/A |
| 结构化检测 | ✓ 管道/重定向/子shell | — |
| Session 白名单 | ✓ toolName + command | — |
| Promise 暂停 | ✓ Map<toolUseID, Promise> | — |
