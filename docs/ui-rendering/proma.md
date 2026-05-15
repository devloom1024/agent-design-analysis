# Proma — UI 渲染层

## 架构特点

Electron 桌面应用，采用 **IPC 事件推送 + Jotai 原子状态驱动** 的流式策略。Chat 和 Agent 模式使用独立的全局监听器，但共享底层的 Markdown/代码渲染组件。

## 技术栈

| 层级 | 技术 |
|------|------|
| UI 框架 | React 18 + TypeScript |
| 容器 | Electron 39 |
| 状态管理 | Jotai (27 个 atom 文件) |
| 样式方案 | Tailwind CSS 3 + shadcn/ui (Radix UI) |
| 富文本编辑器 | TipTap 3.19 |
| Markdown 解析 | `react-markdown` |
| 图表/公式 | Beautiful Mermaid + KaTeX |
| 代码高亮 | Shiki 3.22 |
| 构建 | Vite 6 + esbuild |
| 存储 | JSON/JSONL 文件（`~/.proma/`） |

## 流式渲染核心机制

### Agent 模式流式通道

```
Claude Agent SDK binary → child_process stdout
  → MessageChannel (AsyncGenerator)
    → agent-orchestrator.ts 迭代 SDKMessage stream
      → convertSDKMessage() → AgentEvent[]
        → AgentEventBus middleware chain
          → webContents.send(AGENT_IPC_CHANNELS.STREAM_EVENT)
            → useGlobalAgentListeners (46KB hook)
              → useStore().set(agentAtoms) — 直接写入 Jotai
                → React 组件 re-render
```

### Chat 模式流式通道

```
fetch SSE → ReadableStream
  → sse-reader.ts → adapter.parseSSELine()
    → webContents.send(CHAT_IPC_CHANNELS.STREAM_EVENT)
      → useGlobalChatListeners (7KB hook)
        → useStore().set(chatAtoms)
          → React 组件 re-render
```

### 全局监听器模式（Proma 独有）

```typescript
// main.tsx — App 根级别挂载全局监听器
function App() {
  return (
    <>
      <ThemeInitializer />
      <AgentSettingsInitializer />
      <AgentListenersInitializer />   {/* useGlobalAgentListeners */}
      <ChatListenersInitializer />    {/* useGlobalChatListeners */}
      <UpdaterInitializer />
      <AppShell />                    {/* 实际 UI */}
    </>
  );
}

// useGlobalAgentListeners.ts (46KB)
// 永不卸载的全局 Hook，使用 useStore() 直接操作 Jotai atoms
function useGlobalAgentListeners() {
  const store = useStore();

  useEffect(() => {
    const unsubscribe = window.electronAPI.onAgentStreamEvent((event) => {
      // 绕过 React 渲染周期，直接写入 atom
      store.set(agentContentAtom, prev => prev + event.delta);
      store.set(agentRunningAtom, true);
      // ...
    });
    return unsubscribe;
  }, []);

  // 使用 unstable_batchedUpdates 批量更新
  // 多个 atom 更新在一次渲染中完成
}
```

**`unstable_batchedUpdates` 优化**：多个 atom 同时更新时，React 仅触发一次 re-render。

### 性能架构

```
常规 React: atom 更新 → re-render → atom 更新 → re-render (串联)
Proma Jotai: useStore() 直接写入 → unstable_batchedUpdates → 一次 re-render
```

## UI 组件清单

### 消息渲染组件

| 组件 | 说明 |
|------|------|
| `ChatMessageItem` | Chat 消息气泡（用户/AI 双栏布局） |
| `AgentMessageItem` | Agent 消息容器 |
| `MarkdownRenderer` | react-markdown + Shiki + Mermaid + KaTeX |
| `ThinkingBlock` | 可折叠思考展示（含 signature 信息） |
| `ThinkingTrigger` | 思考中占位指示器 |
| `ToolActivityItem` | 工具调用卡片（含进度+耗时+结果） |
| `ToolGroupItem` | 工具组聚合展示 |
| `SubAgentCard` | 子 Agent 调用卡片（code-reviewer/explorer/researcher） |
| `PermissionRequestCard` | 权限请求卡片（Allow/Deny/Always Allow） |
| `AskUserQuestionForm` | 用户问题表单（选项/输入） |
| `ExitPlanApproval` | 计划退出审批 |
| `CompactingIndicator` | 上下文压缩指示器 |
| `ErrorMessage` | 错误卡片（含重试按钮） |
| `RetryBanner` | 自动重试状态横幅 |
| `TokenUsageBar` | Token 用量 + 费用统计 |
| `ContextWindowBar` | 上下文窗口占用率 |

### ToolActivityItem — 工具调用组件

