# JetBrains CC GUI：桥接标签到统一 WebView Message

## 范围与入口

JetBrains CC GUI 同时接入：

- `ClaudeSDKBridge`
- `CodexSDKBridge`

两者共同注入 `ClaudeSession`。`SessionProviderRouter.getSessionMessages(provider, sessionId, cwd)` 根据 provider 调用 Claude 或 Codex 历史读取；运行时消息则分别由 `ClaudeMessageHandler` 与 `CodexMessageHandler` 解析，再统一写入 `SessionState.messages: List<ClaudeSession.Message>`。

## Stream Protocol

### Java handler 输入事件

Claude handler 支持：

| type | payload | 说明 |
| --- | --- | --- |
| `user` | JSON string | 用户消息或 tool result。 |
| `assistant` | JSON string | 完整 assistant message，包含 `message.content[]`。 |
| `thinking` | empty/string | 进入 thinking 状态。 |
| `content` | string | 非流式完整 content。 |
| `content_delta` | string | 文本增量。 |
| `thinking_delta` | string | thinking 增量。 |
| `stream_start` | empty | 开始一轮流式输出。 |
| `stream_end` | empty | 结束流式输出。 |
| `session_id` | string | provider session id。 |
| `tool_result` | JSON/string | 工具结果。 |
| `message_end` | empty | 单条消息结束。 |
| `result` | JSON string | turn result。 |
| `usage` | JSON string | token usage。 |
| `slash_commands` | JSON/list | slash command 列表。 |
| `system` | JSON/string | system message。 |
| `node_log` | string | Node bridge 日志，转发到前端。 |

Codex handler 支持：

| type | payload | 说明 |
| --- | --- | --- |
| `assistant` | JSON string | Codex assistant item/message，包含 thinking、tool_use、text。 |
| `user` | JSON string | Codex tool_result 或用户消息。 |
| `result` | JSON string | turn usage/result。 |
| `session_id` | string | Codex thread id。 |
| `event_msg` | JSON string | Codex event wrapper，当前主要读取 `token_count`。 |
| `stream_start` | empty | 开始 streaming。 |
| `stream_end` | empty | 结束 streaming。 |
| `thinking_delta` | string | thinking 增量。 |
| `content_delta` | string | content 增量。 |
| `content` | string | 兼容旧格式的完整 content。 |
| `status` | string | 状态栏消息。 |
| `message_end` | empty | 消息结束。 |

### 运行时统一消息：`ClaudeSession.Message`

```java
public static class Message {
  public enum Type { USER, ASSISTANT, SYSTEM, ERROR }
  public Type type;
  public String content;
  public long timestamp;
  public JsonObject raw;
}
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `type` | 统一角色/类别。Claude 和 Codex 都归一为 `USER`、`ASSISTANT`、`SYSTEM`、`ERROR`。 |
| `content` | 用于快速展示/搜索的纯文本。工具调用、tool_result、thinking 通常不直接拼进该字段。 |
| `timestamp` | Java 侧创建消息时的 `System.currentTimeMillis()`。历史恢复时可由原始消息重建。 |
| `raw` | provider 原始或合并后的 JSON。结构化 UI 主要依赖它渲染 thinking、tool_use、tool_result、usage。 |

### WebView 传输消息：`ClaudeMessage`

Java 通过 `MessageJsonConverter.convertMessagesToJson()` 转为：

```ts
interface ClaudeMessage {
  type: 'user' | 'assistant' | 'error' | 'task_notification' | 'notification' | string;
  content?: string;
  raw?: ClaudeRawMessage | string;
  timestamp?: string;
  isStreaming?: boolean;
  isOptimistic?: boolean;
  __turnId?: number;
  [key: string]: unknown;
}
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `type` | 从 Java `Message.Type` 小写化而来，也允许前端派生 `notification`/`task_notification`。 |
| `content` | 文本内容，错误内容会按规则截断。 |
| `raw` | 截断后的结构化 raw。只保留 `uuid/type/isMeta/text/content/message.content` 等传输需要字段，并对超长 tool_result 做截断。 |
| `timestamp` | Java 时间戳传输到前端。 |
| `isStreaming` | 前端运行态标记，用于流式合并和 loading。 |
| `isOptimistic` | 前端乐观用户消息标记。 |
| `__turnId` | 前端运行态 turn 隔离 id，避免不同 streaming turn 的 assistant fragments 被错误合并；历史消息没有该字段。 |
| index signature | 前端扩展状态，非持久化 canonical 字段。 |

### Raw content block

前端 `ClaudeContentBlock` 支持：

| block type | 字段 | 说明 |
| --- | --- | --- |
| `text` | `text` | 普通文本。 |
| `thinking` | `thinking`, `text` | Claude thinking 或 Codex reasoning。Codex handler 会把 `thinking_delta` 累加进 raw block。 |
| `tool_use` | `id`, `name`, `input` | 工具调用声明。 |
| `image` | `src`, `mediaType`, `alt` | 图片。 |
| `attachment` | `fileName`, `mediaType` | 附件。 |
| `task_notification` | `icon`, `summary`, `status` | 任务通知。 |
| `tool_result` | `tool_use_id`, `content`, `is_error` | 工具结果，属于 `ToolResultBlock`。 |

