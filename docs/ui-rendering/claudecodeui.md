# Claude Code UI — UI 渲染层

## 架构特点

Web 应用 + WebSocket 直连 Claude SDK。流式渲染采用 **WebSocket 100ms flush + react-markdown 重新解析** 策略。

## 技术栈

| 层级 | 技术 |
|------|------|
| UI 框架 | React 18 + TypeScript |
| 样式方案 | Tailwind CSS 3 |
| Markdown 解析 | `react-markdown` |
| 代码高亮 | `react-syntax-highlighter` (Prism) |
| 通信协议 | WebSocket |
| 状态管理 | `useSessionStore` (Map-based) |

## 流式渲染核心机制

### useChatRealtimeHandlers.ts — 实时消息处理

```typescript
// 核心 Hook
function useChatRealtimeHandlers() {
  // 消息缓冲 + 定时刷新
  const messageBuffer = useRef<NormalizedMessage[]>([]);

  // 100ms 刷新间隔
  const FLUSH_INTERVAL = 100;

  useEffect(() => {
    const interval = setInterval(() => {
      if (messageBuffer.current.length > 0) {
        // 合并所有缓冲消息到 sessionStore
        flushMessages(messageBuffer.current);
        messageBuffer.current = [];
      }
    }, FLUSH_INTERVAL);
    return () => clearInterval(interval);
  }, []);
}
```

### 完整流式流程

```
WebSocket message → messageBuffer (累积 100ms)
  → flushMessages() → sessionStore Map 更新
    → ChatMessage 组件 re-render
      → react-markdown 重新解析全文
        → react-syntax-highlighter Prism 高亮代码块
```

### 消息渲染通道

```typescript
// MessageKind 分发渲染
const MessageRenderer = ({ message }: { message: NormalizedMessage }) => {
  switch (message.kind) {
    case 'text':        return <TextMessage />;
    case 'reasoning':   return <Reasoning />;
    case 'tool_use':    return <ToolRenderer />;
    case 'tool_result': return <ToolResultMessage />;
    case 'user':        return <UserMessage />;
    case 'error':       return <ErrorMessage />;
    case 'system':      return <SystemMessage />;
    // ... 14 种消息类型
  }
};
```

## 思考/推理块渲染

### Reasoning 组件

```typescript
// Reasoning.tsx — 可折叠思考过程展示
const Reasoning = ({ content, isStreaming }) => {
  // 状态
  const [isExpanded, setIsExpanded] = useState(false);

  // 流式时自动展开
  useEffect(() => {
    if (isStreaming) setIsExpanded(true);
  }, [isStreaming]);

  return (
    <details open={isExpanded || isStreaming}>
      <summary>Thinking {isStreaming && '...'}</summary>
      <div className="reasoning-content">
        <Markdown>{content}</Markdown>
      </div>
    </details>
  );
};
```

### ReasoningTrigger 组件

```typescript
// 推理触发标识 — 显示 "Thinking..." 占位符
// 收到 reasoning 内容后自动替换为 Reasoning 组件
const ReasoningTrigger = () => (
  <div className="thinking-indicator animate-pulse">
    Thinking...
  </div>
);
```

## 工具调用渲染

### ToolRenderer 组件

```typescript
const ToolRenderer = ({ toolCall }) => {
  // 根据工具类型匹配配置
  const config = toolConfigs[toolCall.toolName];

  return (
    <div className="tool-call-card">
      <ToolHeader icon={config.icon} name={config.displayName} />
      <ToolInput input={toolCall.input} schema={config.inputSchema} />
      <ToolStatus status={toolCall.status} />
      {toolCall.result && <ToolOutput result={toolCall.result} />}
    </div>
  );
};
```

### 工具配置注册表

```typescript
const toolConfigs = {
  Read:       { icon: FileIcon, displayName: 'Read file', inputSchema: {...} },
  Write:      { icon: EditIcon, displayName: 'Write file', inputSchema: {...} },
  Edit:       { icon: PencilIcon, displayName: 'Edit file', inputSchema: {...} },
  Bash:       { icon: TerminalIcon, displayName: 'Run command', inputSchema: {...} },
  Grep:       { icon: SearchIcon, displayName: 'Search', inputSchema: {...} },
  // ... 更多工具
};
```

## 权限 UI 组件

### PermissionRequestsBanner

```typescript
const PermissionRequestsBanner = ({ requests }) => {
  return (
    <div className="permission-banner">
      {requests.map(req => (
        <PermissionRequestCard
          key={req.requestId}
          toolName={req.toolName}
          input={req.input}
          onAllow={() => resolvePermission(req.requestId, 'allow')}
          onDeny={() => resolvePermission(req.requestId, 'deny')}
        />
      ))}
    </div>
  );
};
```

### AskUserQuestion 交互

```typescript
// 交互工具特殊处理：无限等待
// AskUserQuestion → 表单展示问题 + 选项
// ExitPlanMode → 展示计划 + 批准按钮
// timeoutMs = 0 → 永不超时
```

## 历史消息（Session）渲染

### useSessionStore — 状态管理

```typescript
const useSessionStore = create((set, get) => ({
  // 所有消息存储
  messages: new Map<string, NormalizedMessage>(),

  // 当前会话消息 ID 列表（有序）
  currentSessionMessageIds: [],

  // 分页状态
  pagination: {
    currentPage: 1,
    pageSize: 20,
    hasMore: true,
  },

  loadChatHistory: async (sessionId) => {
    // 全量加载消息到 Map
    // 设置分页起始状态
  },

  loadMoreMessages: () => {
    // 滚动触发 → currentPage++ → 渲染更多消息
  },
}));
```

### 滚动分页

```
初始加载 → 显示最近 20 条消息
  ↓
用户向上滚动到顶部
  ↓
loadMoreMessages() → currentPage++ → 显示下 20 条
  ↓
hasMore = false → 显示 "Beginning of conversation"
```

### 消息列表虚拟化

不使用虚拟滚动库，通过**分页加载 + DOM 复用**实现类虚拟化效果：
- 每次只渲染 20 条消息
- 滚动到顶部加载更多
- 旧消息保持在 DOM 中（总消息量有限）

## 渲染优化

- **100ms flush 缓冲** — 避免每条消息都触发渲染
- **React.memo** — `ChatMessage` 组件 memo 化
- **useCallback** — 事件处理器稳定引用
- **Tailwind JIT** — 仅生成使用的 CSS
