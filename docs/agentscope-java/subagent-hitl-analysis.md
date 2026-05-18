# Sub Agent 与 Human-in-the-Loop：嵌套挂起问题分析

## 问题描述

当 Sub Agent（作为工具调用的子 Agent）内部触发 HITL 机制时，挂起信号会丢失。这是因为 `SubAgentTool` 的 `callAsync()` 对父 Agent 而言是一个**同步阻塞的工具调用**——父 Agent 期待它返回最终结果，而不是挂起信号。

## 当前 SubAgentTool 的实现

### 调用链路

```
父 Agent LLM 返回 tool_use → 父 Agent.acting() → Toolkit 执行 SubAgentTool.callAsync()
    → SubAgentTool 创建/加载子 Agent → subAgent.call(userMsg)
    → 子 Agent 完整的 ReAct 循环（reasoning → acting → reasoning → ...）
    → 返回最终 Msg → SubAgentTool.buildResult() 提取文本
    → ToolResultBlock.text("session_id: xxx\n\n回答内容...")
    → 父 Agent 继续推理
```

### 关键代码：buildResult() — HITL 信号的"坟墓"

```java
// SubAgentTool.java:387-395
private ToolResultBlock buildResult(Msg response, String sessionId) {
    String textContent = response.getTextContent();

    return ToolResultBlock.text(
        String.format(
            "session_id: %s\n\n%s",
            sessionId, textContent != null ? textContent : "(No response)"));
}
```

当子 Agent 返回 `GenerateReason.TOOL_SUSPENDED` 时，`response` 中包含：
- `ToolUseBlock`（子 Agent 待执行的外部工具，如 `ask_user`）
- `ToolResultBlock`（带 `isSuspended() == true` 标记）
- `GenerateReason.TOOL_SUSPENDED`

但 `buildResult()` **只提取了 `getTextContent()`**，以上所有信息全部丢弃。子 Agent 的挂起状态对父 Agent 和最终调用方完全不可见。

## 问题场景

```
调用方 → 父 Agent → [tool_use: call_researchagent]
                       → SubAgentTool → 子 Agent (ReActAgent)
                           → LLM 推理 → [tool_use: ask_user]
                               → UserInteractionTool 抛出 ToolSuspendException
                               → 子 Agent.acting() → buildSuspendedMsg()
                               → 返回 TOOL_SUSPENDED（含 ask_user 的 ToolUseBlock）
                       → SubAgentTool.buildResult() ← 问题在这里！
                       → 只提取了 textContent = "[Awaiting external execution]"
                       → 返回 ToolResultBlock.text("session_id: abc\n\n[Awaiting...]")
                   → 父 Agent 看到："子 Agent 完成了，返回了文本"
                   → 父 Agent 继续推理，可能输出错误结论
```

## 为什么当前设计如此

读 SubAgentTool 的完整实现可以发现，**它并非没有考虑多轮对话**——它通过 `session_id` 支持跨调用的会话保持：

```java
// SubAgentTool.java:131-204
private Mono<ToolResultBlock> executeConversation(ToolCallParam param) {
    // 获取或创建 session_id
    String sessionId = (String) input.get(PARAM_SESSION_ID);
    boolean isNewSession = sessionId == null;

    // 创建 Agent 实例，加载已有状态
    Agent agent = agentProvider.provide();
    if (!isNewSession && agent instanceof StateModule) {
        loadAgentState(finalSessionId, (StateModule) agent);
    }

    // 执行
    Msg userMsg = Msg.builder()...build();
    result = executeWithoutStreaming(agent, userMsg, ...);

    // 保存状态
    return result.doOnSuccess(r -> saveAgentState(...));
}
```

设计意图是 LLM 可以多轮与子 Agent 对话：第一次 `call_researchagent(message="...")`，得到回答后，第二次 `call_researchagent(session_id="abc", message="继续...")`。

但问题在于：**多轮对话的驱动者是父 Agent 的 LLM，而不是外部调用方**。当子 Agent 挂起等待用户输入时，需要外部调用方介入，但父 LLM 并不知道这件事。

## 解决方案设计

### 核心思路