### Delta 合并策略

| 机制 | 说明 |
| --- | --- |
| `assistantContent` | 当前 assistant 文本累加器。 |
| `currentAssistantMessage` | 当前 turn 的 assistant message；delta 会更新它而不是频繁创建新消息。 |
| `MessageMerger.mergeAssistantMessage()` | 合并完整 assistant raw。`tool_use.id` 和 `tool_result.tool_use_id` 是 keyed block；没有 key 的 `text/thinking` 用“更完整内容”覆盖。 |
| `ReplayDeduplicator` | Claude partial/full message 重放去重，避免 delta 已显示后 full message 再重复追加。 |
| `StreamMessageCoalescer` | Java 到 JCEF 的 `updateMessages` 节流；小 delta 走 `onContentDelta/onThinkingDelta`，大 raw 刷新自适应降频。 |

## Message 持久化

JetBrains CC GUI 的完整对话持久化主要来自 provider 原生历史；插件自己持久化的是 session 索引缓存。

### Claude 原生历史

`ClaudeHistoryReader` 读取：

- `~/.claude/history.jsonl`
- `~/.claude/projects/{project}/...jsonl`

核心历史 message DTO：

```java
class ConversationMessage {
  public String uuid;
  public String sessionId;
  public String parentUuid;
  public String timestamp;
  public String type;
  public Message message;
  public Boolean isMeta;
  public Boolean isSidechain;
  public String cwd;

  static class Message {
    public String role;
    public Object content;
    public Usage usage;
  }

  static class Usage {
    public int input_tokens;
    public int output_tokens;
    public int cache_creation_input_tokens;
    public int cache_read_input_tokens;
  }
}
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `uuid` | Claude JSONL 消息唯一 id。 |
| `sessionId` | Claude session id。 |
| `parentUuid` | 上一条消息 id，用于恢复树/链。 |
| `timestamp` | 原生消息时间。 |
| `type` | Claude 原生事件类型，如 `user`/`assistant`/`summary` 等。 |
| `message.role` | 对话角色。 |
| `message.content` | 字符串或 content block 数组，包含 text、thinking、tool_use、tool_result。 |
| `message.usage` | token usage。 |
| `isMeta` | meta 消息，不应作为普通对话展示。 |
| `isSidechain` | sidechain/subagent 类消息标记。 |
| `cwd` | 工作目录。 |
| usage 字段 | 输入、输出、cache write、cache read token。 |

### Codex 原生历史

`CodexHistoryReader` 读取 `~/.codex/sessions`。基础 DTO：

```java
class CodexMessage {
  public String timestamp;
  public String type;
  public JsonObject payload;
}
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `timestamp` | Codex JSONL 事件时间。 |
| `type` | Codex 事件类型，例如 session metadata、item、turn、event message。 |
| `payload` | 原生事件负载，内部可能包含 `message/content/tool_call/usage/token_count/cwd` 等。插件按需解析。 |

### 插件 session 索引缓存

`SessionIndexManager` 写入 `~/.codemoss/cache/claude-session-index.json` 和 `codex-session-index.json`：

```java
class SessionIndex {
  public int version;
  public long lastUpdated;
  public Map<String, ProjectIndex> projects;
}

class ProjectIndex {
  public long lastDirScanTime;
  public int fileCount;
  public List<SessionIndexEntry> sessions;
}

class SessionIndexEntry {
  public String sessionId;
  public String title;
  public int messageCount;
  public long lastTimestamp;
  public long firstTimestamp;
  public long fileSize;
  public String cwd;
  public long fileLastModified;
  public String fileRelativePath;
}
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| `version` | 索引 schema 版本，当前源码为 `3`。版本不匹配会重建。 |
| `lastUpdated` | 索引文件最近保存时间。 |
| `projects` | project path 到 `ProjectIndex` 的映射。Codex 也按项目过滤/分组。 |
| `lastDirScanTime` | 目录上次扫描时间。 |
| `fileCount` | 扫描时文件数量，用于判断是否需要增量/全量更新。 |
| `sessions` | 该项目下 session 摘要。 |
| `sessionId` | provider session id。 |
| `title` | 会话标题，通常来自首条用户消息或 summary。 |
| `messageCount` | 消息数量。 |
| `lastTimestamp` | 最后一条消息时间。 |
| `firstTimestamp` | 第一条消息时间。 |
| `fileSize` | transcript 文件大小。 |
| `cwd` | Codex 专用工作目录字段。 |
| `fileLastModified` | transcript 文件 mtime，用于增量重读。 |
| `fileRelativePath` | 相对 provider 根目录的文件路径，用于 sessionId 与文件名不一致时定位。 |

### 设计评价

JetBrains CC GUI 的统一 message 便于 UI 渲染，但完整结构仍依赖 `raw`。它没有把 Claude/Codex 原生消息转换成一个长期稳定的自有数据库 schema，因此历史兼容性好，但跨 provider 的结构化查询能力较弱。
