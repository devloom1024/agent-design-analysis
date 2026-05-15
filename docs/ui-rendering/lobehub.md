# LobeHub — UI 渲染层

## 架构特点

Next.js Web 应用，采用 **API 协议工厂 + @lobehub/ui 组件体系**。流式渲染利用 `@lobehub/ui` Markdown 的 `animated` 属性内置打字机效果。

## 技术栈

| 层级 | 技术 |
|------|------|
| UI 框架 | Next.js 16 + React 19 + TypeScript |
| 组件库 | `@lobehub/ui` v5.12.0 + antd |
| Markdown | `@lobehub/ui` Markdown（内置 `animated` 流式动画） |
| 代码高亮 | Shiki（`@lobehub/ui` 内置） |
| 虚拟滚动 | `virtua` v0.48 |
| 状态管理 | zustand |
| Session 存储 | PostgreSQL + Drizzle ORM |
| 流式通信 | fetch SSE ReadableStream |

## 流式渲染核心机制

### 数据流路径

```
AI Provider → runner() → AgentInstruction
  → ChatStreamPayload (SSE chunk)
    → UIChatMessage (normalized)
      → UIMessageRenderer (type dispatch)
        → <Markdown animated={isStreaming} />
```

### ChatStreamPayload — SSE 消息格式

```typescript
interface ChatStreamPayload {
  id: string;
  role: 'assistant' | 'user' | 'system' | 'tool';
  content: string;
  delta?: string;
  finishReason?: 'stop' | 'tool_calls' | 'length';
  toolCalls?: ChatToolPayload[];
  parentId?: string;
  sessionId?: string;
  error?: any;
}
```

### SSE ReadableStream 消费

```typescript
// 使用 fetch 读取 SSE 流
async function* streamChat(payload: ChatRequestPayload) {
  const response = await fetch('/api/chat', {
    method: 'POST',
    body: JSON.stringify(payload),
  });

  const reader = response.body!.getReader();
  const decoder = new TextDecoder();

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    const text = decoder.decode(value, { stream: true });
    for (const chunk of parseSSEChunks(text)) {
      yield JSON.parse(chunk) as ChatStreamPayload;
    }
  }
}
```

### UIChatMessage — 标准化消息

```typescript
interface UIChatMessage {
  id: string;
  role: UIMessageRoleType;
  content: string;
  streaming?: boolean;
  error?: any;
  toolPayloads?: ChatToolPayload[];
  parentId?: string;
  childrenIds?: string[];
  // LobeHub 特有
  extra?: {
    fromModel?: string;
    fromProvider?: string;
    translate?: { content: string; target: string };
    reasoning?: { content: string; duration: number };
  };
}
```

## UIMessageRoleType 组件清单（12 种）

```
Default             — 通用消息（回退组件）
AIChatMessage       — AI 回复（Markdown 渲染）
UserMessage         — 用户消息气泡
SystemMessage       — 系统通知
ToolMessage         — 工具调用结果
ThinkingGroup       — 思考过程组
AssistantGroup      — 子 Agent 消息组（分区算法）
Intervention        — 权限审批介入
WorkflowCollapse    — 工作流折叠展示
CompressedGroup     — 上下文压缩组（含流式摘要）
FunctionCall        — 函数调用展示
LoadingMessage      — 加载占位符
```

### AIChatMessage — Markdown + 流式动画

```tsx
import { Markdown } from '@lobehub/ui';

const AIChatMessage = ({ content, streaming, extra }) => (
  <div className="chat-message ai-message">
    {/* 推理内容 */}
    {extra?.reasoning && (
      <ThinkingGroup
        content={extra.reasoning.content}
        duration={extra.reasoning.duration}
      />
    )}

    {/* 正文 — 关键: animated prop 启用打字机效果 */}
    <Markdown
      animated={streaming}
      variant="chat"
    >
      {content}
    </Markdown>

    {/* 翻译内容 */}
    {extra?.translate && (
      <TranslationBlock
        content={extra.translate.content}
        target={extra.translate.target}
      />
    )}
  </div>
);
```

### @lobehub/ui Markdown 的 animated 属性

```typescript
// @lobehub/ui 内部实现（简化）
// animated=true 时：
// 1. 逐字符/逐词显示内容
// 2. 使用 CSS transition 实现平滑出现
// 3. 代码块一次性渲染（完成后才高亮）
// 4. 数学公式延迟渲染（等待闭合标记）

// 自定义 remark/rehype 插件管道：
<Markdown
  remarkPlugins={[
    remarkGfm,        // GitHub Flavored Markdown
    remarkMath,       // 数学公式
    remarkMermaid,    // Mermaid 图表
    // ... 8+ 自定义插件
  ]}
  rehypePlugins={[
    rehypeShiki,      // Shiki 代码高亮
    rehypeKatex,      // KaTeX 数学渲染
  ]}
/>
```

### ThinkingGroup — 可折叠推理展示

```tsx
const ThinkingGroup = ({ content, duration, streaming }) => (
  <Collapse defaultActive={streaming}>
    <Collapse.Panel
      header={
        <Flex>
          <IconBrain />
          <span>Thinking {streaming && '...'}</span>
          {duration && <span className="duration">{duration}s</span>}
        </Flex>
      }
    >
      <Markdown>{content}</Markdown>
    </Collapse.Panel>
  </Collapse>
);
```

### AssistantGroup — 子 Agent 分区

