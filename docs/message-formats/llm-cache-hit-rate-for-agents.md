# Agent 开发中的 LLM Cache 命中率设计

本文基于 `claude-code-sourcemap` 还原源码，分析 Claude Code 如何复用 LLM Prompt Cache，并提炼出 Agent 应用在设计消息目录、系统提示词、工具 schema、子 Agent、压缩与遥测时可以借鉴的做法。

结论先说清楚：提高 LLM Cache 命中率，不是给模型响应做本地缓存，而是让每次发给模型的请求前缀尽量稳定。只要 system prompt、tools、历史消息前缀、thinking 配置、header/body 参数中任意一项频繁变化，服务端 Prompt Cache 就会失效。

## 术语

| 术语 | 含义 | 对缓存的影响 |
| --- | --- | --- |
| Prompt Cache | 模型服务端对输入 prompt 前缀的缓存，如 Anthropic Prompt Caching | 命中后减少重复输入 token 计费和延迟 |
| Cache Key 输入 | 影响缓存命中的请求字段，如 model、system、tools、messages prefix、thinking、headers | 任意字段变化都可能导致 miss |
| Stable Prefix | 多轮请求中应保持字节稳定的前缀内容 | 应尽量进入 cache |
| Dynamic Tail | 每轮变化的内容，如当前用户输入、日期变化、工具结果增量 | 应放在请求尾部，避免污染 Stable Prefix |
| Cache Breakpoint | 使用 `cache_control` 标记的缓存切点 | 告诉模型服务端缓存到哪里 |
| Cache Bust | 本应命中的缓存因为内容、顺序、TTL、header 等变化失效 | 会导致大量 `cache_creation_input_tokens` |

## Claude Code sourcemap 的核心实现

### 1. 通过 `cache_control` 标记可缓存前缀

Claude Code 对 Anthropic Prompt Cache 的接入集中在：

- `codes/claude-code-sourcemap/restored-src/src/services/api/claude.ts`
- `codes/claude-code-sourcemap/restored-src/src/utils/api.ts`
- `codes/claude-code-sourcemap/restored-src/src/bootstrap/state.ts`

核心函数：

```ts
getPromptCachingEnabled(model)
getCacheControl({ scope, querySource })
should1hCacheTTL(querySource)
buildSystemPromptBlocks(...)
addCacheBreakpoints(...)
```

`getCacheControl` 生成的形态可以概括为：

```ts
type PromptCacheControl = {
  type: 'ephemeral';
  ttl?: '1h';
  scope?: 'global';
};
```

其中：

- 默认是短 TTL 的 ephemeral cache。
- 命中特定条件时使用 `ttl: '1h'`。
- 全局稳定内容可以使用 `scope: 'global'`。
- 是否启用缓存可通过环境变量禁用，例如 `DISABLE_PROMPT_CACHING`、`DISABLE_PROMPT_CACHING_SONNET`。

重点不在字段本身，而在它对请求格式的要求：被 `cache_control` 覆盖的前缀必须稳定。如果每轮都把动态内容塞到 system prompt 或 tool schema 里，加再多 `cache_control` 也没有意义。

### 2. system prompt 分块，静态和动态分离

Claude Code 不把 system prompt 当成一整段字符串直接缓存，而是拆成多个 system block。

普通模式大致分为：

| block | 内容 | 是否缓存 |
| --- | --- | --- |
| attribution header | 版权、来源或启动上下文 | 通常不缓存 |
| CLI system prompt prefix | CLI 固定系统提示词前缀 | 缓存 |
| rest system prompt | 剩余稳定系统提示词 | 缓存 |

global cache 模式下会更细：

| block | 内容 | 是否缓存 |
| --- | --- | --- |
| attribution | 动态归因或启动信息 | 不缓存 |
| CLI prefix | 可能随构建或模式变化 | 不缓存或 org cache |
| static content before boundary | 明确稳定的系统提示词 | global cache |
| dynamic content after boundary | 会话、工具、环境相关动态内容 | 不进入 global cache |

这个设计给 Agent 开发的启发是：system prompt 不能只有一个 `string` 字段，最好从一开始就设计成结构化 block。

