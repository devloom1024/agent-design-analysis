# Claude Code UI — Agent 生命周期

## Claude SDK 模式

### 会话状态

无显式状态枚举，通过 `activeSessions` Map 追踪：

```javascript
activeSessions.set(sessionId, {
  instance: queryInstance,  // SDK 查询实例
  startTime: Date.now(),
  status: 'active',         // 'active' | 'aborted'
  tempImagePaths,
  tempDir,
  writer                   // WebSocket 写入器
});
```

### 完整时序

```
1. 客户端 WebSocket 连接
2. 客户端发送 { command, options: { sessionId, model, ... } }
3. queryClaudeSDK():
   a. mapCliOptionsToSDK() — 映射 CLI 选项到 SDK 格式
   b. 加载 MCP 配置（global + project）
   c. 处理图片附件（base64 → 临时文件）
   d. 设置 hooks.Notification
   e. 设置 canUseTool 权限回调
   f. query({ prompt, options }) 创建 SDK 查询实例
   g. addSession() 注册到 activeSessions
4. for await (const message of queryInstance):
   a. 捕获首个消息的 session_id
   b. sessionsService.normalizeMessage() → NormalizedMessage[]
   c. WebSocket 发送给前端
   d. result 消息中提取 tokenBudget
5. finally:
   a. removeSession()
   b. 清理临时文件
   c. 发送 complete 事件 + notifyRunStopped
6. 异常时: 发送 error 事件 + notifyRunFailed
```

### 中止流程

```javascript
abortClaudeSDKSession(sessionId):
  1. activeSessions.get(sessionId)
  2. session.instance.interrupt()  // SDK 中止
  3. session.status = 'aborted'
  4. 清理临时文件、removeSession
```

### WebSocket 重连

```javascript
reconnectSessionWriter(sessionId, newRawWs):
  1. activeSessions.get(sessionId)
  2. session.writer.updateWebSocket(newRawWs)  // 替换写入器
```

### Tool Approval 机制

- `pendingToolApprovals` Map 存储待处理权限
- `waitForToolApproval()` 返回 Promise
- 默认超时 **55s**，交互工具（AskUserQuestion）设为 **0**（无限等待）
- `resolveToolApproval(requestId, decision)` 完成 Promise

## Cursor CLI 子进程模式

### 生命周期

```javascript
spawnCursor(command, options, writer):
  1. cross-spawn 启动 cursor-agent 子进程
  2. activeCursorProcesses.set(processKey, childProcess)
  3. stdout 流式处理：
     - type='system' → 捕获 session_id → session_created
     - type='assistant' → normalizeMessage → WebSocket
     - type='result' → complete 事件
  4. stderr → error 事件
  5. Workspace Trust 检测: 自动追加 --trust 重试
```

### 中止

```javascript
abortCursorSession(processKey):
  process.kill('SIGTERM')
  activeCursorProcesses.delete(processKey)
```

## Session 同步服务

- 所有 provider 并行 `synchronize()`（`Promise.allSettled`）
- 成功则更新 `scanStateDb` 的扫描时间戳
- 支持单文件同步（`synchronizeProviderFile()`）
