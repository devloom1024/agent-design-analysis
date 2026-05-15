# acpx — Agent 生命周期

## ACP 客户端生命周期

### AcpClient 关键属性

```typescript
connection?: ClientSideConnection
agent?: ChildProcess
initResult?: InitializeResponse
loadedSessionId?: string
activePrompt?: { sessionId, promise }
closing: boolean
agentStartedAt?: string
lastAgentExit?: AgentExitInfo
```

### start() 流程

```
1. 已有连接且进程存活 → 直接返回
2. 解析 agent 命令（resolveBuiltInAgentLaunch）
3. 特殊处理：Gemini 参数、Qoder 参数
4. 设置环境变量 → spawn() 子进程
5. attachAgentLifecycleObservers
6. 创建 NDJSON 消息流
7. ClientSideConnection 建立协议层
   - 注册 sessionUpdate、requestPermission、文件读写、终端管理回调
8. Promise.race(initialize vs 启动失败监控)
9. authenticateIfRequired
```

### createSession() 流程

```
1. 获取 ClientSideConnection
2. 构建 session/new 请求（cwd、MCP servers、元数据）
3. Claude ACP 独立超时处理
4. 返回 { sessionId, agentSessionId, configOptions, models }
```

### prompt() 流程

```
1. connection.prompt() 发送 session/prompt
2. activePrompt = { sessionId, promise }
3. 检测权限失败（consumePromptPermissionFailure）
4. finally: 清理 activePrompt 和 cancellingSessionIds
```

### closeSession() 流程

```
1. connection.closeSession({ sessionId })
2. 清空 loadedSessionId
3. 不支持的 agent → 抛 ACP_BACKEND_UNSUPPORTED_CONTROL
```

### close() 三阶段优雅关闭

```
Phase 1 (< stdinCloseGraceMs): 关闭 stdin → 等待自然退出
Phase 2 (< 1500ms): SIGTERM
Phase 3 (< 1000ms): SIGKILL
→ 分离 stdio 句柄
→ 拒绝所有 pending 请求
→ 重置所有状态
```

## AcpRuntimeManager 生命周期

### ensureSession() 流程

```
1. 检查是否可重用（shouldReuseExistingRecord）
2. 创建 AcpClient → client.start()
3. resumeSessionId ? client.createSession() : client.loadSession()
4. 创建 SessionRecord → sessionStore.save()
5. Persistent 模式缓存到 pendingPersistentClients
```

### startTurn() 流程

```
1. sessionStore.load() 加载 SessionRecord
2. 克隆 conversation，提交 prompt
3. 获取 pending client 或新建
4. connectAndLoadSession() 连接/加载
   - 进程存活 → 直接重用
   - 进程死亡 → client.start()
   - load 失败 → createSession + replay（回放 mode/model/config）
5. runPromptTurn() 发送 prompt
6. 流式响应 → AsyncEventQueue → AcpRuntimeEvent 异步迭代器
7. 完成: 更新 lifecycle 快照、保存 record、缓存/关闭 client
```

### close() 流程

```
1. 取消当前 prompt
2. discardPersistentState=true:
   → closeBackendSession (session/close)
   → reset_on_next_ensure
3. discardPersistentState=false:
   → 仅关闭 pending client
4. record.closed = true → save
```

### 重连逻辑

```
connectAndLoadSession():
  1. isProcessAlive(record.pid) → 重用
  2. client.loadSessionWithOptions()
  3. 失败 → shouldFallbackToNewSession()
     → client.createSession()
     → replayDesiredMode/Model/ConfigOptions()
  4. same-session-only → 失败抛 SessionResumeRequiredError
```

## ACP 协议完整生命周期

```
initialize
  → session/new (或 session/load)
  → [session/set_mode / session/set_model / session/set_config_option]
  → session/prompt
  → session/update (0..N 次流式通知)
  → [session/cancel]
  → session/close
```