推荐格式：

```ts
type SystemPromptBlock = {
  id: string;
  scope: 'product_static' | 'workspace_static' | 'session_dynamic' | 'turn_dynamic';
  text: string;
  cache?: {
    enabled: boolean;
    ttl?: '5m' | '1h';
    scope?: 'org' | 'global';
  };
};
```

设计约束：

- 产品固定提示词放在最前面，并长期保持字节不变。
- 工作区规则、项目规则、用户偏好可以放在次稳定层。
- 当前时间、当前文件、当前 git 状态、MCP 连接状态不要混入静态系统提示词。
- 如果必须提供动态上下文，放到 message tail 或 attachment delta。

### 3. 每次请求只设置一个消息级缓存切点

`addCacheBreakpoints` 的关键做法是：每次请求只放一个 message-level `cache_control`。

正常请求：

```text
messages[0 ... n-1] + cache_control on messages[n-1]
```

fire-and-forget fork 或不希望写入尾部缓存时：

```text
messages[0 ... n-2] + cache_control on messages[n-2]
messages[n-1]     + no cache write
```

这个细节很重要。很多 Agent 实现会在多个位置随手打 cache marker，最后导致缓存碎片多、尾部临时内容被写入、后续请求无法有效复用。

推荐原则：

- 一次模型请求只有一个主要缓存断点。
- 缓存断点尽量放在“历史稳定前缀”的最后一条消息。
- 当前用户输入、临时工具结果、fork 专属 prompt 不要轻易写入长期缓存。
- 如果有子 Agent 或后台任务，允许它读取父会话缓存，但避免它把短生命周期尾部写进主缓存。

### 4. tool schema 使用稳定主体和动态 overlay

Claude Code 的工具 schema 处理在 `toolToAPISchema` 中做了缓存：

- 基础 schema 按 tool name 或 tool name + structured output schema 缓存。
- 基础 schema 包含 `name`、`description`、`input_schema`、`strict` 等稳定字段。
- `defer_loading`、`cache_control` 这类每次请求可能不同的字段作为 overlay 追加。
- overlay 不反向污染基础 schema 缓存。

同时，`assembleToolPool` 会保持 built-in tools 和 MCP tools 的顺序稳定：

```text
sorted built-in tools + sorted allowed MCP tools
```

这样新增 MCP tool 不会插入 built-in tool 中间，避免整个 tool schema 前缀变化。

Agent 开发里的推荐格式：

```ts
type StableToolSchema = {
  name: string;
  description: string;
  inputSchema: unknown;
  outputSchema?: unknown;
  strict?: boolean;
};

type ToolRequestOverlay = {
  deferLoading?: boolean;
  cacheControl?: PromptCacheControl;
  enabled?: boolean;
};

type ToolPromptEntry = {
  stable: StableToolSchema;
  overlay?: ToolRequestOverlay;
};
```

设计约束：

- 工具描述不要包含动态列表，例如“当前可用 agent 有 A、B、C”。
- 工具描述不要包含随机路径、临时文件名、当前日期、剩余预算这类高频变化字段。
- MCP 工具排序必须稳定。不要直接使用对象遍历顺序。
- 动态工具可用性用 overlay 或 tail attachment 表达，不要改 stable schema。

### 5. 动态状态移出稳定前缀

Claude Code 对动态内容做了大量“尾部化”处理，避免 cache bust。

典型案例：

| 动态内容 | 容易犯的错误 | Claude Code 的处理 | 可借鉴点 |
| --- | --- | --- | --- |
| 日期跨天 | 修改第一条 user context 里的日期 | 追加 date change attachment | 日期变化放尾部 delta |
| Agent 列表 | 写进 AgentTool description | 生成 agent listing delta attachment | 动态列表不要进入 tool schema |
| MCP instructions | 每次追加 system prompt | 改为 message delta 或专门 attachment | 外部连接状态放尾部 |
| deferred tools | 每次 prepend 到消息前面 | 持久化为 deferred_tools_delta attachment | 不要 prepend 动态内容 |
| settings 路径 | 用随机 UUID 临时文件 | 用内容 hash 生成稳定路径 | 请求中出现的路径也要稳定 |
| memory 日期 | “3 days ago” 每天变化 | 预计算或改为稳定表达 | 避免相对时间污染前缀 |