**让 SubAgentTool 在检测到子 Agent HITL 状态时，自身抛出 `ToolSuspendException`，将挂起信号传播到父 Agent 层。**

```
子 Agent HITL → SubAgentTool 检测 → 保存子 Agent 状态 → 抛出 ToolSuspendException
    → 父 Agent 捕获 → ToolResultBlock.suspended()
    → 父 Agent.acting() → buildSuspendedMsg() → TOOL_SUSPENDED
    → 最终调用方收到挂起消息（含子 Agent 的待处理工具信息）
```

### 修改点 1：SubAgentTool 检测 HITL 状态

```java
// SubAgentTool.callAsync() 中，执行后检测
private Mono<ToolResultBlock> executeConversation(ToolCallParam param) {
    // ... 现有逻辑 ...

    Mono<ToolResultBlock> result = executeWithoutStreaming(...);

    return result.flatMap(response -> {
        GenerateReason reason = response.getGenerateReason();

        // 检测子 Agent 的 HITL 状态
        if (reason == GenerateReason.TOOL_SUSPENDED
                || reason == GenerateReason.REASONING_STOP_REQUESTED
                || reason == GenerateReason.ACTING_STOP_REQUESTED) {

            // 1. 立即保存子 Agent 状态（不等 doOnSuccess，因为还没成功）
            if (agent instanceof StateModule) {
                saveAgentState(finalSessionId, (StateModule) agent);
            }

            // 2. 提取子 Agent 的挂起信息
            List<ToolUseBlock> pendingTools =
                response.getContentBlocks(ToolUseBlock.class);

            // 3. 构造包含子 Agent 上下文的原因信息
            String reason_text = buildSuspensionReason(
                agent.getName(), pendingTools, response.getTextContent());

            // 4. 向上抛出 ToolSuspendException
            throw new ToolSuspendException(reason_text);
        }

        return buildResult(response, sessionId);
    }).doOnSuccess(r -> {
        if (agent instanceof StateModule) {
            saveAgentState(finalSessionId, (StateModule) agent);
        }
    });
}
```

### 修改点 2：ToolSuspendException 携带子 Agent 上下文

当前 `ToolSuspendException` 只携带一个 `reason` 字符串。为了让最终调用方能理解嵌套挂起的上下文，需要更丰富的结构：

```java
public class ToolSuspendException extends RuntimeException {
    private final String reason;
    // 新增：子 Agent 的挂起上下文
    private final List<ToolUseBlock> nestedPendingTools;  // 子 Agent 等待执行的工具
    private final String nestedAgentName;                  // 子 Agent 名称
    private final String nestedSessionId;                  // 子 Agent 的 session_id（用于恢复）

    // 或更通用的方式：结构化 metadata
    private final Map<String, Object> suspensionContext;
}
```

对应的 `ToolResultBlock.suspended()` 也需要将这些上下文信息序列化到 metadata 中：

```java
public static ToolResultBlock suspended(ToolUseBlock toolUse, ToolSuspendException exception) {
    Map<String, Object> metadata = new HashMap<>();
    metadata.put(METADATA_SUSPENDED, true);

    // 如果存在嵌套挂起上下文，一并携带
    if (exception.getSuspensionContext() != null) {
        metadata.put("nested_suspension", exception.getSuspensionContext());
    }

    return new ToolResultBlock(
        toolUse.getId(), toolUse.getName(),
        List.of(TextBlock.builder().text(reason).build()),
        metadata);
}
```

### 修改点 3：恢复流程

恢复流程涉及**两级恢复**：

```
调用方提供用户输入
    → 包装为父 Agent 的 ToolResultBlock（针对 SubAgentTool 的 tool_use ID）
    → 父 Agent.doCall() → validateAndAddToolResults() → acting()
    → executeToolCalls() → SubAgentTool.callAsync() 再次被调用
    → SubAgentTool 检测到是恢复调用（input 含 user response）
    → 加载子 Agent 状态 → subAgent.call(toolResults) 恢复执行
    → 子 Agent 继续推理 → 返回最终结果
```

SubAgentTool 的恢复检测逻辑：

