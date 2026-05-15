# CC GUI — Agent 生命周期

## 两种执行模式

CC GUI 有 **Daemon（守护进程）** 和 **Per-Process（逐请求进程）** 两种模式。

## Daemon 模式

### 进程启动

1. `start()` 加 `synchronized(startLock)` 防并发
2. 解析 bridge 目录，确认 `daemon.js` 存在
3. `ProcessBuilder` 启动 `node daemon.js`
4. 启动 Reader 线程（stdout NDJSON）、Stderr 线程
5. 启动心跳线程（HeartbeatThread）
6. `CountDownLatch` 等待 daemon 发送 `{"type":"daemon","event":"ready"}`，**超时 30s**
7. daemon.js 预加载 Claude SDK → 发送 `sdk_loaded` 事件

### 心跳检测

- 每 **15s** 发送 `{"method":"heartbeat"}` 到 daemon
- 无活跃请求时：**45s**（3 个周期）无响应视为死亡
- 有活跃请求时：放宽到 **180s**
- `shouldTreatAsUnresponsive()` 综合判断

### 请求处理

```
Java → daemon stdin: {"id":"1","method":"claude.send","params":{...}}
  → 创建 RequestHandler → 存入 pendingRequests ConcurrentHashMap
  → Reader 线程解析 stdout → 按 id 分发到对应 handler
  → 收到 {"id":"1","done":true} 完成 CompletableFuture
  → whenComplete 清理 pendingRequests + activeRequestCount
```

### 自动重启

- `handleDaemonDeath()` 触发，最大重启 **3 次**
- 30s 稳定性窗口内运行后死亡 → 重置重试计数
- 重启前 force-kill 旧进程，所有 pending 请求注入错误

### 关闭

1. `isRunning = false`
2. 取消所有 pending 请求
3. 发送 `{"method":"shutdown"}` 命令
4. 关闭 stdin（触发 daemon readline 'close' 事件）
5. 若进程仍存活 → `destroyForcibly()` + `waitFor(3s)`
6. daemon.js 侧：5s 强制退出定时器 + `shutdownPersistentRuntimes()`

### daemon.js 侧生命周期

- 拦截 `process.stdout.write`、`console.log/error` → 包装为 NDJSON
- 拦截 `process.exit()` → daemon 模式下抛异常
- `commandQueue` (Promise 链) 串行化处理命令
- PPID 监控（10s 间隔）：父进程死亡则退出
- 支持 abort 命令：立即取消当前请求

## Per-Process 模式

```
BaseSDKBridge.executeStreamingCommand()
  → 每次请求启动新的 node channel-manager.js
  → stdin 写入 JSON，stdout 行读取
  → 子进程退出后判定成功/失败
```

## Daemon vs Per-Process

| 维度 | Daemon | Per-Process |
|------|--------|-------------|
| SDK 加载 | 启动时一次 | 每次请求 2-5s |
| 进程复用 | 一个长进程 | 每次新进程 |
| 心跳 | 15s | 无 |
| 自动重启 | 最多 3 次 | 无 |
| 会话状态 | 跨请求持久 | 每次新建 |
