# agent-design-analysis

AI 编码工具的**统一调度架构**深度研究——对 7 个应用项目和 3 个底层 SDK 的全方位对比分析，并补充 2 个 Agent/Harness 参考项目，覆盖架构设计、消息格式、生命周期、权限机制和 UI 渲染五个维度。

## 目录结构

```
agent-design-analysis/
├── codes/                          # 参考代码（git submodule × 13）
│   ├── acpx/                       # ACP CLI 客户端
│   ├── acp-typescript-sdk/         # ACP 协议 TypeScript SDK
│   ├── agentscope-java/            # AgentScope Java 框架
│   ├── agentapi/                   # PTY HTTP API 服务器
│   ├── AionUi/                     # 桌面 AI 应用
│   ├── anthropic-sdk-typescript/   # Anthropic SDK
│   ├── ClawTeam/                   # 多 Agent 群体协作框架
│   ├── claudecodeui/               # Claude Code Web UI
│   ├── jetbrains-cc-gui/           # JetBrains IDE 插件
│   ├── learn-claude-code/          # Claude Code Harness 工程教程
│   ├── lobehub/                    # LobeHub Web 应用
│   ├── openai-node/                # OpenAI Node.js SDK
│   └── Proma/                      # Proma 桌面工作台
├── docs/
│   ├── unified-agent-architecture/ # 统一架构范式
│   ├── message-formats/            # 消息格式标准化
│   ├── agent-lifecycle/            # Agent 会话与进程生命周期
│   ├── tool-permissions/           # 工具调用与权限机制
│   ├── ui-rendering/               # UI 渲染层（流式 + 历史）
│   ├── sdk-protocols/              # 底层 SDK 协议接口定义
│   └── agentscope-java/            # AgentScope Java HITL 与 Tool Suspend 机制
└── README.md
```

## 研究范围

### 应用层（7 个项目）

| 项目 | 类型 | 技术栈 | 核心范式 |
|------|------|--------|---------|
| **CC GUI** | JetBrains IDE 插件 | Java + Node.js + React 19 + JCEF | 文件系统中介 |
| **Claude Code UI** | Web UI | Express + React 18 + Tailwind CSS 3 | WebSocket 直连 |
| **acpx** | CLI 客户端 | TypeScript + Commander | ACP 协议 |
| **AionUi** | 桌面应用 | Electron + React 19 + Arco Design | ACP + 工厂模式 |
| **AgentAPI** | HTTP API 服务器 | Go + Next.js 15 | PTY 终端仿真 / ACP |
| **LobeHub** | Web 应用 | Next.js 16 + React 19 + @lobehub/ui | API 协议工厂 |
| **Proma** | 桌面工作台 | Electron + React 18 + Jotai + shadcn/ui | 双模式混合 |

### 协议层（3 个 SDK）

| SDK | 协议 | 传输 |
|-----|------|------|
| **openai-node** | OpenAI Chat Completions + Responses | HTTP SSE (`[DONE]` 哨兵) |
| **anthropic-sdk-typescript** | Anthropic Messages API | HTTP SSE（命名事件） |
| **acp-typescript-sdk** | Agent Client Protocol | JSON-RPC 2.0 over NDJSON |

### 参考扩展（2 个项目）

| 项目 | 类型 | 核心关注 |
|------|------|---------|
| **ClawTeam** | 多 Agent 群体协作框架 | Agent Swarm、团队模板、任务委派与协作执行 |
| **Learn Claude Code** | Claude Code Harness 工程教程 | Agent Loop、工具系统、上下文管理、权限边界 |

## 五种架构范式

