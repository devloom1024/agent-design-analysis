# LobeHub 架构分析

## 项目概述

LobeHub 是一个开源的综合性 AI Agent 框架，支持 80+ 大语言模型 Provider、语音合成、多模态和可扩展的 Function Call 插件系统。采用 **pnpm monorepo** 架构，通过工厂模式 + Router Runtime 实现多 Provider 的统一调度。

- 仓库: https://github.com/lobehub/lobehub
- 技术栈: Next.js + React 19 + TypeScript + pnpm monorepo

## 统一抽象层

### 核心接口: `LobeRuntimeAI`

`packages/model-runtime/src/core/BaseAI.ts` — 所有 AI Provider 必须实现的统一接口：

```typescript
export interface LobeRuntimeAI {
  baseURL?: string;
  chat?: (payload: ChatStreamPayload, options?) => Promise<Response>;
  createImage?: (...) => Promise<CreateImageResponse>;
  createVideo?: (...) => Promise<CreateVideoResponse>;
  embeddings?: (...) => Promise<Embeddings[]>;
  generateObject?: (...) => Promise<any>;
  textToSpeech?: (...) => Promise<ArrayBuffer>;
  models?: () => Promise<any>;
  pullModel?: (...) => Promise<Response>;
}
```

大部分方法为可选，Provider 只需实现自己支持的能力。

### 统一请求格式: `ChatStreamPayload`

`packages/model-runtime/src/types/chat.ts` — 所有 Provider 共用的请求格式（兼容 OpenAI 格式）：

```typescript
interface ChatStreamPayload {
  model: string;
  messages: OpenAIChatMessage[];
  temperature?: number;
  top_p?: number;
  max_tokens?: number;
  tools?: ChatCompletionTool[];
  stream?: boolean;
  provider: string;
  thinking?: {...};
  enabledSearch?: boolean;
}
```

### 外观类: `ModelRuntime`

`packages/model-runtime/src/core/ModelRuntime.ts` — 包装 `LobeRuntimeAI`，注入生命周期钩子：

```typescript
export class ModelRuntime {
  private _runtime: LobeRuntimeAI;

  async chat(payload, options?) { ... }
  async generateObject(payload, options?) { ... }
  async createImage(payload, options?) { ... }
  async embeddings(payload, options?) { ... }
  async textToSpeech(payload, options?) { ... }

  static initializeWithProvider(provider: string, params, hooks?) {
    const providerAI = providerRuntimeMap[provider] ?? LobeOpenAI;
    return new ModelRuntime(new providerAI(params), hooks);
  }
}
```

## Provider 的四种创建方式

### 1. OpenAI 兼容工厂

`packages/model-runtime/src/core/openaiCompatibleFactory/index.ts`

```typescript
export const createOpenAICompatibleRuntime = ({provider, baseURL, ...}) => {
  return class LobeOpenAICompatibleAI implements LobeRuntimeAI {
    client!: OpenAI;
    async chat(payload, options) { /* 使用 OpenAI SDK */ }
    async models() { ... }
    // ...
  };
};
```

适用于大多数兼容 OpenAI 格式的 Provider。

### 2. Anthropic 兼容工厂

`packages/model-runtime/src/core/anthropicCompatibleFactory/index.ts`

```typescript
export const createAnthropicCompatibleRuntime = ({provider, ...}) => {
  return class LobeAnthropicCompatibleAI implements LobeRuntimeAI {
    client!: Anthropic;
    async chat(payload, options) { /* 使用 Anthropic SDK */ }
  };
};
```

### 3. 自定义实现

适用于无法通过工厂创建的 Provider（如 Google Gemini 使用自己的 GenAI SDK）。

### 4. Router Runtime（多协议路由）

`packages/model-runtime/src/core/RouterRuntime/createRuntime.ts`

```typescript
export const createRouterRuntime = ({id, routers, models, ...}) => {
  return class UniformRuntime implements LobeRuntimeAI {
    async chat(payload, options?) {
      return this.runWithFallback(payload.model, (runtime) => runtime.chat!(payload, options));
    }
  };
};
```

`runWithFallback` 实现链式 failover：根据模型名匹配 router → 按序尝试每个 option → 失败自动 fallback。

典型用例 — DeepSeek 同时支持 OpenAI 和 Anthropic 格式：

```typescript
routers: [
  { model: 'deepseek-r1', options: [{ apiType: 'anthropic', baseURL: '...' }] },
  { model: '*', options: [{ apiType: 'openai', baseURL: '...' }] },
]
```

## 支持的 Provider 规模

`providerRuntimeMap` 包含 **80+ 个 Provider**，包括 OpenAI、Anthropic、Google、DeepSeek、Qwen、Zhipu、Moonshot、OpenRouter、Groq、Together、Perplexity、Mistral、xAI 等，以及各类 Coding Plan Provider。