```tsx
const ToolActivityItem = ({ activity }: { activity: ToolActivity }) => {
  // activity 结构
  interface ToolActivity {
    toolUseId: string;
    toolName: string;
    input: Record<string, unknown>;
    status: 'pending' | 'running' | 'completed' | 'error';
    startTime: number;
    duration?: number;
    result?: string;
    error?: string;
  }

  return (
    <Collapsible defaultOpen={activity.status === 'running'}>
      <ToolHeader>
        <Icon name={activity.toolName} />
        <span>{activity.toolName}</span>
        <StatusBadge status={activity.status} />
        {activity.duration && <DurationBadge ms={activity.duration} />}
      </ToolHeader>
      <ToolInput>
        <CodeBlock language="json">{JSON.stringify(activity.input)}</CodeBlock>
      </ToolInput>
      {activity.result && <ToolResult>{activity.result}</ToolResult>}
      {activity.error && <ToolError>{activity.error}</ToolError>}
    </Collapsible>
  );
};
```

### ai-elements/ — 共享渲染组件

```typescript
// apps/electron/src/renderer/components/ai-elements/
CodeBlock         // Shiki 语法高亮 (packages/ui)
MermaidBlock      // Beautiful Mermaid 图表渲染
FileDiffView      // Diff 对比视图
FilePreview       // 文件内容预览
ProgressBar       // 不确定进度条 (Streaming 时)
ReasoningRenderer // 思考过程（可折叠，含 signature 信息）
```

### TipTap 编辑栏

```
用户输入区域使用 TipTap 富文本编辑器:
- 支持 Markdown 快捷键
- 支持 @ 提及文件
- 支持拖拽图片附件
- 支持斜杠命令 (/)
- AI 辅助输入内联建议
```

## 权限 UI 组件

### PermissionRequestCard

```tsx
const PermissionRequestCard = ({ request }) => (
  <Card className="border-yellow-300 bg-yellow-50">
    <CardHeader>
      <IconShield /> Permission Required
    </CardHeader>
    <CardBody>
      <div>Tool: <strong>{request.toolName}</strong></div>
      {request.toolName === 'Bash' && (
        <div>Command: <code>{request.input.command}</code></div>
      )}
      {/* 危险分析结果展示 */}
      {request.riskAssessment && (
        <RiskWarning assessment={request.riskAssessment} />
      )}
    </CardBody>
    <CardFooter>
      <Button variant="outline" onClick={() => denyPermission(request)}>Deny</Button>
      <Button onClick={() => approvePermission(request, 'once')}>Allow</Button>
      <Button variant="primary" onClick={() => approvePermission(request, 'always')}>
        Always Allow
      </Button>
    </CardFooter>
  </Card>
);
```

## 历史消息（Session）渲染

### 存储架构

```
~/.proma/
├── conversations.json            # Chat 会话索引
├── agent-sessions.json           # Agent 会话索引
├── conversations/{id}.jsonl      # Chat 消息历史 (追加写)
├── agent-sessions/{id}.jsonl     # Agent 消息历史 (追加写)
└── agent-workspaces/{slug}/      # Agent 工作空间
    └── {session-id}/             # 每个 session 独立工作目录
```

### 历史加载流程

```
1. 读取 agent-sessions.json → 获取 session 列表
2. 用户选择 session → 读取 {sessionId}.jsonl
3. 全量解析 JSONL → 构建消息列表
4. Jotai atoms 更新 → UI 渲染

Chat: conversation-manager.ts (14KB)
Agent: agent-session-manager.ts (51KB)
```

### 消息恢复渲染

```
历史消息全量加载（无分页）
JSONL 文件 → 逐行 parse → 消息对象数组 → React 渲染

活跃 session 流式模式:
  [历史消息] + [当前流式消息]
  滚动位置: 自动跟随流式输出到底部
  用户手动上滚 → 停止自动跟随

非活跃 session 查看模式:
  [历史消息]
  滚动位置: 从顶部开始
```

### 上下文压缩消息

```tsx
// 压缩后的消息以折叠卡片展示
const CompressedMessageBlock = ({ summary, originalCount }) => (
  <Card className="bg-gray-100 border-dashed">
    <IconCompress />
    <span>Context was compressed ({originalCount} messages condensed)</span>
    <MarkdownRenderer content={summary} />
  </Card>
);
```

## Tab 管理

```typescript
// tab-atoms.ts
// Agent session 可以打开为独立 Tab
interface Tab {
  id: string;
  type: 'agent' | 'chat' | 'settings';
  sessionId: string;
  title: string;
}

// 支持拖拽排序
// 每个 Tab 独立维护滚动位置和消息状态
// 切换 Tab 不影响其他 Tab 的流式接收
```

## 渲染优化

| 优化点 | 实现 |
|--------|------|
| Jotai 直接写入 | `useStore()` 绕过 React 渲染周期 |
| 批量更新 | `unstable_batchedUpdates` 合并多个 atom 变更 |
| 全局单例 Hook | `useGlobalAgentListeners` 永不卸载 |
| Shiki | 构建时预处理，零运行时开销 |
| 零依赖 UI | Radix UI 按需导入 |
| JSONL 追加写 | 无索引开销，写入快 |
