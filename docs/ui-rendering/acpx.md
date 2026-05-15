# acpx — UI 渲染层

## 架构特点

**纯 CLI 项目**，无 GUI 前端。渲染完全依赖终端输出，通过 TextOutputFormatter 实现流式效果。

## 技术栈

| 层级 | 技术 |
|------|------|
| 运行时 | Node.js + TypeScript |
| CLI 框架 | Commander |
| 输出格式 | Text（默认）/ JSON / Quiet |
| 颜色方案 | ANSI escape codes |
| Session 存储 | `~/.acpx/sessions/*.json` |

## 三种输出格式化器

### TextOutputFormatter — 默认交互式格式

```typescript
class TextOutputFormatter implements OutputFormatter {
  // 直接写入 stdout，实现打字机效果
  formatTextDelta(delta: string): void {
    process.stdout.write(delta);
    // 不做任何缓冲或解析
    // 终端自行处理换行和渲染
  }

  formatToolCall(toolCall: ToolCall): void {
    // "[tool] Bash: ls -la"
    process.stdout.write(`\n${colors.bold('[tool]')} ${toolCall.title}\n`);
  }

  formatToolResult(result: ToolResult): void {
    // 缩进显示工具输出
    process.stdout.write(result.content);
  }
}
```

### JSONOutputFormatter — 结构化输出

```typescript
class JSONOutputFormatter implements OutputFormatter {
  // 每个事件一行 JSON（JSONL 格式）
  formatTextDelta(delta: string): void {
    const event = JSON.stringify({
      type: 'text_delta',
      delta,
      timestamp: Date.now(),
    });
    process.stdout.write(event + '\n');
  }
}
```

### QuietOutputFormatter — 静默模式

```typescript
class QuietOutputFormatter implements OutputFormatter {
  // 仅输出最终结果，不输出流式过程
  // 适用于脚本和管道场景
}
```

## 流式渲染机制

```
Agent 响应 → AcpRuntimeEvent (text_delta)
  → TextOutputFormatter.formatTextDelta()
    → process.stdout.write(delta)
      → 终端原生渲染（打字机效果）
```

### ANSI 颜色方案

```typescript
const colors = {
  bold:    '\x1b[1m',
  dim:     '\x1b[2m',
  red:     '\x1b[31m',
  green:   '\x1b[32m',
  yellow:  '\x1b[33m',
  blue:    '\x1b[34m',
  cyan:    '\x1b[36m',
  reset:   '\x1b[0m',
};

// 工具调用行: [tool] Read · file.ts  (蓝色加粗 [tool])
// 错误信息: 红色文本
// 推理文本: 灰色/暗淡
```

### 排版规则

```
用户消息:   ───────────────────────── (分隔线)
            用户输入文本

Agent 回复: 直接流式输出，无前缀
            [tool] 工具调用 — 加粗 + 蓝色

推理过程:   灰色调暗显示（dim）
            默认折叠（可通过配置展开）
```

## 工具调用显示

### 工具调用行格式

```
[tool] <kind> · <title>
[tool] Read · src/index.ts
[tool] Bash · npm install
[tool] Edit · package.json
```

### 工具结果格式

```
[tool result] — 缩进显示
  > 文件内容 / 命令输出 ...
```

## 权限交互（TTY 模式）

```typescript
// promptForToolPermission() — 终端交互式询问
async function promptForToolPermission(tool: ToolCall): Promise<Decision> {
  // 终端输出:
  // ╔══════════════════════════════════════╗
  // ║ Tool: Read                           ║
  // ║ File: src/secret.ts                  ║
  // ║ Do you want to allow? [y/N/a]        ║
  // ║ y=once N=deny a=always allow         ║
  // ╚══════════════════════════════════════╝
  //
  // 读取 stdin → 返回决策
}
```

## 历史消息（Session）存储

### 存储路径

```
~/.acpx/
  ├── sessions/
  │   ├── <session-id-1>.json
  │   ├── <session-id-2>.json
  │   └── index.json              # 会话索引
  └── config.json
```

### Session JSON 结构

```json
{
  "id": "session-uuid",
  "created": "2024-01-15T10:30:00Z",
  "updated": "2024-01-15T11:45:00Z",
  "messages": [
    {
      "role": "user",
      "content": "请读取文件",
      "timestamp": "2024-01-15T10:30:00Z"
    },
    {
      "role": "assistant",
      "content": "正在读取...",
      "toolCalls": [...],
      "timestamp": "2024-01-15T10:30:05Z"
    }
  ],
  "permissionStats": {
    "requested": 5,
    "approved": 4,
    "denied": 1
  }
}
```

### 历史回放

```
acpx session resume <id> → 加载 JSON → 恢复上下文
acpx session list       → 读取 index.json → 表格输出
```

### 历史渲染特点

- 不渲染为对话格式，仅用于上下文恢复
- 会话列表通过 `index.json` 快速索引
- JSON 文件可直接用任何工具查看
- 无分页、无虚拟滚动（纯文件读写）
