# CC GUI (JetBrains Plugin) 架构分析

## 项目概述

IntelliJ IDEA 插件，为 Claude Code 和 OpenAI Codex 提供图形界面。采用 **Java (JetBrains Plugin) + Node.js (ai-bridge)** 双层架构，通过统一的桥接层与不同 AI 服务的 SDK 通信。

- 仓库: https://github.com/zhukunpenglinyutong/jetbrains-cc-gui
- 技术栈: Java + Node.js + React/TypeScript (Webview)

## 统一抽象层

### Java 端: `BaseSDKBridge` 抽象类

`src/main/java/.../provider/common/BaseSDKBridge.java` — 模板方法模式的核心基类：

```java
public abstract class BaseSDKBridge {
    protected abstract String getProviderName();
    protected abstract void configureProviderEnv(Map<String, String> env, String stdinJson);
    protected abstract void processOutputLine(String line, ...);

    // 模板方法：定义进程创建→输出读取→资源清理的固定流程
    protected CompletableFuture<SDKResult> executeStreamingCommand(...);
}
```

两个具体实现：
- **ClaudeSDKBridge** — 支持 daemon（持久进程）和 per-process 两种模式
- **CodexSDKBridge** — 仅 per-process 模式

### Node.js 端: `channel-manager.js`

`ai-bridge/channel-manager.js` — 策略模式的命令分发入口：

```javascript
const providerHandlers = {
  claude: handleClaudeCommand,   // → channels/claude-channel.js (7 条命令)
  codex: handleCodexCommand,    // → channels/codex-channel.js (2 条命令)
  system: handleSystemCommand
};
```

## Daemon 模式（关键优化）

`ai-bridge/daemon.js` 是长期运行的 Node.js 进程，通过 **NDJSON over stdio** 协议处理多个请求，避免每次调用都启动新进程（省去 2-5 秒 SDK 加载时间）。

```
Java → daemon stdin:  {"id":"1","method":"claude.send","params":{...}}
daemon stdout → Java: {"id":"1","line":"[CONTENT_DELTA]..."}
daemon stdout → Java: {"id":"1","done":true,"success":true}
```

Java 端 `DaemonBridge` 管理进程生命周期：心跳检测（15s）、自动重启（最多 3 次）、请求取消。

## 支持的 Provider

| Provider | 类型 | SDK | 执行模式 |
|----------|------|-----|---------|
| Claude Code | 原生 | `@anthropic-ai/claude-agent-sdk` | daemon + per-process |
| OpenAI Codex | 原生 | `@openai/codex-sdk` | per-process |
| 第三方兼容 (8家) | 通过 Claude | 复用 Claude SDK，设置 `ANTHROPIC_BASE_URL` | 取决于配置 |

### Provider Presets（预设的第三方兼容端点）

智谱、Kimi、DeepSeek、MiniMax、小米、Qwen、OpenRouter、小米计费版 — 均通过将 `ANTHROPIC_BASE_URL` 指向各家的 Anthropic 兼容端点实现。

## Provider 切换机制

配置文件 `~/.codemoss/config.json` 管理 provider 注册：
- `ProviderManager` / `CodexProviderManager` 管理各自的 provider 列表
- 切换时更新 `config.json` 中的 `current` 字段
- 同步到 `~/.claude/settings.json` 供 SDK 读取

## 关键设计模式

| 模式 | 位置 | 说明 |
|------|------|------|
| 模板方法 | `BaseSDKBridge.executeStreamingCommand()` | 定义固定流程，子类实现解析细节 |
| 策略模式 | `channel-manager.js` providerHandlers | 按 provider 名分发命令 |
| 适配器模式 | `ClaudeStreamAdapter` | 将 SDK 流事件适配为统一消息格式 |
| 守护进程/代理 | `DaemonBridge` + `daemon.js` | 代理所有 SDK 调用，减少进程启动开销 |
| 观察者模式 | `MessageDispatcher` handler 注册/分发 | 消息到达通知所有注册的 handler |

## 核心文件

| 文件 | 职责 |
|------|------|
| `src/main/java/.../provider/common/BaseSDKBridge.java` | Java 端抽象基类 |
| `src/main/java/.../provider/common/DaemonBridge.java` | 守护进程管理器 |
| `src/main/java/.../provider/claude/ClaudeSDKBridge.java` | Claude bridge 实现 |
| `src/main/java/.../provider/codex/CodexSDKBridge.java` | Codex bridge 实现 |
| `src/main/java/.../settings/ProviderManager.java` | Provider 管理 |
| `ai-bridge/channel-manager.js` | Node.js 端策略分发入口 |
| `ai-bridge/daemon.js` | 持久守护进程 |
| `ai-bridge/channels/claude-channel.js` | Claude 命令处理 |
| `ai-bridge/channels/codex-channel.js` | Codex 命令处理 |
| `ai-bridge/config/api-config.js` | API 认证配置 |
| `ai-bridge/utils/sdk-loader.js` | SDK 动态加载 |
| `webview/src/types/provider.ts` | Provider 类型定义 + 预设 |