```java
private Mono<ToolResultBlock> executeConversation(ToolCallParam param) {
    Map<String, Object> input = param.getInput();
    String sessionId = (String) input.get(PARAM_SESSION_ID);
    String message = (String) input.get(PARAM_MESSAGE);

    Agent agent = agentProvider.provide();

    // 加载已有状态
    loadAgentState(sessionId, (StateModule) agent);

    // ★ 关键：检测是否是恢复调用
    if (hasPendingToolUse(agent)) {
        // 这是恢复场景 — message 中包含的是用户对子 Agent 挂起工具的响应
        // 构造子 Agent 的 ToolResultBlock
        ToolUseBlock pendingTool = getFirstPendingTool(agent);
        ToolResultBlock userResult = ToolResultBlock.of(
            pendingTool.getId(),
            pendingTool.getName(),
            TextBlock.builder().text(message).build());

        Msg toolResultMsg = Msg.builder()
            .role(MsgRole.TOOL)
            .content(userResult)
            .build();

        // 用 tool result 恢复子 Agent
        return ((ReActAgent) agent).call(List.of(toolResultMsg))
            .map(response -> buildResult(response, sessionId))
            .doOnSuccess(r -> saveAgentState(sessionId, (StateModule) agent));
    }

    // 正常首次调用
    Msg userMsg = Msg.builder()...build();
    return executeWithoutStreaming(agent, userMsg, ...)
        .flatMap(response -> {
            if (isHitlState(response.getGenerateReason())) {
                saveAgentState(sessionId, ...);
                throw new ToolSuspendException(...);
            }
            return Mono.just(buildResult(response, sessionId));
        });
}
```

## 调用方视角：完整交互流程

### 场景：父 Agent 调用子 Agent，子 Agent 需要询问用户

```
1. 用户: "帮我分析这份数据"
2. 父 Agent LLM: tool_use call_analyst(message="分析数据")
3. 子 Agent LLM: tool_use ask_user(question="请确认分析维度", ui_type="select", options=["销售", "用户", "财务"])
4. 子 Agent: UserInteractionTool 抛出 ToolSuspendException
5. 子 Agent 返回 TOOL_SUSPENDED
6. SubAgentTool 检测 → 保存子 Agent 状态(session_id="s1")
   → 抛出 ToolSuspendException(reason="子Agent等待用户选择分析维度")
7. 父 Agent 捕获 → ToolResultBlock.suspended(call_analyst_toolUse, exception)
8. 父 Agent acting() → buildSuspendedMsg() → 返回 TOOL_SUSPENDED
9. 调用方收到挂起消息:
   {
     "generateReason": "TOOL_SUSPENDED",
     "content": [
       { "type": "tool_use", "name": "call_analyst", "id": "tu_1", "input": {...} },
       { "type": "tool_result", "id": "tu_1", "metadata": {
           "agentscope_suspended": true,
           "nested_suspension": {
             "agent_name": "AnalystAgent",
             "session_id": "s1",
             "pending_tools": [
               { "name": "ask_user", "input": {"question": "请确认分析维度", "ui_type": "select", "options": [...]} }
             ]
           }
         }
       }
     ]
   }
10. UI 渲染子 Agent 的 ask_user 交互组件
11. 用户选择: "销售"
12. 调用方构造恢复消息:
    ToolResultBlock resumeResult = ToolResultBlock.builder()
        .id("tu_1")   // SubAgentTool 的 tool_use ID
        .name("call_analyst")
        .output(TextBlock.from("销售"))
        .metadata(Map.of("subagent_session_id", "s1"))
        .build();
    parentAgent.call(Msg.of(resumeResult)).block();
13. 父 Agent doCall() → acting() → SubAgentTool.callAsync() 再次执行
14. SubAgentTool 加载 session "s1" → 检测到子 Agent 有 pending tool
    → 构造 ToolResultBlock → subAgent.call(toolResultMsg)
15. 子 Agent 收到 ask_user 的结果("销售")
    → 继续推理 → 完成分析 → 返回最终结果
16. SubAgentTool.buildResult() → 分析报告文本
17. 父 Agent 看到最终结果 → 继续推理 → 返回给用户
```