| 范式 | 代表项目 | 核心思路 |
|------|---------|---------|
| **私有协议适配层** | CC GUI, Claude Code UI | 定义内部抽象接口，每个 AI 工具实现该接口 |
| **标准化协议** | acpx, AionUi | ACP (Agent Client Protocol) — JSON-RPC 2.0 over stdio |
| **终端仿真** | AgentAPI | PTY 伪终端控制 CLI，屏幕差异算法解析输出 |
| **API 协议工厂** | LobeHub | 工厂模式生成 Provider，统一 OpenAI/Anthropic HTTP 协议 |
| **双模式混合** | Proma | Chat (Provider Adapter + SSE) + Agent (SDK child_process) 并存 |

## 文档索引

### [unified-agent-architecture/](docs/unified-agent-architecture/)

统一架构范式分析，包含项目全景对比表（类型、技术栈、协议、消息标准化、进程模式、设计模式）和五种架构范式的详细说明。

### [message-formats/](docs/message-formats/)

消息格式标准化分析，对比 7 个项目的归一化消息类型（NormalizedMessage、TMessage、AgentEvent 等）、消息类别覆盖矩阵（11 种消息类别）、流式处理策略（替换式/追加式/合并式/原子写入式）。

### [agent-lifecycle/](docs/agent-lifecycle/)

会话与进程生命周期分析，对比状态机设计（隐式到 7 态 FSM）、进程启动/关闭策略、重连与恢复机制、Proma 独有的 forkSession / rewindSession 功能。

### [tool-permissions/](docs/tool-permissions/)

工具调用与权限机制分析，对比 5 种权限流转范式（文件系统中介/WebSocket/ACP/SDK 回调/指令系统）、权限策略配置矩阵（全部批准/只读/选择性/结构化检测）、权限记忆机制。

### [ui-rendering/](docs/ui-rendering/)

UI 渲染层分析，对比技术栈（React/Vue/CLI）、流式策略（逐帧更新/re-parse/stdout.write/IPC/原子写入）、消息组件清单（文本/思考/工具/权限/计划/压缩）、Session 存储方案。

### [sdk-protocols/](docs/sdk-protocols/)

底层 SDK 协议接口定义，包含 OpenAI Chat Completions（6 种角色 + ContentPart）、Anthropic Messages（ContentBlock[] 强类型 + Extended Thinking）、ACP（JSON-RPC 2.0 双向 + Session 管理 + 权限流程）的完整请求/响应结构参考。

### [agentscope-java/](docs/agentscope-java/)

AgentScope Java 框架的 HITL（Human-in-the-Loop）与 Tool Suspend 机制深度分析，包含双层挂起架构（ToolSuspendException 工具级挂起 + Hook.stopAgent Agent 级暂停）、ReActAgent 主循环中的恢复原理、SubAgent/自定义 AgentTool 嵌套场景下的 HITL 支持方案对比。

## 克隆仓库

本项目包含 git submodule，克隆时请使用：

```bash
git clone --recurse-submodules <repo-url>
```

如果已克隆但未拉取 submodule，执行：

```bash
git submodule update --init --recursive
```

## 子模块

| 路径 | 来源 |
|------|------|
| `codes/acpx` | https://github.com/openclaw/acpx |
| `codes/acp-typescript-sdk` | https://github.com/agentclientprotocol/typescript-sdk |
| `codes/agentscope-java` | https://github.com/agentscope-ai/agentscope-java |
| `codes/agentapi` | https://github.com/coder/agentapi |
| `codes/AionUi` | https://github.com/iOfficeAI/AionUi |
| `codes/anthropic-sdk-typescript` | https://github.com/anthropics/anthropic-sdk-typescript |
| `codes/ClawTeam` | https://github.com/HKUDS/ClawTeam |
| `codes/claudecodeui` | https://github.com/siteboon/claudecodeui |
| `codes/jetbrains-cc-gui` | https://github.com/zhukunpenglinyutong/jetbrains-cc-gui |
| `codes/learn-claude-code` | https://github.com/shareAI-lab/learn-claude-code |
| `codes/lobehub` | https://github.com/lobehub/lobehub |
| `codes/openai-node` | https://github.com/openai/openai-node |
| `codes/Proma` | https://github.com/ErlichLiu/Proma |
