# Proma 架构分析

## 项目概述

Proma 是一个本地优先的 AI 桌面工作台（Electron + React 18），同时支持 **Chat 模式**（多 Provider SSE）和 **Agent 模式**（Claude Agent SDK）。采用 **第五种架构范式：双模式混合架构**。

- 仓库: https://github.com/ErlichLiu/Proma
- 技术栈: Electron + React 18 + TypeScript + Jotai + Tailwind CSS + Bun monorepo

## 统一抽象层 — 双模式混合架构

### 模式一：Chat 模式 — Provider Adapter 工厂

`packages/core/src/providers/` — 三个 Provider 适配器：

```typescript
// AnthropicAdapter  — Messages API（也用于 DeepSeek, Kimi, MiniMax）
// OpenAIAdapter    — Chat Completions API（也用于 Zhide, Doubao, Tongyi）
// GoogleAdapter    — Gemini Generative Language API
```

通过 SSE（`sse-reader.ts`）统一流式消费：`fetch → ReadableStream → data: 解析 → adapter.parseSSELine()`

### 模式二：Agent 模式 — Claude Agent SDK

`apps/electron/src/main/lib/adapters/claude-agent-adapter.ts`（37KB）— 实现 `AgentProviderAdapter` 接口：

```
@anthropic-ai/claude-agent-sdk query()
  → MessageChannel (AsyncGenerator)
    → child_process stdin/stdout
      → SDKMessage stream
```

**支持的 Agent 渠道**：Anthropic, DeepSeek, Kimi API, Kimi Coding Plan, Zhide AI, MiniMax, Doubao, Tongyi Qianwen（均通过 Anthropic 兼容协议）。

OpenAI、Google、Custom 渠道仅支持 Chat 模式。

### 整体通信架构

```
┌──────────────────────────────────────────────────────────────┐
│ Renderer (React 18)                                          │
│   Jotai atoms ← useGlobalAgentListeners / useGlobalChatListeners │
├──────────────────────────────────────────────────────────────┤
│ Preload (contextBridge)                                       │
├──────────────────────────────────────────────────────────────┤
│ Main Process                                                  │
│   ├── agent-orchestrator.ts (101KB) — Agent 核心编排          │
│   ├── agent-service.ts (12KB) — IPC 薄层                      │
│   ├── chat-service.ts (21KB) — Chat SSE 管理                  │
│   ├── claude-agent-adapter.ts (37KB) — SDK 适配器             │
│   └── providers/ (core 包) — Chat Provider 适配器             │
├──────────────────────────────────────────────────────────────┤
│ Process                                                       │
│   ├── Chat: fetch SSE → Provider API (Anthropic/OpenAI/Google)│
│   └── Agent: child_process → Claude Agent SDK binary          │
└──────────────────────────────────────────────────────────────┘
```

## Provider 切换机制

用户在每个会话/对话创建时选择 Channel（provider + model）。Chat 和 Agent 模式独立选择。

```typescript
// Channel 配置
interface ChannelConfig {
  id: string;
  name: string;
  provider: 'anthropic' | 'openai' | 'google' | 'deepseek' | ...;
  model: string;
  apiKey: string;  // 通过 Electron safeStorage 加密存储
  mode: 'chat' | 'agent';
}
```

Agent 模式下还支持：
- **Workspace** — 独立的工作目录、MCP Servers、Skills
- **SubAgent** — 通过 SDK Agent tool 调用子 Agent（code-reviewer, explorer, researcher）

## 关键设计模式

| 模式 | 位置 | 说明 |
|------|------|------|
| 适配器模式 | `claude-agent-adapter.ts` + Provider adapters | SDK/API → 统一事件流 |
| 工厂模式 | Provider adapter 注册 | 按 provider 类型创建适配器 |
| 事件总线 | `agent-event-bus.ts` | 中间件链式处理 Agent 事件 |
| 原子化状态 | Jotai atoms (27 文件) | 细粒度响应式状态管理 |
| 策略模式 | `permission-rules.ts` | 安全/危险命令分类策略 |
| Promise/Map 暂停 | `agent-permission-service.ts` | 权限等待用户决策 |
| 消息通道 | `claude-agent-adapter.ts` | AsyncGenerator 保持 stdin 开放 |

## 核心文件

| 文件 | 职责 |
|------|------|
| `apps/electron/src/main/ipc.ts` | 所有 IPC 处理器注册 (113KB) |
| `apps/electron/src/main/lib/agent-orchestrator.ts` | Agent 核心编排 (101KB) |
| `apps/electron/src/main/lib/agent-session-manager.ts` | Session CRUD + JSONL (51KB) |
| `apps/electron/src/main/lib/adapters/claude-agent-adapter.ts` | SDK 适配器 (37KB) |
| `apps/electron/src/main/lib/agent-prompt-builder.ts` | 系统提示构建 (27KB) |
| `apps/electron/src/main/lib/agent-permission-service.ts` | 工具权限 (13KB) |
| `apps/electron/src/main/lib/agent-workspace-manager.ts` | 工作空间/MCP/Skills (28KB) |
| `apps/electron/src/main/lib/chat-service.ts` | Chat SSE 流管理 (21KB) |
| `apps/electron/src/main/lib/conversation-manager.ts` | Chat 对话 CRUD (14KB) |
| `packages/core/src/providers/sse-reader.ts` | 通用 SSE 读取器 |
| `packages/core/src/providers/index.ts` | Provider 适配器注册 |
| `packages/shared/src/constants/permission-rules.ts` | 权限规则定义 |
| `packages/shared/src/types/agent.ts` | Agent IPC 通道 + 类型 |
| `apps/electron/src/renderer/atoms/agent-atoms.ts` | Agent 状态原子 (31KB) |
| `apps/electron/src/renderer/hooks/useGlobalAgentListeners.ts` | 全局 Agent 监听器 (46KB) |