这部分是最值得 Agent 系统学习的地方：缓存命中率不是 API 层一个开关能解决的，而是整个上下文组织方式都要围绕“稳定前缀、动态尾部”来设计。

推荐消息目录中的上下文分层：

```ts
type AgentContextForLLM = {
  stablePrefix: {
    system: SystemPromptBlock[];
    tools: ToolPromptEntry[];
    longTermMemorySummary?: string;
    projectRules?: string;
  };
  reusableMessagePrefix: PersistedMessage[];
  dynamicTail: {
    currentUserMessage: PersistedMessage;
    dateDelta?: AttachmentBlock;
    agentDelta?: AttachmentBlock;
    mcpDelta?: AttachmentBlock;
    permissionDelta?: AttachmentBlock;
    toolResultDelta?: AttachmentBlock[];
  };
};
```

### 6. forked agent 复用父会话前缀

`forkedAgent.ts` 中的 `CacheSafeParams` 说明了 Anthropic cache key 的核心输入：

- system prompt
- tools
- model
- messages prefix
- thinking config

forked agent 会构造：

```ts
initialMessages = [...forkContextMessages, ...promptMessages]
```

也就是：

1. 先复用父会话已经缓存过的消息前缀。
2. 再追加 fork 自己的 prompt。
3. 通过 `skipCacheWrite` 避免 fork 临时尾部污染缓存。

这对多 Agent 系统很关键。很多系统启动子 Agent 时，会重新拼一份 system prompt、重新整理工具、重新摘要上下文，结果看起来语义相同，字节却完全不同，Prompt Cache 命中率很差。

推荐子 Agent 调用格式：

```ts
type SubAgentInvocation = {
  parentCacheRef: {
    sessionId: string;
    prefixMessageIds: string[];
    systemPromptVersion: string;
    toolSchemaVersion: string;
    model: string;
    thinkingConfigHash: string;
  };
  promptTail: PersistedMessage[];
  cachePolicy: {
    readParentPrefix: true;
    writeTailCache: false;
  };
};
```

设计约束：

- 子 Agent 尽量继承父 Agent 的 stable prefix。
- 子 Agent 特有指令放到 tail，不要重新生成完整 system prompt。
- 子 Agent 如果只是一次性分析，默认不要写入新的缓存尾巴。
- 多 Agent 并发时，确保 tool schema 顺序、system block 顺序一致。

### 7. compact 与 cache editing

Claude Code 有两类和缓存相关的 compact 思路。

第一类是普通 microcompact：

- 当长时间间隔后缓存可能过期，旧 tool result 继续原样保留会导致下一次请求重写成本很高。
- microcompact 会清理或替换一些旧 tool result，减少后续 cache write 体积。

第二类是 cached microcompact：

- 在 API 层给已缓存前缀里的 `tool_result` 加 `cache_reference`。
- 后续通过 `cache_edits` 删除或替换旧缓存中的部分内容。
- 本地消息不直接修改，而是在 API 请求层插入 `cache_reference` 和 `cache_edits`。

可抽象为：

```ts
type CacheEditBlock = {
  type: 'cache_edits';
  edits: Array<{
    type: 'delete';
    cache_reference: string;
  }>;
};

type CacheReferenceBlock = {
  type: 'tool_result';
  tool_use_id: string;
  content: unknown;
  cache_reference?: string;
};
```

对 Agent 开发的启发：

- compact 不应该只考虑“上下文长度”，还要考虑“缓存重写成本”。
- 对大工具结果、搜索结果、文件内容，应该设计可引用、可删除、可替换的块。
- 本地 transcript 可以保持原始事实，API 请求层再做 cache edit overlay。
- compact 后要告诉 cache break detector 哪些 cache read 下降是预期行为。

### 8. header/body 参数也要稳定

很多系统只关注 prompt 文本，却忽略 header 和 body 参数。Claude Code 会把一些特性开关做成 session-stable latch，例如：

