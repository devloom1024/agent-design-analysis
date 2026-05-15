# CC GUI — 工具调用与权限

## 架构特点

使用 **文件系统中介模式**——Java 后端和 Node.js 桥接层之间通过 JSON 文件通信。

```
Claude Agent SDK → Node.js Bridge → 文件系统 (JSON) → Java PermissionService → JCEF 前端
```

## 权限决策类型

```java
// PermissionResponse 枚举
ALLOW(1, "Allow"),
ALLOW_ALWAYS(2, "Allow and don't ask again"),
DENY(3, "Deny");
```

## 完整流转路径

```
1. SDK 产生工具调用 → Node.js 侧 canUseTool 回调触发
2. 写入 CLAUDE_PERMISSION_DIR 目录下的 JSON 文件
   { requestId, toolName, inputs }
3. PermissionRequestWatcher 文件监控轮询
4. PermissionService.handlePermissionRequest() 四阶段决策:
   ├── 阶段一: 工具级记忆
   │   PermissionDecisionStore.getToolDecision(toolName)
   │   记住则直接写响应文件
   ├── 阶段二: Diff Review
   │   Edit/Write 等修改文件工具 → DiffReviewService 显示 diff
   ├── 阶段三: 参数级记忆
   │   PermissionDecisionStore.getParameterDecision(toolName, inputs)
   │   相同参数（如同文件路径）记住则直接响应
   └── 阶段四: 前端弹窗
       PermissionDialogRouter → JCEF window.showPermissionDialog(data)
5. 用户选择 Allow / Allow Always / Deny
6. PermissionHandler.handlePermissionDecision()
   - 解析 channelId, allow, remember
   - remember=true → 存入 PermissionDecisionStore
   - CompletableFuture<Integer> complete → 值写入文件
7. PermissionFileProtocol.writePermissionResponse(requestId, allow)
8. Node.js 读取文件 → 返回 SDK
```

## 权限记忆存储

**两级缓存**：
- **工具级** (`Map<String, PermissionResponse>`): key = toolName
- **参数级** (`Map<String, Map<String, PermissionResponse>>`): key = toolName → inputs hash

## 配置方式

| 配置项 | 值 |
|--------|-----|
| 文件目录 | `CLAUDE_PERMISSION_DIR` 环境变量 |
| 超时 | `PERMISSION_TIMEOUT_SECONDS = 300`（5 分钟） |
| 回退 UI | Java Swing `JOptionPane` 对话框 |
| 超时行为 | 自动 DENY |

## 人工交互工具

除常规工具权限外，还支持三种特殊交互：

```java
PermissionDialogShower       → CompletableFuture<Integer> showPermissionDialog()
AskUserQuestionDialogShower  → CompletableFuture<JsonObject> showAskUserQuestionDialog()
PlanApprovalDialogShower     → CompletableFuture<JsonObject> showPlanApprovalDialog()
```

## 消息分发

`MessageDispatcher`（责任链模式）：
- `permission_decision` → `handlePermissionDecision()`
- `ask_user_question_response` → `handleAskUserQuestionResponse()`
- `plan_approval_response` → `handlePlanApprovalResponse()`
