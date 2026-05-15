# AionUi — Agent 生命周期

## 七态形式化状态机（AcpSession）

```
idle → starting → active → prompting → active → suspended → resuming → active → ...
  ↓                ↓                           ↓
error ←────────────┴───────────────────────────┘
```

### 状态转换矩阵

```typescript
const VALID_TRANSITIONS = {
  idle:       ['starting'],
  starting:   ['active', 'starting', 'error', 'idle'],
  active:     ['prompting', 'suspended', 'idle'],
  prompting:  ['active', 'resuming', 'error', 'idle'],
  suspended:  ['resuming', 'idle'],
  resuming:   ['active', 'resuming', 'error', 'idle'],
  error:      ['starting', 'idle'],
};
```

### AcpSession 核心组件

| 组件 | 职责 |
|------|------|
| `configTracker` | 追踪 model/mode/config 的 desired vs current |
| `messageTranslator` | 协议消息 → UI 消息翻译 |
| `permissionResolver` | 权限请求评估/缓存/自动批准 |
| `inputPreprocessor` | 用户输入预处理（文件引用解析） |
| `lifecycle` | 进程启动/协议初始化/会话建立 |
| `promptExecutor` | 提示执行（超时 300s） |

### 状态转换方法

```
start():        idle/error → lifecycle.start() → active
stop():         停止计时器 → 拒绝 pending 权限 → 清理 → idle
suspend():      active → lifecycle.teardown() → suspended
sendMessage():  active → promptExecutor.execute()
                 suspended → 缓存 prompt → lifecycle.resume()
                 其他 → INVALID_STATE 错误
cancelPrompt(): 停止计时器 + 拒绝权限 + 取消 prompt
```

### onDisconnect() 状态分流

| 断连时状态 | 行为 |
|-----------|------|
| `prompting` | 清空 client → 发送 crash 信号 → resumeFromDisconnect() 自动重启 |
| `active`（后台超时） | 静默进入 `suspended`，不发 crash 信号 |
| `starting`/`resuming` | 发送 crash 信号 → `suspended` |

## SessionLifecycle 进程生命周期

### doStart() 流程

```
1. status = 'starting'
2. spawnAndInit():
   a. 创建 AcpClient
   b. 注册 disconnect handler
   c. client.start() → ACP initialize
3. establishSession():
   a. 有 sessionId → tryLoadOrCreate() (优先 load)
   b. 无 sessionId → client.createSession()
4. 遇到 AUTH_REQUIRED → authPending + teardown + auth_required 信号
5. applySessionResult() → 同步 config/model/mode → status = 'active'
6. yoloMode → 自动设置全自动模式
7. reassertConfig() → 应用 pending 更改
```

### 重试策略

- 最多 `maxStartRetries`（默认 **3 次**）
- 指数退避：**1s, 2s, 4s**
- 重试前清理 bunx 缓存
- 耗尽重试 → enterError()

### resume() 流程

```
1. status = 'resuming'
2. 重新 spawnAndInit()
3. tryLoadOrCreate()
4. reassertConfig()
5. status = 'active'
6. flushPendingPrompt()
```

### teardown()

```
client.close() → 清空 _client
```

## AcpConnection 连接层

### connect() 流程

```
1. 根据 backend 类型选择连接器
   (connectClaude / connectCodebuddy / connectCodex / connectGenericBackend)
2. setupChildProcessHandlers():
   - 收集 stderr（head 512 + tail 1536）
   - 监听 error（ENOENT）
   - 监听 exit（启动/运行阶段分流）
   - stdin NDJSON 解析
3. initialize() 发送 initialize 请求
   超时: 60s，竞争 processExitPromise
4. isSetupComplete = true
```

### sendPrompt() 流程

```
1. 发送 session/prompt 请求
2. lastPromptSentAt → TTFC 追踪
3. 超时管理:
   - session/prompt: promptTimeoutMs (默认 300s, 最小 30s)
   - 其他方法: 60s
   - 超时 → session/cancel（不杀进程）
4. 流式更新: 首帧记录 TTFC，每次更新重置超时
5. 权限请求期间: 暂停所有超时计时器
```

### disconnect() 流程

```
1. agent 声明支持 session/close → 尝试 session/close (2s 超时)
2. isSetupComplete = false (防止 exit 被误认为 crash)
3. terminateChild()
4. 重置所有状态属性
```

## AcpRuntime 空闲回收

```
IdleReclaimer 每 30s 扫描:
  active 状态 + 超过 idleTimeoutMs (默认 5min) → auto suspend()
```

## 完整用户交互流程

```
用户创建对话 → createConversation()
  → 浅克隆 AgentConfig
  → 注入 team-guide MCP server
  → 加载用户配置的 MCP servers
  → 创建 AcpSession + 注册回调
  → session.start()
  → 存入 sessions Map
用户发送消息 → session.sendMessage(text)
  → lifecycle 确保进程存活 + 会话建立
  → promptExecutor 执行
  → ACP session/prompt → 流式 session/update
  → AcpAdapter 转换为 UI 消息
  → IPC/WebSocket 发送到前端
空闲超时 → IdleReclaimer → session.suspend()
用户再次发送 → session.resume() → 重新建立连接
shutdown → 所有 active/prompting session → suspend()
```