- 1h prompt cache allowlist
- 1h prompt cache eligibility
- AFK/autonomous mode header
- fast mode header
- cache editing beta header
- thinking clear 状态

原因很直接：如果中途打开某个 beta header 或改 body 参数，服务端可能认为 cache key 变了，前面 50K 到 70K token 的缓存就不能复用。

推荐请求参数格式：

```ts
type LLMRequestCacheKeyParts = {
  provider: string;
  model: string;
  systemHash: string;
  toolSchemaHash: string;
  messagePrefixHash: string;
  thinkingConfigHash?: string;
  betaHeadersHash?: string;
  extraBodyParamsHash?: string;
};
```

工程约束：

- 同一 session 内影响 cache key 的开关尽量 latch。
- 如果必须切换，需要明确认为这是一次 cache boundary。
- 实验开关不要在每轮请求随机变化。
- body 参数默认值要显式化，避免 undefined 和默认值在不同调用路径里交替出现。

### 9. 建立 cache break detection 和 usage 遥测

Claude Code 不只是做缓存，还会检测缓存是否被打破。

它会记录请求前的可观测 cache key：

- system blocks，包括 `cache_control`
- system blocks 去掉 `cache_control` 后的内容
- tool schemas
- model
- fast mode
- global cache strategy
- beta headers
- overage
- cached microcompact
- thinking effort
- extra body params

响应回来后，它会比较：

```ts
cache_read_input_tokens
cache_creation_input_tokens
cache_creation.ephemeral_1h_input_tokens
cache_creation.ephemeral_5m_input_tokens
```

如果 cache read 明显下降，就尝试判断是 system、tools、headers、TTL、body params 还是 cache edit 导致。

Agent 产品也应该把缓存遥测设计成一等能力。

推荐指标：

```ts
type PromptCacheMetrics = {
  requestId: string;
  sessionId: string;
  model: string;
  inputTokens: number;
  outputTokens: number;
  cacheReadInputTokens: number;
  cacheCreationInputTokens: number;
  cacheCreation5mInputTokens?: number;
  cacheCreation1hInputTokens?: number;
  cacheHitRatio: number;
  cacheKeyParts: LLMRequestCacheKeyParts;
  suspectedBreakReason?: string;
};
```

推荐报警规则：

| 现象 | 可能原因 | 排查方向 |
| --- | --- | --- |
| cache read 从高位突然降为 0 | system 或 tool schema 变了 | hash system/tool |
| 每天 0 点后 cache creation 飙升 | 日期写在稳定前缀 | 查 user context 和 memory |
| 安装 MCP 后缓存大面积失效 | tool 顺序或 tool description 变化 | 查 tool schema diff |
| 子 Agent 请求不命中 | 没复用父 messages prefix | 查 fork 参数 |
| 同一功能 A/B 用户命中率不同 | beta header 或实验参数不稳定 | 查 header/body latch |

## Agent 开发可借鉴的设计原则

### 原则一：缓存优化从上下文模型开始，不从 API 封装开始

不推荐：

```ts
callLLM({
  system: buildSystemPromptEverything(),
  tools: buildToolsEverything(),
  messages: buildMessagesEverything(),
});
```

推荐：

```ts
callLLM({
  stablePrefix: buildStablePrefix(),
  reusableMessages: loadReusableMessagePrefix(),
  dynamicTail: buildDynamicTail(),
  cachePolicy: buildCachePolicy(),
});
```

也就是说，业务层就要知道哪些内容稳定、哪些内容动态，不能到 API 层才尝试“猜测”。

### 原则二：消息目录要保存事实，LLM 请求可以有派生视图

入库消息应该尽量保存真实发生过的事实，例如用户输入、assistant 输出、工具调用、工具结果。  
但发给 LLM 的请求可以是一个派生视图：

- 可以跳过 progress。
- 可以压缩旧 tool result。
- 可以追加 date delta。
- 可以把动态 agent list 放到尾部。
- 可以对缓存前缀添加 `cache_control`。

因此建议区分：

```ts
type TranscriptMessage = PersistedMessage;
type LLMRequestMessage = ProviderSpecificMessage & {
  cache_control?: PromptCacheControl;
  cache_reference?: string;
};
```

