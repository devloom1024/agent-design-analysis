# Claude Code UI (CloudCLI) 架构分析

## 项目概述

为 Claude Code、Cursor CLI、Codex、Gemini CLI 提供统一 Web UI 的开源项目。前后端分离架构，通过 **五面接口（five-facet interface）** 的 Provider 抽象统一多种 AI 编码工具。

- 仓库: https://github.com/siteboon/claudecodeui
- 技术栈: React + TypeScript + Vite + Tailwind CSS（前端），Express + WebSocket + SQLite（后端）

## 统一抽象层

### 核心类型: `LLMProvider` → `NormalizedMessage`

```typescript
// server/shared/types.ts
export type LLMProvider = 'claude' | 'codex' | 'gemini' | 'cursor';

export type NormalizedMessage = {
  id: string; sessionId: string; timestamp: string;
  provider: LLMProvider;
  kind: MessageKind;  // 'text' | 'tool_use' | 'tool_result' | 'thinking' | 'error' | ...
  role?: 'user' | 'assistant';
  content?: string;
};
```

所有 provider 的原始消息最终归一化为 `NormalizedMessage`。

### 五面接口: `IProvider`

```typescript
// server/shared/interfaces.ts
export interface IProvider {
  readonly id: LLMProvider;
  readonly mcp: IProviderMcp;                    // MCP 服务器管理
  readonly auth: IProviderAuth;                  // 认证状态
  readonly skills: IProviderSkills;              // Skills 发现
  readonly sessions: IProviderSessions;          // 消息归一化 + 历史
  readonly sessionSynchronizer: IProviderSessionSynchronizer; // 磁盘会话同步
}
```

每个 provider 的 5 个维度都有独立接口，子实现可自由替换。

### Provider 注册中心（工厂模式）

```typescript
// server/modules/providers/provider.registry.ts
const providers: Record<LLMProvider, IProvider> = {
  claude: new ClaudeProvider(),
  codex: new CodexProvider(),
  cursor: new CursorProvider(),
  gemini: new GeminiProvider(),
};
```

### 抽象基类

```typescript
// server/modules/providers/shared/base/abstract.provider.ts
export abstract class AbstractProvider implements IProvider {
  abstract readonly mcp: IProviderMcp;
  abstract readonly auth: IProviderAuth;
  abstract readonly skills: IProviderSkills;
  abstract readonly sessions: IProviderSessions;
  abstract readonly sessionSynchronizer: IProviderSessionSynchronizer;
}
```

## 四大 Provider 执行引擎对比

| Provider | 执行方式 | SDK/库 | 通信方式 |
|----------|----------|--------|---------|
| Claude | SDK 直接调用 | `@anthropic-ai/claude-agent-sdk` | Node.js 进程内 |
| Codex | SDK 直接调用 | `@openai/codex-sdk` | Node.js 进程内 |
| Cursor | 子进程 | `child_process` + cross-spawn | `cursor-agent` CLI |
| Gemini | 子进程 | `child_process` + cross-spawn | `gemini` CLI |

所有四个引擎实现相同的接口: `(command, options, ws) => Promise<void>` + `abort()` / `isActive()` / `getActive()`。

## Provider 切换机制

- **前端**: `useChatProviderState` hook → `localStorage.setItem('selected-provider', id)`
- **后端 WebSocket**: 根据消息类型 `claude-command` / `codex-command` / `cursor-command` / `gemini-command` 分发
- **后端 REST**: `/api/agent` 路由根据 `provider` 参数选择执行引擎

## 关键设计模式

| 模式 | 位置 | 说明 |
|------|------|------|
| 抽象工厂 | `provider.registry.ts` + `AbstractProvider` | 按 id 创建 auth/mcp/skills/sessions/sessionSynchronizer 产品族 |
| 适配器模式 | `list/*/` 六个文件 | 将各 SDK/CLI 差异适配到统一 IProvider |
| 策略模式 | `sessions.service.ts` | 按 provider 选择不同的 normalizeMessage 实现 |
| 模板方法 | `mcp.provider.ts` (McpProvider) | 定义 MCP 操作骨架，子类实现具体读写格式 |
| 依赖注入 | WebSocket 服务 | 所有 provider 执行函数作为依赖注入 |
| 外观模式 | `services/*.service.ts` | 封装注册中心访问，提供简洁 API |

## 核心文件

| 文件 | 职责 |
|------|------|
| `server/shared/interfaces.ts` | IProvider 五面接口定义 |
| `server/shared/types.ts` | LLMProvider、NormalizedMessage 核心类型 |
| `server/modules/providers/provider.registry.ts` | Provider 注册中心 |
| `server/modules/providers/shared/base/abstract.provider.ts` | 抽象基类 |
| `server/modules/providers/shared/mcp/mcp.provider.ts` | MCP 模板基类 |
| `server/modules/providers/services/sessions.service.ts` | 消息归一化服务 |
| `server/modules/providers/services/session-synchronizer.service.ts` | 会话同步服务 |
| `server/claude-sdk.js` | Claude SDK 执行引擎 |
| `server/openai-codex.js` | Codex SDK 执行引擎 |
| `server/cursor-cli.js` | Cursor CLI 执行引擎 |
| `server/gemini-cli.js` | Gemini CLI 执行引擎 |
| `server/modules/websocket/services/chat-websocket.service.ts` | WebSocket 消息分发 |
| `server/routes/agent.js` | REST API Agent 路由 |
