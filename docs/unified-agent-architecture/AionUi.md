# AionUi 架构分析

## 项目概述

AionUi 是一个 Electron + React 桌面应用，将命令行 AI Agent 转变为现代化 AI 聊天界面。采用 **两层抽象架构**：底层通过 ClientFactory 统一调用 LLM API，上层通过 ACP 协议统一调度 CLI Agent。

- 仓库: https://github.com/iOfficeAI/AionUi
- 技术栈: Electron + React + TypeScript + Vite + Bun

## 统一抽象层 — 两层架构

### 第一层: LLM API 抽象 — `ProtocolConverter` + `ClientFactory`

`src/common/api/ProtocolConverter.ts` — 协议转换器接口：

```typescript
export interface ProtocolConverter<TInput, TOutput, TResponse> {
  convertRequest(input: TInput): TOutput;
  convertResponse(response: any, originalModel: string): TResponse;
}
```

`src/common/api/ClientFactory.ts` — 工厂类，根据 `AuthType` 创建对应客户端：

```typescript
export class ClientFactory {
  static async createRotatingClient(provider, options) {
    switch (getProviderAuthType(provider)) {
      case AuthType.USE_OPENAI:    return new OpenAIRotatingClient(...)
      case AuthType.USE_GEMINI:    return new GeminiRotatingClient(...)
      case AuthType.USE_VERTEX_AI: return new GeminiRotatingClient(...)
      case AuthType.USE_ANTHROPIC: return new AnthropicRotatingClient(...)
      default:                     return new OpenAIRotatingClient(...)
    }
  }
}
```

`RotatingApiClient<T>` 抽象基类提供 API Key 轮换、自动重试、错误处理。

支持 20+ 个 LLM 平台：OpenAI、Gemini、Anthropic、DeepSeek、Qwen、Zhipu、Moonshot、OpenRouter、Ollama 等。

### 第二层: CLI Agent 抽象 — ACP 协议

通过 ACP (Agent Communication Protocol) 统一调度 17 种 CLI Agent：

| Backend | CLI 命令 | ACP 参数 |
|---------|---------|---------|
| claude | `claude` | `--experimental-acp` |
| qwen | `qwen` | `--acp` |
| codex | `codex` | (默认 ACP) |
| codebuddy | `codebuddy` | `--acp` |
| goose | `goose` | `acp` |
| auggie | `auggie` | `--acp` |
| kimi | `kimi` | `acp` |
| opencode | `opencode` | `acp` |
| droid | `droid` | `exec --output-format acp` |
| copilot | `copilot` | `--acp --stdio` |
| qoder | `qodercli` | `--acp` |
| vibe | `vibe-acp` | (默认) |
| cursor | `agent` | `acp` |
| kiro | `kiro-cli` | `acp` |
| hermes | `hermes` | `acp` |
| snow | `snow` | `--acp` |
| custom | 用户配置 | 用户定义 |

非 ACP 的执行引擎：Gemini、Aion CLI (Rust)、OpenClaw Gateway、Nanobot、Remote Agent。

### ACP 消息适配器

`src/process/agent/acp/AcpAdapter.ts` — 将 ACP 协议更新转换为内部 `TMessage` 格式：

```
ACP SessionUpdate → AcpAdapter → TMessage (text / tool_call / plan / ...)
```

### ACP 会话状态机

`src/process/acp/session/AcpSession.ts` — 有限状态机管理会话生命周期：

```
idle → starting → active → prompting → active → ...
                  ↓
                error
```

## Provider 切换机制

用户创建对话时选择 Agent 类型，每种 Agent 使用独立的会话类型（`TChatConversation` 泛型参数）：

```typescript
type TChatConversation = 
  | TBaseConversation<'gemini', ...>
  | TBaseConversation<'acp', ...>
  | TBaseConversation<'codex', ...>
  | TBaseConversation<'openclaw-gateway', ...>
  | TBaseConversation<'nanobot', ...>
  | TBaseConversation<'remote', ...>
  | TBaseConversation<'aionrs', ...>
```

Agent 注册表（`AgentRegistry`）负责检测和注册可用的 CLI Agent，合并优先级: `Aionrs > Gemini > Builtin > Other > Remote > Extension > Custom`。

## 关键设计模式

| 模式 | 位置 | 说明 |
|------|------|------|
| 抽象工厂 | `ClientFactory` | 根据 AuthType 创建不同 SDK 客户端 |
| 适配器模式 | 通信适配器 (`adapter/`) + 协议转换器 + ACP 适配器 | 三层适配 |
| 注册表模式 | `AgentRegistry` | 中央注册所有检测到的执行引擎 |
| 模板方法 | `RotatingApiClient` | API Key 轮换 + 自动重试的固定模板 |
| 状态机 | `AcpSession` | ACP 会话生命周期的形式化状态管理 |
| 桥接模式 | `IPlatformServices` | 抽象 Electron 和 Node.js 的平台差异 |
| 代理模式 | `registry.ts` WebSocket 广播器 | 发布-订阅多播 |

## 核心文件

| 文件 | 职责 |
|------|------|
| `src/common/api/ClientFactory.ts` | LLM API 客户端工厂 |
| `src/common/api/RotatingApiClient.ts` | API 抽象基类（重试+轮换） |
| `src/common/api/ProtocolConverter.ts` | 协议转换器接口 |
| `src/common/types/acpTypes.ts` | ACP 核心类型 + 后端配置列表 |
| `src/common/types/detectedAgent.ts` | 检测到的 Agent 类型定义 |
| `src/process/agent/AgentRegistry.ts` | Agent 执行引擎注册表 |
| `src/process/agent/acp/AcpDetector.ts` | ACP CLI 检测器 |
| `src/process/agent/acp/AcpConnection.ts` | ACP 连接管理器 |
| `src/process/agent/acp/AcpAdapter.ts` | ACP → 内部消息适配器 |
| `src/process/agent/acp/acpConnectors.ts` | ACP 后端连接器 |
| `src/process/acp/session/AcpSession.ts` | ACP 会话状态机 |
| `src/common/platform/IPlatformServices.ts` | 平台抽象接口 |
| `src/common/adapter/main.ts` | 主进程通信适配器 |
| `src/common/config/storage.ts` | 会话详情配置类型 |
