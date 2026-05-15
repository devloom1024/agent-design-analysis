# AgentAPI — UI 渲染层

## 架构特点

**最简实现** — 仅提供 Web Chat UI 基础骨架。纯文本 SSE 流式显示，无 markdown 解析，无代码高亮，无 Session 存储。

## 技术栈

| 层级 | 技术 |
|------|------|
| UI 框架 | Next.js 15 + React 19 + TypeScript |
| 样式方案 | Tailwind CSS 4 |
| 基础组件 | Radix UI |
| 流式通信 | 原生 EventSource (SSE) |
| Markdown | ❌ 不支持（纯文本） |
| 代码高亮 | ❌ 不支持 |
| 虚拟滚动 | ❌ 不支持 |
| Session 存储 | ❌ 不支持 |

## 流式渲染机制

### SSE 流式接收

```typescript
// ChatInterface.tsx
function ChatInterface({ agentId, sessionId }) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [streamingContent, setStreamingContent] = useState('');

  const sendMessage = async (userInput: string) => {
    // 通过 fetch 发送用户消息
    const response = await fetch('/api/chat', {
      method: 'POST',
      body: JSON.stringify({ message: userInput, agentId }),
    });

    // 读取 SSE 流
    const reader = response.body?.getReader();
    const decoder = new TextDecoder();

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      const text = decoder.decode(value, { stream: true });
      // 解析 SSE 格式: "data: <content>\n\n"
      const lines = text.split('\n');
      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const content = line.slice(6);
          setStreamingContent(prev => prev + content);
        }
      }
    }

    // 流结束 → 添加到消息列表
    setMessages(prev => [...prev, {
      role: 'assistant',
      content: streamingContent,
    }]);
    setStreamingContent('');
  };
}
```

### 渲染组件

```tsx
// 消息气泡 — 极简实现
const MessageBubble = ({ message }) => (
  <div className={`flex ${message.role === 'user' ? 'justify-end' : 'justify-start'}`}>
    <div className={`max-w-[80%] p-3 rounded-lg ${
      message.role === 'user'
        ? 'bg-blue-500 text-white'
        : 'bg-gray-100 text-gray-900'
    }`}>
      <pre className="whitespace-pre-wrap font-sans text-sm">
        {message.content}
      </pre>
    </div>
  </div>
);

// 流式指示器
const StreamingIndicator = ({ content }) => (
  <div className="flex justify-start">
    <div className="bg-gray-100 p-3 rounded-lg max-w-[80%]">
      <pre className="whitespace-pre-wrap font-sans text-sm">
        {content}
        <span className="animate-pulse">▊</span> {/* 闪烁光标 */}
      </pre>
    </div>
  </div>
);
```

## 组件清单

| 组件 | 文件 | 说明 |
|------|------|------|
| `ChatInterface` | `chat-interface.tsx` | 主聊天容器 |
| `ChatInput` | `chat-input.tsx` | 输入框 + 发送按钮 |
| `MessageBubble` | `chat-interface.tsx` | 消息气泡（内联） |
| `StreamingIndicator` | `chat-interface.tsx` | 流式内容 + 闪烁光标 |
| `AgentSelector` | `sidebar.tsx` | Agent 类型选择器 |
| `SessionList` | `sidebar.tsx` | 会话列表（仅 sessionId 显示） |

## 不支持的功能

### 无 Markdown/代码高亮

```
所有内容使用 <pre className="whitespace-pre-wrap">
纯文本显示，保留空白和换行
代码块无语法高亮，作为普通文本渲染
```

### 无工具调用 UI

```
PTY 模式 → CLI 自身 TUI 处理工具显示
ACP 模式 → 仅格式化文本: "[Tool: Read] filename"
不做任何特殊 UI 渲染
```

### 无 Session 持久化

```
会话仅在浏览器内存中
刷新页面 → 丢失所有消息
无历史记录功能
```

## 消息格式化中的特殊处理

### PTY 模式消息清洗

```go
// FormatMessage() — 过滤终端控制序列
func FormatMessage(agentType AgentType, raw string) string {
    // 移除:
    // - 用户输入回显 (TTY echo)
    // - TUI 选择框 (fzf/fuzzyfinder)
    // - Bracketed Paste 序列 (\x1b[200~ ... \x1b[201~)
    // - ANSI escape codes
}
```

### Bracketed Paste 包裹

```go
// 避免特殊字符被 TTY 解释为快捷键
msg := fmt.Sprintf("\x1b[200~%s\x1b[201~", userInput)
```

## 设计哲学

AgentAPI 的前端是**概念验证级别的参考实现**：
- 仅展示 Agent 通信的基本流程
- 不追求生产级 UI 体验
- 所有增强功能（Markdown、代码高亮、权限 UI）留给下游消费者实现
- ACP 协议承载结构化数据 → 下游可构建丰富的自定义 UI
