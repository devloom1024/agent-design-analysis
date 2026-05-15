# 工具调用与权限机制分析

对 6 个项目的工具调用（Tool Call）流转路径和权限（Permission）决策机制的详细对比研究。

## 权限机制全景对比

| 维度 | CC GUI | Claude Code UI | acpx | AgentAPI | AionUi | LobeHub |
|------|--------|---------------|------|----------|--------|---------|
| **传输协议** | 文件系统 JSON | WebSocket | ACP JSON-RPC stdio | ACP stdio / PTY | ACP JSON-RPC stdio | 内存指令/事件 |
| **权限回调** | 文件轮询 + CompletableFuture | canUseTool Promise + Map | resolvePermissionRequest | ACP RequestPermission | handlePermissionRequest + Promise | request_human_approve |
| **自动批准** | ALLOW_ALWAYS 记忆 | allowedTools / bypassPermissions | approve-all / approve-reads | 硬编码 auto-approve | ApprovalStore 缓存 | userInterventionConfig |
| **工具种类推断** | 按 toolName 字符串 | matchesToolPermission + Bash模式 | inferToolKind() 基于标题 | ACP ToolCall.Kind | ACP toolCall.kind | apiName 枚举 |
| **超时策略** | 5 分钟 → DENY | 55 秒 → deny | 无（同步等待） | N/A | 30 分钟 / 团队模式无限 | N/A |
| **权限记忆** | 工具级 + 参数级 | allowedTools 白名单 | approvalStats 统计 | 无 | ApprovalStore | securityBlacklist |
| **非交互策略** | N/A (有 GUI) | N/A | deny / fail | auto-approve | N/A (有 GUI) | finish 指令 |

## 工具调用流转路径对比

### 范式对比

| 范式 | 代表项目 | 流转路径 |
|------|---------|---------|
| **文件系统中介** | CC GUI | AI → Node.js → 文件系统 JSON → Java 轮询 → 前端弹窗 → 文件响应 → Node.js → AI |
| **WebSocket 直连** | Claude Code UI | AI → SDK canUseTool → WebSocket → 前端审批 → WebSocket → resolveToolApproval → AI |
| **ACP 协议回调** | acpx, AionUi | AI → ACP session/request_permission → 策略决策 / UI → ACP Response → AI |
| **PTY 透传** | AgentAPI (PTY) | AI → PTY → CLI 自身 TUI 处理（AgentAPI 不介入） |
| **指令系统** | LobeHub | AI → runner() → AgentInstruction → InstructionExecutor → AgentEvent → UI 审批 → 继续执行 |

## 权限策略配置对比

| 策略 | CC GUI | Claude Code UI | acpx | AgentAPI | AionUi | LobeHub |
|------|--------|---------------|------|----------|--------|---------|
| 全部批准 | ALLOW_ALWAYS 记忆 | bypassPermissions | approve-all | ACP 硬编码 | YOLO 模式 | userInterventionConfig |
| 只读批准 | — | plan 模式 | approve-reads | — | 自动推断 toolKind | — |
| 全部拒绝 | — | disallowedTools | deny-all | — | — | securityBlacklist |
| 选择性批准 | 弹窗决策 | canUseTool 回调 | TTY 交互 | — | 弹窗 + ApprovalStore | human_approve_required |
| 参数级记忆 | ✓ inputs hash | — | — | — | serializeKey | — |
| 工具级记忆 | ✓ toolName Map | ✓ allowedTools | — | — | ApprovalStore Map | — |

## 各项目详情

| 项目 | 文档 |
|------|------|
| CC GUI (JetBrains 插件) | [jetbrains-cc-gui.md](./jetbrains-cc-gui.md) |
| Claude Code UI | [claudecodeui.md](./claudecodeui.md) |
| acpx | [acpx.md](./acpx.md) |
| AgentAPI | [agentapi.md](./agentapi.md) |
| AionUi | [AionUi.md](./AionUi.md) |
| LobeHub | [lobehub.md](./lobehub.md) |