## 更简洁的方案：SubAgentTool 内部消化 HITL（不传播到父 Agent）

如果不想让父 Agent LLM 感知子 Agent 的 HITL 状态（看作实现细节），可以换一种思路：

**SubAgentTool 在内部完成整个 HITL 循环，通过 ToolEmitter 把交互事件转发出去，最终只把子 Agent 的最终结果返回给父 Agent。**

```
子 Agent HITL → SubAgentTool 不抛异常
    → 通过 ToolEmitter 发送 "subagent_needs_input" 事件给调用方
    → SubAgentTool 内部 block 等待用户输入（通过某种 Future/Channel 机制）
    → 调用方通过另一个 API 提交用户输入
    → Channel 收到输入 → SubAgentTool 继续执行子 Agent
    → 子 Agent 最终完成 → SubAgentTool 返回最终结果
```

这种方案下，父 Agent LLM **完全不知道子 Agent 发生过 HITL**，工具调用看上去只是一个"比较慢"的同步调用而已。

但这要求 SubAgentTool 的 `callAsync()` **阻塞等待外部输入**，而当前架构中 `callAsync()` 返回的是 `Mono<ToolResultBlock>`（响应式），这可以通过 `Mono.create()` 配合一个外部可完成的 Sink 来实现。

## 对比总结

| 维度 | 方案 A: 向上传播挂起 | 方案 B: 内部消化 |
|------|---------------------|-----------------|
| **父 Agent 感知** | 父 Agent 知道工具挂起了 | 父 Agent 无感知 |
| **父 LLM 行为** | 可能尝试重新调用或其他策略 | 无影响 |
| **调用方复杂度** | 需要解析嵌套挂起上下文 | 需要支持事件监听 + 输入注入 API |
| **架构侵入性** | 修改 ToolSuspendException + SubAgentTool | 需要新的输入注入通道 |
| **适用场景** | 父 Agent 需要根据挂起做决策 | 子 Agent 的交互是纯 UI 层的 |

两种方案可以共存——方案 A 作为默认行为，方案 B 通过 `SubAgentConfig` 配置项开启。

## 当前代码中的"种子"

SubAgentTool 已有的设计实际上为方案 B 做了一些准备：

1. **`config.isForwardEvents()`** — 开启后，子 Agent 的 streaming events 会通过 `ToolEmitter` 转发，调用方可以监听子 Agent 的实时状态
2. **`SubagentEventBus`** — 允许子 Agent 事件注入父 Agent 的 event stream
3. **Session 持久化** — 子 Agent 状态可以在挂起期间持久化

方案 A 需要的改动更少（只需修改 `SubAgentTool` 和增强 `ToolSuspendException`），方案 B 虽然架构更干净但需要更多基础设施（输入注入通道、超时处理等）。

---

## 基于 AgentTool 接口自定义 Sub Agent

### AgentTool 接口的能力边界

`SubAgentTool` 只是 `AgentTool` 的一种实现。框架内还有 `McpTool`、`SchemaOnlyTool`、`ShellCommandTool` 等。直接实现 `AgentTool` 接口自定义 sub agent 时，开发者拥有更多控制权，但需要自行处理更多细节。

```java
public interface AgentTool {
    String getName();
    String getDescription();
    Map<String, Object> getParameters();
    Mono<ToolResultBlock> callAsync(ToolCallParam param);  // ← 核心方法
}
```

### ToolCallParam 提供的上下文

`callAsync(ToolCallParam param)` 的 `param` 携带了父 Agent 的完整上下文：

| 字段 | 类型 | 说明 |
|------|------|------|
| `getToolUseBlock()` | `ToolUseBlock` | 当前工具调用的 ID 和输入参数 |
| `getInput()` | `Map<String, Object>` | 已解析的工具输入参数 |
| `getAgent()` | `Agent` | **父 Agent 实例**（可访问其 memory、toolkit 等） |
| `getEmitter()` | `ToolEmitter` | 流式输出发射器（中间结果会触发 ActingChunk Hook） |
| `getContext()` | `ToolExecutionContext` | 自定义上下文对象（通过 `register()` 注入的 POJO） |