不要把 provider-specific 的 cache 字段直接污染业务主表。

### 原则三：稳定内容必须有版本和 hash

要提高命中率，需要能回答“为什么 miss”。这要求 stable prefix 可观测。

推荐记录：

```ts
type StablePrefixManifest = {
  systemPromptVersion: string;
  systemPromptHash: string;
  toolSchemaVersion: string;
  toolSchemaHash: string;
  memorySummaryHash?: string;
  projectRulesHash?: string;
  betaHeadersHash?: string;
  createdAt: string;
};
```

版本用于人为判断，hash 用于自动 diff。

### 原则四：动态信息只追加，不前插

前插动态内容是缓存命中率杀手。

不推荐：

```text
[new date context]
[system prompt]
[tools]
[history messages]
[current user]
```

推荐：

```text
[system prompt]
[tools]
[history messages]
[current user]
[date delta]
```

原因是 Prompt Cache 通常按前缀复用。越靠前变化，影响越大。

### 原则五：工具描述要像 API contract，不要像运行时状态面板

工具 schema 应描述能力边界，而不是当前状态。

不推荐：

```text
AgentTool description:
当前可用 agents: frontend, backend, reviewer
当前权限模式: plan
当前 MCP servers: github, linear
```

推荐：

```text
AgentTool description:
Launch or delegate work to an available agent.

Runtime attachment:
Available agents changed:
- frontend
- backend
- reviewer
```

### 原则六：子 Agent 默认读父缓存，不默认写尾部缓存

子 Agent 常见任务是搜索、审查、总结、探索。它们通常需要父上下文，但它们的尾部 prompt 生命周期很短。

推荐默认策略：

```ts
{
  readParentCache: true,
  writeSubAgentTailCache: false,
  persistSubAgentTranscript: true
}
```

这样既能复用父上下文，又不会污染主会话缓存。

### 原则七：compact 要同时优化上下文长度和缓存成本

如果只按 token 数 compact，可能会出现：

- 上下文变短了，但 stable prefix 每次都变，缓存命中率下降。
- compact 摘要每次生成时间戳或随机标题，导致缓存前缀变化。
- 大 tool result 留在 cache prefix 中，过期后重写成本很高。

推荐 compact 策略：

| 内容 | 推荐处理 |
| --- | --- |
| 旧文本对话 | 摘要为稳定 summary block |
| 大文件内容 | 替换为 file reference + excerpt hash |
| 工具结果 | 支持 cache reference 或内容替换 entry |
| progress/status | 不进入 compact 主链 |
| permission event | 只保留最终决策和必要理由 |

## 推荐的端到端请求格式

下面是一套适合 Agent 应用的 LLM 请求组装格式。

```ts
type AgentLLMRequest = {
  requestId: string;
  sessionId: string;
  turnId: string;

  cacheKey: LLMRequestCacheKeyParts;

  model: string;
  thinking?: {
    enabled: boolean;
    budgetTokens?: number;
    effort?: 'low' | 'medium' | 'high';
  };

  system: Array<SystemPromptBlock & {
    providerCacheControl?: PromptCacheControl;
  }>;

  tools: ToolPromptEntry[];

  messages: Array<{
    id: string;
    role: 'user' | 'assistant' | 'tool';
    content: MessageBlock[];
    providerCacheControl?: PromptCacheControl;
    providerCacheReference?: string;
  }>;

  cachePolicy: {
    enabled: boolean;
    breakpoint: 'system' | 'last_reusable_message' | 'none';
    ttl?: '5m' | '1h';
    scope?: 'org' | 'global';
    skipTailWrite?: boolean;
  };

  overlays?: {
    dynamicAttachments?: AttachmentBlock[];
    cacheEdits?: CacheEditBlock[];
  };
};
```

组装顺序建议：

1. 读取 transcript 或 DB 中的事实消息。
2. 过滤 progress、UI status、无关 telemetry。
3. 选择可复用历史前缀。
4. 构造稳定 system blocks。
5. 构造稳定 tool schema，并做 deterministic sort。
6. 把日期、agent list、MCP 状态、权限变化放入 dynamic attachments。
7. 在最后一条可复用 message 上加 cache breakpoint。
8. 发送请求前记录 cache key manifest。
9. 响应后记录 cache read/write metrics。
10. 如果 cache read 异常下降，保存本次和上次 cache key diff。

