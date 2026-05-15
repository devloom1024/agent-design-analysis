# CC GUI — UI 渲染层

## 架构特点

JCEF (Java Chromium Embedded Framework) 嵌入浏览器 + React 19 前端。流式渲染采用**轻量自定义逐帧更新**策略。

## 技术栈

| 层级 | 技术 |
|------|------|
| UI 框架 | React 19 + TypeScript |
| 组件库 | Ant Design 6 |
| 构建工具 | Vite 7 |
| Markdown 解析 | `marked` |
| 代码高亮 | `highlight.js` |
| 虚拟滚动 | 自定义 `VirtualList` |
| 宿主容器 | JCEF (Java Chromium Embedded Framework) |
| 通信桥接 | JCEF → Java → WebSocket → Node.js → Claude SDK |

## 流式渲染核心机制

### 基本流式流程

```
Node.js Claude SDK → stream 回调 → WebSocket → Java → JCEF → React state
  → renderStreamingContent() → 逐帧增量更新 DOM
```

### MarkdownBlock.tsx — 自定义轻量流式渲染器

```typescript
// 核心渲染方法
function renderStreamingContent(
  content: string,       // 当前累积的完整文本
  prevContent: string,   // 上次渲染的文本
): ReactNode {
  // 策略：不重新解析整个 markdown，而是按帧增量更新
  // 仅渲染新增部分，减少重绘

  // 1. 代码块处理：处于代码块内部时，使用 <pre><code> 直接渲染
  // 2. 普通文本：marked.parse() 解析
  // 3. 增量对比：只更新变化的部分
}
```

**增量更新策略**：
- 检测是否处于代码块内部（未闭合的 ``` 标记）
- 代码块内：直接 `<pre><code>` 追加，不经过 markdown 解析
- 普通段落：调用 `marked.parse()` 重新解析全文
- 利用 React key 机制最小化 DOM 操作

### 代码块实时渲染

```typescript
// StreamingCodeBlock 组件
// 代码块未完成时：纯文本追加显示，无语法高亮
// 代码块完成后：触发 highlight.js 高亮
// 完成判定：检测到闭合的 ``` 标记
```

## 自定义 UI 组件清单

### toolBlocks/ 目录

自定义工具块组件，每种工具类型一个独立组件：

| 组件 | 工具类型 | 渲染内容 |
|------|---------|---------|
| `ReadBlock` | Read | 文件内容预览，行号 + 语法高亮 |
| `WriteBlock` | Write | diff 视图，新增/删除行对比 |
| `EditBlock` | Edit | 变更前后对比，行内 diff |
| `BashBlock` | Bash | 终端输出，ANSI 颜色还原 |
| `GrepBlock` | Grep | 搜索结果列表，文件名 + 匹配行 |
| `GlobBlock` | Glob | 文件匹配列表 |
| `TodoBlock` | TodoWrite | 任务列表，checkbox 状态 |
| `AgentBlock` | Agent | 子 Agent 状态卡片 |
| `AskUserQuestionBlock` | AskUserQuestion | 问题 + 选项表单 |
| `ExitPlanModeBlock` | ExitPlanMode | 计划展示 + 批准按钮 |

### 权限 UI 组件

- **PermissionDialog** — JCEF 原生窗口弹窗，三个按钮（Allow / Allow Always / Deny）
- **DiffReviewService** — Edit/Write 工具自动展示文件 diff
- **AskUserQuestionDialog** — 多选/单选问题弹窗
- **PlanApprovalDialog** — 计划审批弹窗

## 历史消息（Session）渲染

### 存储架构

```
[前端 React] ← JCEF → [Java 后端]
                         ├── SessionManager — 管理会话元数据
                         ├── MessageStore — 消息持久化（JSON 文件）
                         └── PermissionDecisionStore — 权限记忆
```

### 消息列表组件

```typescript
// MessageList 组件
// 使用自定义 VirtualList 实现虚拟滚动
// 特性：
// - 仅渲染可视区域内的消息
// - 消息从 Java 后端同步加载
// - 支持按 sessionId 切换会话
// - 新消息自动滚动到底部（streaming 时）
```

### 自定义 VirtualList 设计

```typescript
class VirtualList {
  // 核心参数
  itemHeight: number;          // 固定项高度估算
  overscan: number;            // 预渲染行数（默认 5）
  visibleRange: [number, number]; // 当前可视范围

  // 滚动回调 → 重新计算 visibleRange → 仅渲染范围内容器
  // 使用 CSS transform: translateY() 定位
}
```

### 会话切换

```
1. 用户选择新 sessionId
2. Java MessageStore.loadMessages(sessionId)
3. 返回消息列表 → VirtualList reset
4. 滚动到顶部（历史模式）或底部（活跃会话）
```

## 与 Claude Code CLI 集成

```
Claude Code CLI ← Node.js Bridge ← WebSocket → Java Backend ← JCEF → React UI
                    │
                    ├── 流式输出: stdout 逐行 → WebSocket → React state
                    └── 权限请求: 拦截 canUseTool → 文件系统 JSON → Java 轮询
```

## 渲染优化

- **React.memo** — 消息列表项避免不必要的重渲染
- **useMemo** — marked.parse() 结果缓存（非流式部分）
- **requestAnimationFrame** — 流式更新节流，避免过度渲染
- **虚拟滚动** — 大量历史消息仅渲染可视区
- **增量更新** — 流式内容仅渲染新增部分，不重新解析全文
