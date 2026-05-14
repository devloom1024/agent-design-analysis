# acpx 架构分析

## 项目概述

acpx 是一个 **headless CLI 客户端**，通过 **Agent Client Protocol (ACP)** —— 一种基于 JSON-RPC 2.0 over stdio 的标准化协议 —— 统一调用各种 AI 编码工具。核心思想是"一个协议，多种适配器"。

- 仓库: https://github.com/openclaw/acpx
- 技术栈: TypeScript + Commander.js + Zod
- 核心依赖: `@agentclientprotocol/sdk`

## 统一抽象层

### 架构全景

```
用户/编排器
    ↓ (统一 CLI: acpx codex "prompt", acpx claude "prompt")
acpx (ACP 客户端)
    ↓ (JSON-RPC 2.0 over stdio)
ACP 适配器层  ← 每个工具有一个适配器
    ↓ (调用原生 CLI)
原生 Coding Agent  ← Claude Code, Codex CLI, Gemini CLI 等
```

### 底层: `AcpClient` 传输层

`src/acp/client.ts` — ACP 协议传输实现，管理子进程并通过 NDJSON/stdio 通信：

- `initialize` → 协商协议版本和能力
- `session/new` → 创建新会话
- `session/prompt` → 发送提示并流式接收响应
- `session/cancel` → 取消正在运行的提示
- `session/set_mode` → 设置会话模式
- `session/close` → 关闭会话

### 高层: `AcpRuntime` 运行时

`src/runtime/public/contract.ts` — 编程化使用的高层抽象：

```typescript
export interface AcpRuntime {
  ensureSession(input): Promise<AcpRuntimeHandle>;
  startTurn(input): AcpRuntimeTurn;
  runTurn(input): AsyncIterable<AcpRuntimeEvent>;
  cancel(input): Promise<void>;
  close(input): Promise<void>;
}
```

### 统一事件模型

```typescript
type AcpRuntimeEvent =
  | { type: "text_delta"; text: string }
  | { type: "status"; text: string }
  | { type: "tool_call"; text: string; title?: string }
  | { type: "done"; stopReason?: string }
  | { type: "error"; message: string; code?: string };
```

## 支持的 Provider（16 个内置）

| 名称 | 适配器命令 | 包装工具 |
|------|-----------|---------|
| `pi` | `npx pi-acp` | Pi Coding Agent |
| `openclaw` | `openclaw acp` | OpenClaw |
| `codex` | `npx @zed-industries/codex-acp` | Codex CLI (OpenAI) |
| `claude` | `npx @agentclientprotocol/claude-agent-acp` | Claude Code (Anthropic) |
| `gemini` | `gemini --acp` | Gemini CLI (Google) |
| `cursor` | `cursor-agent acp` | Cursor CLI |
| `copilot` | `copilot --acp --stdio` | GitHub Copilot |
| `droid` | `droid exec --output-format acp` | Factory Droid |
| `iflow` | `iflow --experimental-acp` | iFlow |
| `kilocode` | `npx @kilocode/cli acp` | Kilocode |
| `kimi` | `kimi acp` | Kimi CLI (MoonshotAI) |
| `kiro` | `kiro-cli-chat acp` | Kiro CLI |
| `opencode` | `npx opencode-ai acp` | OpenCode |
| `qoder` | `qodercli --acp` | Qoder CLI |
| `qwen` | `qwen --acp` | Qwen Code |
| `trae` | `traecli acp serve` | Trae CLI |

## 适配器的三种形态

1. **npm 包适配器**: `pi`, `codex`, `claude`, `kilocode`, `opencode` — 通过 `npx` 分发
2. **原生内置适配器**: `openclaw`, `gemini`, `cursor`, `copilot`, `droid`, `iflow`, `kimi`, `kiro`, `qoder`, `qwen`, `trae` — CLI 工具自带 ACP 支持
3. **自定义适配器**: 通过 `--agent` 参数或配置文件指定任意可执行程序

## Provider 切换机制

```bash
acpx codex "fix the tests"       # 直接切换到 Codex
acpx claude "refactor auth"      # 直接切换到 Claude
acpx gemini "review this PR"     # 直接切换到 Gemini
```

- **默认代理**: 通过 `~/.acpx/config.json` 的 `defaultAgent` 设置，默认 `codex`
- **优先级链**: `--agent` 覆盖 > 配置文件 agents > 内置注册表

## 关键设计模式

| 模式 | 位置 | 说明 |
|------|------|------|
| 适配器模式 | ACP 适配器 + `agent-command.ts` | 将各 CLI 差异适配到统一的 ACP 协议 |
| 注册表模式 | `agent-registry.ts` | 16 个内置代理的名称→命令映射 |
| 工厂模式 | `createAcpRuntime()`, `AcpClient` | 创建核心对象 |
| 策略模式 | 权限策略 / 输出格式策略 | `approve-all` / `approve-reads` / `deny-all` |
| 异步队列模式 | `src/cli/queue/` | 同一会话的 prompt 排队处理 |

## 核心文件

| 文件 | 职责 |
|------|------|
| `src/agent-registry.ts` | 代理注册表（16 个内置代理） |
| `src/acp/client.ts` | ACP 传输层客户端 |
| `src/acp/agent-command.ts` | 各代理特有命令适配 |
| `src/runtime/engine/manager.ts` | 高层运行时管理器 |
| `src/runtime/public/contract.ts` | 公共接口定义 |
| `src/runtime.ts` | 运行时入口 |
| `src/cli-core.ts` | CLI 主入口 |
| `src/cli-public.ts` | 动态注册 agent 子命令 |
| `src/cli/config.ts` | 配置文件加载 |
| `src/types.ts` | 核心类型定义 |
| `conformance/spec/v1.md` | ACP 协议规范 |
