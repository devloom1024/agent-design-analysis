# AionUi — UI 渲染层

## 架构特点

Electron 桌面应用，采用 **IPC 消息通道 + composeMessage 合并去重** 的流式策略。配合 `react-virtuoso` 达成高性能虚拟滚动。

## 技术栈

| 层级 | 技术 |
|------|------|
| UI 框架 | React 19 + TypeScript |
| 容器 | Electron |
| 组件库 | Arco Design |
| 原子 CSS | UnoCSS |
| Markdown 解析 | `react-markdown` |
| 代码高亮 | `react-syntax-highlighter` (highlight.js) |
| 虚拟滚动 | `react-virtuoso` v4 |
| Session 存储 | SQLite (better-sqlite3) |
| 通信 | IPC (主进程 ↔ 渲染进程) + ACP stdio |

## 流式渲染核心机制

### 消息接收通道

```
Agent stdio → ACP JSON-RPC → AcpConnection (主进程)
  → IPC channel → AcpAgent (渲染进程)
    → AcpMessageParser → TMessage
      → useAcpMessage hook → React state
        → composeMessage() → 合并/去重 → UI 渲染
```

### useAcpMessage Hook

```typescript
function useAcpMessage() {
  // 监听 IPC 消息事件
  useEffect(() => {
    const unsubscribe = window.electron.onAcpMessage((event) => {
      // 解析 ACP session/update
      const message = parseAcpMessage(event);

      // 按 messageId 合并到消息列表
      setMessages(prev => composeMessage(prev, message));
    });

    return unsubscribe;
  }, []);
}
```

### composeMessage — 合并策略

```typescript
function composeMessage(
  existing: IMessage[],
  incoming: IMessage
): IMessage[] {
  // 按 msg_id / toolCallId / sessionId 匹配
  // 策略：
  //   1. 新消息 → push 到末尾
  //   2. 已有消息 → 原地更新 (替换 content)
  //   3. 工具状态更新 → 更新 toolCall status
  //   4. 流式文本 → 追加 delta 到现有消息

  const index = existing.findIndex(m => m.id === incoming.id);
  if (index >= 0) {
    // 合并更新：替换该位置
    const updated = [...existing];
    updated[index] = mergeMessage(existing[index], incoming);
    return updated;
  } else {
    return [...existing, incoming];
  }
}
```

### composeMessageWithIndex — O(1) 索引

```typescript
// 维护 Map<string, number> 索引
// messageId → 数组位置
// 查找 O(1) 而非 O(n)
const messageIndex = useRef(new Map<string, number>());

function composeMessageWithIndex(prev: IMessage[], msg: IMessage): IMessage[] {
  const idx = messageIndex.current.get(msg.id);
  if (idx !== undefined && idx < prev.length) {
    const updated = [...prev];
    updated[idx] = mergeMessage(prev[idx], msg);
    return updated;
  }
  // 新消息 → 添加索引
  messageIndex.current.set(msg.id, prev.length);
  return [...prev, msg];
}
```

## 消息组件清单（14 种）

```
MessageText            — 纯文本/Markdown 消息
MessageThinking        — 推理思考块（可折叠）
MessageToolCall        — 单个工具调用卡片
MessageToolGroup       — 多个工具聚合分组
MessageAcpPermission   — ACP 权限请求弹窗
MessageAcpToolCall     — ACP 工具调用（含状态更新）
MessageCodexPermission — Codex 权限请求
MessageCodexToolCall   — Codex 工具调用
MessageSystem          — 系统通知
MessageError           — 错误消息
MessageUser            — 用户消息
MessagePlan            — 计划展示
MessageSessionSwitch   — 会话切换标记
MessageDivider         — 时间分隔线
```

### MessageText — 流式 Markdown 渲染

```tsx
const MessageText = ({ content, isStreaming }) => (
  <div className="message-text">
    <ReactMarkdown
      remarkPlugins={[remarkGfm, remarkMath]}
      rehypePlugins={[rehypeHighlight, rehypeKatex]}
    >
      {content}
    </ReactMarkdown>
    {isStreaming && <span className="cursor-blink">▊</span>}
  </div>
);
```

### MessageThinking — 可折叠推理块

```tsx
const MessageThinking = ({ content, status }) => {
  const [expanded, setExpanded] = useState(status === 'streaming');

  return (
    <Collapse activeKey={expanded ? 'thinking' : undefined}>
      <CollapseItem key="thinking" header="Thinking Process">
        <ReactMarkdown>{content}</ReactMarkdown>
      </CollapseItem>
    </Collapse>
  );
};
```

### MessageToolGroup — 工具组聚合

```tsx
// 将连续的工具调用聚合为一个分组卡片
// 例如: Read → Bash → Edit → Write 自动归组
const MessageToolGroup = ({ tools }) => (
  <Accordion>
    {tools.map(tool => (
      <ToolCallItem key={tool.id} tool={tool} />
    ))}
  </Accordion>
);
```

### MessageAcpPermission — 权限审批 UI

```tsx
const MessageAcpPermission = ({ permission }) => {
  const options = permission.options; // AcpPermissionOption[]

  return (
    <div className="permission-card bg-yellow-50 border border-yellow-200">
      <div className="permission-header">
        <IconWarning /> {permission.title}
      </div>
      <div className="permission-body">
        <pre>{JSON.stringify(permission.rawInput, null, 2)}</pre>
      </div>
      <div className="permission-actions">
        {options.map(opt => (
          <Button key={opt.optionId}
            type={opt.kind.startsWith('allow') ? 'primary' : 'default'}
            onClick={() => approvePermission(opt.optionId)}>
            {opt.name}
          </Button>
        ))}
      </div>
    </div>
  );
};
```

## 历史消息（Session）渲染

### SQLite 存储架构

```sql
-- 会话表
CREATE TABLE sessions (
  id TEXT PRIMARY KEY,
  title TEXT,
  agent_type TEXT,
  created_at INTEGER,
  updated_at INTEGER
);

-- 消息表
CREATE TABLE messages (
  id TEXT PRIMARY KEY,
  session_id TEXT REFERENCES sessions(id),
  type TEXT,           -- TMessageType
  content TEXT,        -- JSON serialized
  created_at INTEGER
);
```

### 历史加载流程

```
1. 用户选择 Session → loadSession(sessionId)
2. SQLite 查询:
   SELECT * FROM messages
   WHERE session_id = ?
   ORDER BY created_at ASC
3. ComposeHistoryMessages() — 将原始消息合并为 UI 消息
   (同一 toolCall 的多次 update → 一条 MessageAcpToolCall)
4. 渲染 → react-virtuoso 虚拟滚动列表
```

### react-virtuoso 集成

```tsx
import { Virtuoso } from 'react-virtuoso';

const ChatView = ({ messages }) => (
  <Virtuoso
    data={messages}
    itemContent={(index, message) => (
      <MessageRenderer message={message} />
    )}
    followOutput="smooth"        // 流式时平滑跟随
    atBottomStateChange={(atBottom) => {
      // 用户手动上滚时不自动跟随
    }}
    initialTopMostItemIndex={messages.length - 1}  // 从底部开始
  />
);
```

## 渲染优化

| 优化点 | 实现 |
|--------|------|
| O(1) 消息合并 | `composeMessageWithIndex` 用 Map 索引 |
| 虚拟滚动 | `react-virtuoso` 仅渲染可视区消息 |
| IPC 批处理 | 主进程批量发送消息事件 |
| React.memo | `MessageRenderer` + 各子组件 memo 化 |
| 60s 工具清理 | completed/failed 工具自动清理 |
| UnoCSS | 按需生成 CSS，零运行时 |