### SubAgentTool 帮你做了什么 vs 你需要自己做什么

| 能力 | SubAgentTool 已实现 | 自定义 AgentTool 需自行处理 |
|------|---------------------|---------------------------|
| 子 Agent 实例创建 | `SubAgentProvider.provide()` 每次创建新实例 | 自行管理 Agent 生命周期 |
| Session 持久化 | `session_id` → `saveTo/loadFrom` | 自行实现或复用 |
| 多轮对话 | 通过 `session_id` 参数支持 | 需自行设计 |
| 事件转发 | `config.isForwardEvents()` → ToolEmitter | 可自行调用 `param.getEmitter().emit()` |
| Schema 生成 | 自动生成 `{session_id, message}` 参数 | 自行在 `getParameters()` 中定义 |
| **HITL 检测** | **❌ 未实现** | **✅ 可以自定义** |
| **挂起信号传播** | **❌ 丢失** | **✅ 可直接 throw** |
| **恢复机制** | **❌ 未实现** | **✅ 可自行设计** |

### 自定义 AgentTool 实现 HITL 的三种模式

#### 模式 1：直接抛出 ToolSuspendException（最简单）

```java
public class HitlAwareSubAgent implements AgentTool {
    private final SubAgentProvider<ReActAgent> provider;

    @Override
    public Mono<ToolResultBlock> callAsync(ToolCallParam param) {
        ReActAgent subAgent = provider.provide();
        String message = (String) param.getInput().get("message");

        Msg userMsg = Msg.builder()
            .role(MsgRole.USER)
            .content(TextBlock.builder().text(message).build())
            .build();

        return subAgent.call(List.of(userMsg))
            .flatMap(response -> {
                // ★ 关键：检测子 Agent 的 HITL 状态
                if (response.getGenerateReason() == GenerateReason.TOOL_SUSPENDED
                        || response.getGenerateReason() == GenerateReason.REASONING_STOP_REQUESTED) {

                    // 提取子 Agent 的待处理工具信息
                    String pendingInfo = extractPendingToolInfo(response);
                    // 直接抛出 → ToolExecutor.onErrorResume 会转为 ToolResultBlock.suspended()
                    throw new ToolSuspendException(
                        "[" + subAgent.getName() + "] " + pendingInfo);
                }

                // 正常完成 — 返回最终结果
                return Mono.just(
                    ToolResultBlock.text(subAgent.getName() + ": " + response.getTextContent()));
            });
    }
}
```

**优点**：只需在 `callAsync()` 中加判断 + `throw`。`ToolExecutor` 的 `onErrorResume(ToolSuspendException.class, ...)` 已经处理了异常转换。
**缺点**：不具备恢复能力——子 Agent 状态未保存，恢复时无法接续。

#### 模式 2：挂起 + Session 持久化恢复（完整方案）

```java
public class HitlAwareSubAgent implements AgentTool {
    private final SubAgentProvider<ReActAgent> provider;
    private final Session session;  // 持久化 session

    @Override
    public Mono<ToolResultBlock> callAsync(ToolCallParam param) {
        ReActAgent subAgent = provider.provide();
        String message = (String) param.getInput().get("message");

        // ★ 从 metadata 提取 session 上下文（恢复场景）
        // ToolCallParam 的 input 是工具参数，但恢复信息需要通过其他通道传递
        // ...

        Msg userMsg = Msg.builder()
            .role(MsgRole.USER)
            .content(TextBlock.builder().text(message).build())
            .build();

        return subAgent.call(List.of(userMsg))
            .flatMap(response -> {
                if (isHitlState(response)) {
                    // 保存子 Agent 状态以便恢复
                    String sessionId = UUID.randomUUID().toString();
                    subAgent.saveTo(session, sessionId);

                    // 将 sessionId 嵌入异常消息（通过 reason 或扩展字段）
                    String pendingInfo = extractPendingToolInfo(response);
                    throw new ToolSuspendException(
                        "[sub_session:" + sessionId + "] " + pendingInfo);
                }
                return Mono.just(ToolResultBlock.text(response.getTextContent()));
            });
    }
}
```