```tsx
// Partition Algorithm: 按 parentId 将消息分组
// 同一子 Agent 的消息聚合到一个 AssistantGroup
const AssistantGroup = ({ messages, agentName }) => {
  const [collapsed, setCollapsed] = useState(false);

  return (
    <div className="assistant-group">
      <div className="group-header" onClick={() => setCollapsed(!collapsed)}>
        <IconAgent />
        <span>{agentName}</span>
        <span className="msg-count">{messages.length} messages</span>
      </div>
      {!collapsed && messages.map(msg => (
        <UIMessageRenderer key={msg.id} message={msg} />
      ))}
    </div>
  );
};
```

### CompressedGroup — 上下文压缩

```tsx
// 上下文压缩后的消息折叠显示
const CompressedGroup = ({ summary, originalCount }) => (
  <div className="compressed-group">
    <div className="compressed-badge">
      <IconCompress />
      <span>Context Compressed ({originalCount} messages)</span>
    </div>
    <Markdown streaming={summary.isStreaming}>
      {summary.content}
    </Markdown>
  </div>
);
```

### Intervention — 权限审批介入

```tsx
// 权限审批系统 — LobeHub 的人工介入机制
const Intervention = ({ intervention }) => {
  switch (intervention.status) {
    case 'pending':
      return <InterventionPending tool={intervention.tool} />;
    case 'approved':
      return <InterventionApproved tool={intervention.tool} />;
    case 'rejected':
      return <InterventionRejected tool={intervention.tool}
               reason={intervention.rejectedReason} />;
    case 'aborted':
      return <InterventionAborted />;
  }
};
```

## 工具调用渲染

### ChatToolPayload 在 UI 中的渲染

```tsx
// 按 type 分发不同的渲染组件
const ToolRenderer = ({ tool }: { tool: ChatToolPayload }) => {
  switch (tool.type) {
    case 'default':
    case 'standalone':
      return <DefaultToolCallCard tool={tool} />;
    case 'markdown':
      return <MarkdownToolResult tool={tool} />;
    case 'mcp':
      return <MCPToolCallCard tool={tool} />;
  }
};

const DefaultToolCallCard = ({ tool }) => (
  <Card size="small" title={`🔧 ${tool.apiName}`}>
    <Collapse>
      <Collapse.Panel key="input" header="Input">
        <CodeBlock language="json">
          {tool.arguments}
        </CodeBlock>
      </Collapse.Panel>
      <Collapse.Panel key="output" header="Output">
        {/* 工具执行结果 */}
      </Collapse.Panel>
    </Collapse>
  </Card>
);
```

### 三个子系统（Inspectors / Interventions / Renders）

```
Inspectors    → React 组件，展示每个 apiName 的工具调用详情
Interventions → 人工介入流程 (AskUserQuestion 等)
Renders       → 工具执行结果的可视化渲染（含 Streaming 变体）
```

## 历史消息（Session）渲染

### 数据库架构

```
PostgreSQL + Drizzle ORM

sessions 表:
  id, title, agentType, modelName, provider,
  createdAt, updatedAt, userId

messages 表:
  id, sessionId, role, content (JSON),
  parentId, childrenIds (JSON array),
  extra (JSON), createdAt

topic 表:
  id, sessionId, title, createdAt
```

### 历史消息加载

```typescript
// 使用 Drizzle ORM 查询
const loadSessionMessages = async (sessionId: string) => {
  const messages = await db.query.messages.findMany({
    where: eq(messages.schema.sessionId, sessionId),
    orderBy: asc(messages.schema.createdAt),
  });

  return messages.map(deserializeMessage); // JSON → UIChatMessage
};
```

### virtua 虚拟滚动集成

```tsx
import { VList } from 'virtua';

const ChatList = ({ messages, streaming }) => {
  const ref = useRef<VListHandle>(null);

  // 流式时自动滚动到底部
  useEffect(() => {
    if (streaming) {
      ref.current?.scrollToIndex(messages.length - 1);
    }
  }, [messages.length, streaming]);

  return (
    <VList ref={ref} keepMounted={streaming ? [messages.length - 1] : []}>
      {messages.map(msg => (
        <UIMessageRenderer key={msg.id} message={msg} />
      ))}
    </VList>
  );
};
```

`keepMounted` 是关键优化 — 流式消息在视口外也保持挂载，确保 delta 追加不丢失。

## 状态管理（zustand）

```typescript
interface ChatStore {
  messages: UIChatMessage[];       // 当前会话消息
  streamingMessageId: string | null;
  sessions: SessionSummary[];      // 会话列表

  // Actions
  appendMessage: (msg: UIChatMessage) => void;
  updateMessage: (id: string, update: Partial<UIChatMessage>) => void;
  deleteMessage: (id: string) => void;
  setStreaming: (id: string | null) => void;

  loadSessions: () => Promise<void>;
  switchSession: (id: string) => Promise<void>;
}
```

## 渲染优化

| 优化点 | 实现 |
|--------|------|
| 打字机动画 | `@lobehub/ui` Markdown `animated` prop（CSS transition） |
| 虚拟滚动 | `virtua` + `keepMounted` 防止流式消息丢失 |
| 索引管理 | zustand 配合 Map 索引实现 O(1) 消息查找 |
| 代码高亮 | Shiki — 服务端/构建时预处理，零运行时开销 |
| 8+ 插件管道 | remark/rehype 插件按需加载、服务端预处理 |
| SSR | Next.js App Router SSR 首屏渲染 |