## 反模式清单

| 反模式 | 为什么影响命中率 | 修正方式 |
| --- | --- | --- |
| system prompt 每轮拼当前时间 | 第一段 prompt 每轮不同 | 时间放 tail attachment |
| tool description 包含动态 agent 列表 | tool schema hash 每轮变 | agent list 放 delta |
| MCP tools 使用非稳定排序 | 工具顺序变化导致 schema 前缀变 | 按 name 和 namespace 排序 |
| 临时文件路径使用随机 UUID | 路径进入 tool prompt 后每轮不同 | 用内容 hash 或稳定 session path |
| 子 Agent 重新生成完整上下文 | 语义相同但字节不同 | 复用父 messages prefix |
| 多处添加 cache_control | cache entry 碎片化 | 统一一个主要 breakpoint |
| 动态内容 prepend 到 messages | 前缀整体变化 | 动态内容 append 到 tail |
| 实验 header 每轮实时变化 | cache key 变化 | session 内 latch |
| compact summary 带相对时间 | 每天或每轮变化 | 使用绝对时间或稳定摘要 |
| progress 入主消息链 | 噪音增加，恢复和缓存都变复杂 | progress 进入 event log，不进入 prompt prefix |

## 落地检查清单

### 架构层

- 是否区分 `TranscriptMessage` 和 `LLMRequestMessage`。
- 是否有 stable prefix 与 dynamic tail 的显式模型。
- 是否有 system prompt block，而不是单个大字符串。
- 是否有 tool schema cache 或 schema version/hash。
- 是否有 cache key manifest。

### 请求组装层

- system blocks 顺序是否 deterministic。
- tools 顺序是否 deterministic。
- 当前日期、agent list、MCP 状态是否没有进入 system/tool stable prefix。
- 是否只在一个主要位置添加 cache breakpoint。
- 子 Agent 是否复用父 messages prefix。
- fork 或后台任务是否避免写入临时 tail cache。

### 持久层

- transcript 是否保存事实，而不是 provider cache 字段。
- progress/status 是否不进入主消息链。
- tool result 是否可以被 compact 或 reference。
- compact boundary 是否可恢复。
- metadata entry 是否和 message entry 区分。

### 遥测层

- 是否记录 `cache_read_input_tokens`。
- 是否记录 `cache_creation_input_tokens`。
- 是否计算 cache hit ratio。
- 是否能 diff system/tool/header/body/message prefix 的 hash。
- 是否能识别预期 cache drop，例如 compact 或 cache edit。

## 建议的实现优先级

如果从零开始优化 Agent LLM Cache 命中率，建议按这个顺序做：

1. 先让 system prompt 结构化分块，拆出静态和动态。
2. 固定 tool schema 顺序，并禁止动态状态进入 tool description。
3. 为 LLM 请求增加 stable prefix 和 dynamic tail 两层。
4. 在最后一条可复用消息上统一添加 cache breakpoint。
5. 子 Agent 调用复用父前缀，并默认 `skipTailCacheWrite`。
6. 增加 cache read/write usage 统计。
7. 增加 cache key manifest 和 cache break detection。
8. 最后再做 cached compact、cache edit 这类高级优化。

## 总结

Claude Code sourcemap 展示的核心经验可以压缩成一句话：把“会变的东西”从“可缓存前缀”里拿出去。

对 Agent 应用来说，这意味着：

- 消息目录保存事实。
- LLM 请求是事实消息的缓存友好派生视图。
- system prompt 和 tools 要可分块、可排序、可 hash。
- 当前状态、环境变化和 UI 事件走 dynamic tail。
- 子 Agent、compact、MCP、权限系统都要尊重 stable prefix。
- 没有 cache telemetry，就无法持续优化命中率。

这套设计会让 Agent 系统在长会话、多工具、多子 Agent 场景下更稳定地复用 LLM Prompt Cache，减少重复输入 token 成本，也能降低长上下文请求的首 token 延迟。