当工具层面的 `ToolSuspendException` 被捕获并转为 `ToolResultBlock.suspended()` 后，最终调用方收到的 `TOOL_SUSPENDED` 消息中就包含了 `session_id`，恢复时可以通过约定好的方式注入回去。

#### 模式 3：利用 ToolEmitter 转发子 Agent 状态（流式 HITL 感知）

```java
@Override
public Mono<ToolResultBlock> callAsync(ToolCallParam param) {
    ReActAgent subAgent = provider.provide();
    String message = (String) param.getInput().get("message");
    ToolEmitter emitter = param.getEmitter();

    Msg userMsg = Msg.builder()
        .role(MsgRole.USER)
        .content(TextBlock.builder().text(message).build())
        .build();

    return subAgent.call(List.of(userMsg))
        .flatMap(response -> {
            if (isHitlState(response)) {
                // 通过 emitter 发送结构化的 HITL 事件
                // 这个事件会被父 Agent 的 ActingChunkEvent Hook 捕获
                ToolResultBlock hitlChunk = ToolResultBlock.builder()
                    .id(param.getToolUseBlock().getId())
                    .name(param.getToolUseBlock().getName())
                    .output(List.of(TextBlock.builder()
                        .text(serializePendingTools(response))
                        .build()))
                    .metadata(Map.of(
                        "subagent_hitl", true,
                        "subagent_name", subAgent.getName(),
                        "pending_tools", extractPendingTools(response)))
                    .build();
                emitter.emit(hitlChunk);

                // 仍然抛出以触发父 Agent 挂起
                throw new ToolSuspendException("Sub-agent waiting for input");
            }
            return Mono.just(ToolResultBlock.text(response.getTextContent()));
        });
}
```

这个模式结合了两种机制：`ToolEmitter` 提供实时事件流（调用方可以通过监听 Agent 事件获取子 Agent 状态），`ToolSuspendException` 保证父 Agent 正确挂起。

### 一个关键差异：`param.getAgent()` 的利用

`SubAgentTool` 通过 `param.getAgent()` 获取 `RuntimeContext`（用于 context propagation），但**没有利用父 Agent 的其他能力**。自定义 `AgentTool` 可以做得更多：

```java
@Override
public Mono<ToolResultBlock> callAsync(ToolCallParam param) {
    Agent parentAgent = param.getAgent();  // ← 父 Agent 引用

    if (parentAgent instanceof ReActAgent parent) {
        // 可以读取父 Agent 的 memory 来获取上下文
        Memory parentMemory = parent.getMemory();

        // 可以复用父 Agent 的 toolkit（避免重复注册）
        // Toolkit parentToolkit = parent.getToolkit();
    }

    // ...
}
```

这为更高级的模式打开了可能性，比如：
- 子 Agent 继承父 Agent 的 memory 上下文
- 子 Agent 的 HITL 状态直接写入父 Agent 的 memory
- 父 Hook 监控子 Agent 的工具调用并做安全拦截

### 总结：AgentTool 层面的 HITL 支持矩阵

| 实现方式 | 挂起检测 | 挂起传播 | 状态保存 | 恢复能力 | 实现复杂度 |
|----------|---------|---------|---------|---------|-----------|
| **SubAgentTool（当前）** | ❌ | ❌ | ✅ | ❌ | 已内置 |
| **模式 1: 直接 throw** | ✅ | ✅ | ❌ | ❌ | 最低（~5 行） |
| **模式 2: throw + Session** | ✅ | ✅ | ✅ | ✅ | 中等（~30 行） |
| **模式 3: throw + Emitter** | ✅ | ✅ | ❌ | ❌ | 中等（~20 行） |

**核心结论**：`AgentTool.callAsync()` 内部的 `throw new ToolSuspendException()` 会被框架的 `ToolExecutor.onErrorResume()` 正确捕获并转换。这意味着**任何 `AgentTool` 实现都可以通过抛出 `ToolSuspendException` 来实现挂起**，不需要修改框架代码。恢复能力则需要自行管理子 Agent 的状态持久化。
