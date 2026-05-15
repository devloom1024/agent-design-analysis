# Proma — Agent 生命周期

## 架构特点

Proma 的 Agent 模式使用 **SDK 子进程 + JSONL 持久化**，Chat 模式使用 **fetch SSE + AbortController**。两个模式完全独立的生命周期管理。

## Agent 会话状态

### 运行时状态（隐式）

```typescript
// agent-orchestrator.ts — 无显式状态枚举
// 通过以下信号隐式表达状态:
activeSessions: Map<string, Promise<void>>  // 活跃请求防并发
running: boolean                             // agent-atoms 中的流式状态
compacting: boolean                          // 上下文压缩中
retryState: { count, maxRetries }           // 重试状态
```

### AgentStreamState — 前端状态

```
idle → running → compacting? → done
  │                 │
  └── retrying ←────┘
```

### 并发控制

```typescript
// agent-orchestrator.ts
const activeSessions = new Map<string, Promise<void>>();

async function runAgent(sessionId: string, input: string) {
  if (activeSessions.has(sessionId)) {
    throw new Error('Session is already running');
  }
  const promise = executeAgent(sessionId, input);
  activeSessions.set(sessionId, promise);
  await promise;
  activeSessions.delete(sessionId);
}
```

同一 session 不允许并行请求，但不同 session 可同时运行。

## 进程生命周期

### Agent 模式启动流程

```
1. 用户选择 Channel (provider + model) 和 Workspace
2. agent-session-manager.createSession():
   a. 生成 sessionId
   b. 创建工作目录 ~/.proma/agent-workspaces/{slug}/{sessionId}/
   c. 初始化 JSONL 消息文件
3. agent-orchestrator.runAgent():
   a. resolveSDKCliPath() — 解析 SDK 二进制路径
   b. 解密 API Key (Electron safeStorage)
   c. 构建环境变量 + SDK 选项
4. claude-agent-adapter.query():
   a. dynamic import('@anthropic-ai/claude-agent-sdk')
   b. SDK query() → MessageChannel (AsyncGenerator)
   c. 子进程 claude 二进制启动
   d. canUseTool 回调注册 — 权限暂停/恢复
5. 迭代 SDKMessage stream → convertSDKMessage() → AgentEventBus
```

### Agent 模式关闭流程

```
1. 用户取消 / 会话结束 → AbortSignal 触发
2. channel.close() → stdin 关闭 → 子进程自然退出
3. activeSessions.delete(sessionId)
4. 消息持久化到 JSONL
5. 错误时: 部分内容保存 + 自动重试
```

### Chat 模式启动流程

```
1. conversation-manager.createConversation()
2. 用户发送消息 → chat-service.streamChat():
   a. 创建 AbortController → 存入 activeControllers Map
   b. fetch POST → Provider API
   c. sse-reader.ts 消费 ReadableStream
   d. 逐行 data: 解析 → 适配器 parseSSELine()
   e. webContents.send(STREAM_EVENT) → 渲染进程
3. 完成 / 中断 → AbortController 清理
```

## 重连与恢复

### SDK Session Resume

```typescript
// agent-orchestrator.ts
async function resumeSession(sessionId: string) {
  // 1. 尝试 SDK resume
  const resumed = await adapter.query({
    resume: sessionId,
    // ... 其他选项
  });

  // 2. resume 成功 → 继续对话
  if (resumed) return resumed;

  // 3. resume 失败 → 后备方案
  // 原因: session 过期、thinking signature 不兼容
  const recentMessages = await loadRecentMessages(sessionId);

  // 3a. buildRecoveryPrompt() — 注入为 conversation_history
  // 3b. buildContextPrompt() — 注入为 session_recovery block
  return await adapter.query({
    systemPrompt: buildRecoveryPrompt(recentMessages),
    // 全新 session
  });
}
```

**后备上下文恢复策略**：
- `<conversation_history>` → 注入最近 N 条消息摘要
- `<session_recovery>` → 注入文件变更状态信息
- 保留 workspace 文件完整性（文件不回滚）

### 自动重试

```typescript
// agent-orchestrator.ts
const MAX_RETRIES = 25;
const RETRY_BUDGET_MS = 5 * 60 * 1000;  // 5 分钟预算

async function executeWithRetry(sessionId, input) {
  let retries = 0;
  const startTime = Date.now();

  while (retries < MAX_RETRIES) {
    try {
      return await runAgent(sessionId, input);
    } catch (error) {
      if (!isRetryable(error)) throw error;
      if (Date.now() - startTime > RETRY_BUDGET_MS) throw error;

      retries++;
      await sleep(exponentialBackoff(retries));  // 指数退避
    }
  }
}
```

## 独特的 Session 操作

### forkSession() — 会话分叉

```typescript
// agent-session-manager.ts (51KB)
async function forkSession(sessionId: string, fromMessageId: string) {
  // 从指定消息 UUID 处创建分支
  // 1. 复制原 session 的消息（直到 fromMessageId）
  // 2. 创建新 SDK session
  // 3. 文件快照 — 保留分支前的 workspace 文件状态
  return newSessionId;
}
```

### rewindSession() — 会话回退

```typescript
async function rewindSession(sessionId: string, toMessageId: string) {
  // 回退文件和消息到指定位置
  // 1. 利用 SDK 文件检查点回滚文件变更
  // 2. 截断 JSONL 消息到 toMessageId
  // 3. 后续消息保留在分支中（不丢失）
}
```

### 上下文压缩

```typescript
// 当 context window 接近上限时:
// 1. compacting = true — 前端显示压缩指示器
// 2. SDK 自动压缩上下文
// 3. compacting = false — 正常继续
// 4. 压缩后的消息以 CompressedGroup 形式显示
```

## Chat 会话生命周期

```
createConversation()
  → sendMessage()
    → AbortController 创建
      → fetch SSE → 流式接收
        → stopReason === 'end_turn' → 对话等待下一条消息
        → stopReason === 'tool_use' → 执行工具 → 继续 SSE 流
        → 用户取消 → AbortController.abort()
  → deleteConversation() → 清理 JSONL 文件
```

## 进程启动/关闭对比

| 维度 | Agent 模式 | Chat 模式 |
|------|-----------|-----------|
| 启动方式 | `dynamic import` SDK → `query()` → child_process | `fetch` → SSE ReadableStream |
| 超时 | SDK 默认 | fetch 默认 |
| 关闭 | AbortSignal → channel.close() → 子进程退出 | AbortController.abort() |
| 优雅关闭 | stdin 关闭 + 等待子进程退出 | 即时中断 |
| 重试 | 指数退避，最多 25 次，5 分钟预算 | 用户手动重试 |