## Provider 切换机制

**服务端初始化**: `src/server/modules/ModelRuntime/index.ts`

```typescript
export const initModelRuntimeWithUserPayload = (provider, payload, params, hooks) => {
  const runtimeProvider = payload.runtimeProvider ?? provider;
  return ModelRuntime.initializeWithProvider(runtimeProvider, params, hooks);
};
```

- **内置 Provider**: 通过 `ModelProvider` 枚举匹配 → 从 `providerRuntimeMap` 获取类
- **自定义 Provider**: 通过 `sdkType` 字段选择底层引擎（默认 `openai`）
- **用户配置**: 从数据库读取 API Key、baseURL 等

## Claude Code / OpenCode 集成方式

### Claude Code — 作为 Builtin Tool

`packages/builtin-tool-claude-code/` — Claude Code 不是 Model Provider，而是作为**内置工具**集成。定义了 Claude Code 的工具 API（Agent、Bash、Edit、Glob、Grep、Read、Write、Skill 等），在 Tool Calling 流程中被调用。

### OpenCode — 作为 Model Provider

两个 Provider：
- **OpenCode Coding Plan** (`providers/opencodeCodingPlan/`) — 代理编码计划，使用 Router Runtime
- **OpenCode Zen** (`providers/opencodeZen/`) — 自动检测模型类型并路由：Claude 模型 → Anthropic API、GPT-5.x → OpenAI Responses API、其他 → OpenAI API

## 完整调用链路

```
用户消息 → API Route
  → initModelRuntimeWithUserPayload(provider, payload)
    → ModelRuntime.initializeWithProvider(provider, params, hooks)
      → providerRuntimeMap[provider]  // 查找 Provider 类
      → new ProviderClass(params)     // 实例化
      → new ModelRuntime(instance)    // 包装
    ↓
ModelRuntime.chat(payload, options)
  → this._runtime.chat(payload)
    ↓
[OpenAI Compatible]: OpenAI SDK → OpenAIStream
[Anthropic Compatible]: Anthropic SDK → AnthropicStream
[Router Runtime]: runWithFallback → 匹配路由 → 创建子 Runtime → 委托调用
```

## 关键设计模式

| 模式 | 位置 | 说明 |
|------|------|------|
| 工厂模式 | `openaiCompatibleFactory`, `anthropicCompatibleFactory`, `RouterRuntime/createRuntime` | 三大工厂函数创建 Provider 实例 |
| 策略模式 | `runtimeMap.ts` (80+ providers) | 所有 Provider 实现相同接口，可随时切换 |
| 适配器模式 | 工厂内部 `LobeOpenAICompatibleAI` / `LobeAnthropicCompatibleAI` | 将不同 SDK 适配到统一接口 |
| 外观模式 | `ModelRuntime` | 包装 `LobeRuntimeAI`，提供统一 API + 生命周期钩子 |
| 责任链/Failover | `RouterRuntime.runWithFallback()` | 多路由链式尝试 |
| 模板方法 | 工厂函数的默认实现 | 提供默认流程，通过配置选项定制 |
| 抽象工厂 | `ModelRuntime.initializeWithProvider()` | 根据名称创建对应实例 |

## 核心文件

| 文件 | 职责 |
|------|------|
| `packages/model-runtime/src/core/BaseAI.ts` | LobeRuntimeAI 统一接口 |
| `packages/model-runtime/src/core/ModelRuntime.ts` | ModelRuntime 外观类 |
| `packages/model-runtime/src/runtimeMap.ts` | 全局 Provider 注册表（80+） |
| `packages/model-runtime/src/core/openaiCompatibleFactory/index.ts` | OpenAI 兼容工厂 |
| `packages/model-runtime/src/core/anthropicCompatibleFactory/index.ts` | Anthropic 兼容工厂 |
| `packages/model-runtime/src/core/RouterRuntime/createRuntime.ts` | Router Runtime 工厂 |
| `packages/model-runtime/src/core/RouterRuntime/baseRuntimeMap.ts` | API 类型 → Runtime 类映射 |
| `packages/model-runtime/src/types/chat.ts` | 统一请求格式 ChatStreamPayload |
| `packages/model-runtime/src/providers/openai/index.ts` | OpenAI Provider |
| `packages/model-runtime/src/providers/anthropic/index.ts` | Anthropic Provider |
| `packages/model-runtime/src/providers/opencodeZen/index.ts` | OpenCode Zen（Router Runtime） |
| `packages/builtin-tool-claude-code/` | Claude Code 工具集成 |
| `packages/agent-runtime/src/core/runtime.ts` | Agent 执行引擎 |
| `src/server/modules/ModelRuntime/index.ts` | 服务端初始化入口 |
| `src/envs/llm.ts` | 环境变量配置 |
