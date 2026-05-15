# AgentAPI — 工具调用与权限

## 两种模式的不同处理

### PTY 模式——CLI 自身处理

PTY 模式下，AgentAPI **不介入权限决策**。工具调用和权限由 CLI 自身的 TUI 处理（如 Claude Code 的终端交互式提示）。

AgentAPI 只负责：
- `FormatToolCall()` — 从终端输出中剥离任务报告工具调用
- `FormatMessage()` — 移除用户回显和 TUI 输入框

### ACP 模式——硬编码 auto-approve

```go
// acpio.go: 当前 Phase 1 实现
func (c *acpClient) RequestPermission(ctx context.Context, params acp.RequestPermissionRequest) 
    (acp.RequestPermissionResponse, error) {
    // Auto-approve all permissions for Phase 1
    return acp.RequestPermissionResponse{
        Outcome: acp.RequestPermissionOutcome{
            Selected: &acp.RequestPermissionOutcomeSelected{OptionId: "allow"},
        },
    }, nil
}
```

## 工具调用格式化

### FormatToolCall() 处理规则

| AgentType | 处理规则 |
|-----------|---------|
| Claude | 移除 `● coder - coder_report_task (MCP)` 模式 |
| Codex | 移除 `•Called ... Coder.coder_report_task` 模式 |
| 其他 | 直接透传 |

```go
func FormatToolCall(agentType AgentType, message string) (cleaned string, toolCalls []string)
```

### SessionUpdate ACP 工具格式化

```go
// ToolCall → "[Tool: <kind>] <title>"
if params.Update.ToolCall != nil {
    formatted = fmt.Sprintf("\n[Tool: %s] %s\n", tc.Kind, tc.Title)
}
// ToolCallUpdate → "[Tool Status: <status>]"
if params.Update.ToolCallUpdate != nil {
    formatted = fmt.Sprintf("[Tool Status: %s]\n", *tcu.Status)
}
```

## PTY vs ACP 权限差异

| 维度 | PTY | ACP |
|------|-----|-----|
| 权限处理者 | CLI 自身 TUI | agentapi ACP 回调 |
| 决策方式 | 用户终端交互 | 硬编码 auto-approve |
| 可扩展性 | 无（依赖 CLI） | 高（可实现自定义策略） |
| 状态持久化 | 不支持 | 不支持（实验性冲突检查） |

## 消息格式化中的权限规避

PTY 模式下通过消息格式化间接影响权限体验：

- **Bracketed Paste**：`\x1b[200~` / `\x1b[201~` 包裹消息，避免特殊字符被解释为键盘快捷键
- **隐藏字段**：`MessagePartText.Hidden = true` 标记转义序列不显示在历史中
- **两阶段写入**：先写入内容等待回显，再发送回车触发处理（避免与运行中的 TUI 交互冲突）
