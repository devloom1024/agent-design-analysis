# AgentScope Java — Human-in-the-Loop 与 Tool Suspend 机制分析

## 概述

AgentScope Java 提供了**双层挂起/恢复机制**来实现 Human-in-the-Loop（HITL）：

| 层级 | 机制 | 触发点 | 挂起对象 | 使用场景 |
|------|------|--------|----------|----------|
| **Tier 1: 工具级挂起** | `ToolSuspendException` | 工具执行中 | 单个工具调用 | 外部工具执行、向用户询问信息 |
| **Tier 2: Agent 级暂停** | `Hook.stopAgent()` | 推理后 / 执行后 | 整个 Agent 迭代循环 | 敏感工具审批、工具结果审查 |

两种机制最终都通过 `agent.call()` **无参调用** 实现恢复。

---

## Tier 1：工具级挂起 — ToolSuspendException

### 核心流程

```
LLM 返回 tool_use → Toolkit 执行工具 → 工具抛出 ToolSuspendException
    → ToolExecutor.executeCore() 捕获 → 生成 ToolResultBlock(metadata.suspended=true)
    → ReActAgent.acting() 通过 isSuspended() 分离挂起结果
    → buildSuspendedMsg() 构造 GenerateReason.TOOL_SUSPENDED 消息
    → 返回给调用方，等待用户提供 tool result
```

### 1. ToolSuspendException — 挂起信号

```java
// agentscope-core/.../tool/ToolSuspendException.java
public class ToolSuspendException extends RuntimeException {
    private final String reason;  // 挂起原因，会透传给调用方

    public ToolSuspendException(String reason) {
        super(reason != null ? reason : "Tool execution suspended");
        this.reason = reason;
    }
}
```

这是一个简单的 RuntimeException 子类，只携带一个 `reason` 字符串。不包含任何恢复逻辑——恢复机制完全在外层 ReActAgent 中实现。

### 2. ToolExecutor.executeCore() — 异常捕获与转换

```java
// agentscope-core/.../tool/ToolExecutor.java (lines 244-263)
return tool.callAsync(executionParam)
    .onErrorResume(
        ToolSuspendException.class,
        e -> {
            // 关键：将异常转换为带 suspended 标记的 ToolResultBlock
            return Mono.just(ToolResultBlock.suspended(toolCall, e));
        })
    .onErrorResume(
        e -> {
            // 其他异常在此兜底，转为 error
            return Mono.just(ToolResultBlock.error("Tool execution failed: " + errorMsg));
        });
```

`onErrorResume` 是 Reactor 的错误恢复操作符。当 `ToolSuspendException` 被捕获时，**不会传播错误**，而是返回一个正常的 `ToolResultBlock`。这使得工具执行流在 Reactive 层面不中断，挂起状态完全通过消息元数据来表达。

### 3. ToolResultBlock — 挂起状态标记

```java
// agentscope-core/.../message/ToolResultBlock.java
public static final String METADATA_SUSPENDED = "agentscope_suspended";

public boolean isSuspended() {
    return Boolean.TRUE.equals(metadata.get(METADATA_SUSPENDED));
}

public static ToolResultBlock suspended(ToolUseBlock toolUse, ToolSuspendException exception) {
    String content = exception.getReason() != null
        ? exception.getReason()
        : "[Awaiting external execution]";
    return new ToolResultBlock(
        toolUse.getId(),
        toolUse.getName(),
        List.of(TextBlock.builder().text(content).build()),
        Map.of(METADATA_SUSPENDED, true));  // suspended 标记打在 metadata 中
}
```

关键设计点：挂起状态不是通过异常传播，而是通过**消息元数据**承载。`ToolResultBlock` 既是"正常结果"（`output` + `metadata`），又是"挂起信号"（`isSuspended() == true`）。这使得整个异步链保持畅通，挂起判定推迟到 ReActAgent 的消息处理层。

### 4. SchemaOnlyTool — 纯 Schema 工具

```java
// agentscope-core/.../tool/SchemaOnlyTool.java
public class SchemaOnlyTool implements AgentTool {
    @Override
    public Mono<ToolResultBlock> callAsync(ToolCallParam param) {
        return Mono.error(new ToolSuspendException());  // 永远抛出挂起
    }
}
```

用于注册**不由框架执行的外部工具**。典型场景：用户在 `Toolkit` 中注册一个纯 Schema 的工具，LLM 可以"调用"它，但实际执行需要外部客户端完成。`Toolkit.registerSchema(ToolSchema)` 内部会将 `ToolSchema` 包装为 `SchemaOnlyTool`。

### 5. UserInteractionTool — HITL 交互工具

```java
// agentscope-examples/advanced/.../hitl/UserInteractionTool.java
@Tool(name = "ask_user", description = "Ask the user for clarification...")
public String askUser(
    @ToolParam(name = "question") String question,
    @ToolParam(name = "ui_type", required = false) String uiType,    // text/select/confirm/form/date/number
    @ToolParam(name = "options", required = false) List<String> options,
    @ToolParam(name = "fields", required = false) List<Map<String, Object>> fields,
    ...) {
    throw new ToolSuspendException(question);  // 永远抛出挂起
}
```

这是 `ToolSuspendException` 的典型应用模式。工具声明了丰富的参数（UI类型、选项、表单字段等），这些参数被 LLM 填充后，进入 `ToolUseBlock`。即使工具抛出异常挂起，**`ToolUseBlock` 中的参数已经完整保存**，调用方可以从挂起消息中提取这些参数来渲染 UI。

---

## Tier 2：Agent 级暂停 — Hook.stopAgent()

### 核心流程

```
LLM 推理完成 → PostReasoningEvent 触发 Hook
    → Hook 判断需人工确认 → postReasoning.stopAgent()
    → ReActAgent.reasoning() 检测 stopRequested == true
    → 返回 GenerateReason.REASONING_STOP_REQUESTED 消息（含 ToolUseBlock）
    → 用户审查后调用 agent.call() 或 agent.call(toolResults) 恢复
```

### 1. PostReasoningEvent — 推理后拦截

```java
// agentscope-core/.../hook/PostReasoningEvent.java
public void stopAgent() {
    this.stopRequested = true;  // 设置停止标记
}

public void gotoReasoning(List<Msg> msgs) {
    // 另一种控制方式：不停止，而是插入消息后回到推理阶段
    ToolValidator.validateToolResultMatch(reasoningMessage, msgs);
    this.gotoReasoningMsgs = new ArrayList<>(msgs);
}
```

`stopAgent()` 是"暂停并等待"模式，而 `gotoReasoning()` 是"注入消息并继续"模式。两者互斥，但都实现了 Hook 对 Agent 循环的控制。

### 2. PostActingEvent — 执行后拦截

```java
// agentscope-core/.../hook/PostActingEvent.java
public void stopAgent() {
    this.stopRequested = true;  // 暂停在工具执行后
}

public void setToolResult(ToolResultBlock toolResult) {
    this.toolResult = toolResult;  // Hook 可修改工具执行结果
}
```

`PostActingEvent` 在工具执行完成后触发，Hook 可以：
- 审查工具结果并修改（`setToolResult`）
- 调用 `stopAgent()` 暂停，等待用户确认工具结果后再继续

### 3. ToolConfirmationHook 示例 — 敏感工具审批

```java
// agentscope-examples/quickstart/.../HookStopAgentExample.java
static class ToolConfirmationHook implements Hook {
    private static final List<String> SENSITIVE_TOOLS = List.of("delete_file", "send_email");

    @Override
    public <T extends HookEvent> Mono<T> onEvent(T event) {
        if (event instanceof PostReasoningEvent postReasoning) {
            Msg reasoningMsg = postReasoning.getReasoningMessage();
            List<ToolUseBlock> toolCalls = reasoningMsg.getContentBlocks(ToolUseBlock.class);

            boolean hasSensitive = toolCalls.stream()
                .anyMatch(tool -> SENSITIVE_TOOLS.contains(tool.getName()));

            if (hasSensitive) {
                postReasoning.stopAgent();  // 暂停！等待用户确认
            }
        }
        return Mono.just(event);
    }
}
```

---

## ReActAgent 主循环中的挂起/恢复逻辑

ReActAgent 是挂起/恢复机制的核心编排者。关键方法：`doCall()`、`reasoning()`、`acting()`、`buildSuspendedMsg()`。

### doCall() — 入口：区分三种调用模式

```java
// agentscope-core/.../ReActAgent.java (lines 364-398)
protected Mono<Msg> doCall(List<Msg> msgs) {
    Set<String> pendingIds = getPendingToolUseIds();

    // 情况1: 无挂起工具 → 正常处理
    if (pendingIds.isEmpty()) {
        addToMemory(msgs);
        return executeIteration(0);
    }

    // 情况2: 有挂起工具 + 无输入 → 恢复执行!
    if (msgs == null || msgs.isEmpty()) {
        return acting(0);  // 直接进入执行阶段，执行挂起的工具调用
    }

    // 情况3: 有挂起工具 + 有输入 → 验证用户提供的 tool result
    List<ToolResultBlock> providedResults = msgs.stream()
        .flatMap(m -> m.getContentBlocks(ToolResultBlock.class).stream())
        .toList();

    if (!providedResults.isEmpty()) {
        validateAndAddToolResults(msgs, pendingIds);
        return hasPendingToolUse() ? acting(0) : executeIteration(0);
    }

    // 异常：未提供 tool result，且 PendingToolRecoveryHook 被禁用
    throw new IllegalStateException("Pending tool calls exist without results...");
}
```

**恢复的核心机制**就在这里：`agent.call()`（无参）进入情况2，直接跳转到 `acting(0)` 执行挂起的工具。`agent.call(toolResultMsg)` 进入情况3，验证用户提供的工具结果后继续。

### reasoning() — 推理阶段的挂起检测

```java
// agentscope-core/.../ReActAgent.java (lines 610-643)
.flatMap(event -> {
    Msg msg = event.getReasoningMessage();
    if (msg != null) {
        memory.addMessage(msg);
    }

    // HITL 停止检测
    if (event.isStopRequested()) {
        return Mono.just(
            msg.withGenerateReason(GenerateReason.REASONING_STOP_REQUESTED));
    }

    // gotoReasoning 检测
    if (event.isGotoReasoningRequested()) {
        List<Msg> gotoMsgs = event.getGotoReasoningMsgs();
        if (gotoMsgs != null) {
            gotoMsgs.forEach(memory::addMessage);
        }
        return reasoning(iter + 1, true);  // 回到推理，不计入迭代上限
    }

    // 完成检测
    if (isFinished(msg)) {
        return Mono.just(msg);  // 正常结束
    }

    // 继续到 acting 阶段
    return checkInterruptedAsync().then(acting(iter));
});
```

`REASONING_STOP_REQUESTED` 消息中保留了 `ToolUseBlock`，调用方可以从消息中提取工具调用信息来展示给用户审查。

### acting() — 执行阶段的挂起/恢复

```java
// agentscope-core/.../ReActAgent.java (lines 669-731)
private Mono<Msg> acting(int iter) {
    List<ToolUseBlock> pendingToolCalls = extractPendingToolCalls();  // 只取未执行完的

    if (pendingToolCalls.isEmpty()) {
        return executeIteration(iter + 1);  // 全部已执行，继续推理
    }

    return notifyPreActingHooks(pendingToolCalls)
        .flatMap(this::executeToolCalls)
        .flatMap(results -> {
            // 分离成功和挂起的结果
            List<Map.Entry<ToolUseBlock, ToolResultBlock>> successPairs =
                results.stream().filter(e -> !e.getValue().isSuspended()).toList();
            List<Map.Entry<ToolUseBlock, ToolResultBlock>> pendingPairs =
                results.stream().filter(e -> e.getValue().isSuspended()).toList();

            if (successPairs.isEmpty() && !pendingPairs.isEmpty()) {
                return Mono.just(buildSuspendedMsg(pendingPairs));  // 全部挂起
            }

            // 成功的结果通过 PostActingEvent Hook 处理后加入 memory
            return Flux.fromIterable(successPairs)
                .concatMap(this::notifyPostActingHook)
                .last()
                .flatMap(event -> {
                    if (event.isStopRequested()) {
                        return Mono.just(
                            event.getToolResultMsg()
                                .withGenerateReason(GenerateReason.ACTING_STOP_REQUESTED));
                    }
                    if (!pendingPairs.isEmpty()) {
                        return Mono.just(buildSuspendedMsg(pendingPairs));
                    }
                    return executeIteration(iter + 1);
                });
        });
}
```

关键逻辑：
1. **`extractPendingToolCalls()`**：只执行那些在 memory 中还没有对应 `ToolResultBlock` 的工具调用。这是恢复后能"接续"执行的基础——上一次成功的结果已在 memory 中，不会被重复执行。
2. **成功与挂起分离**：`isSuspended()` 将结果分两路。成功的走 Hook 审查管线，挂起的直接构造 `TOOL_SUSPENDED` 消息返回。
3. **PostActingEvent.stopAgent()**：成功执行后也可以暂停，让用户审查工具结果。

### buildSuspendedMsg() — 构造挂起消息

```java
// agentscope-core/.../ReActAgent.java (lines 742-754)
private Msg buildSuspendedMsg(List<Map.Entry<ToolUseBlock, ToolResultBlock>> pendingPairs) {
    List<ContentBlock> content = new ArrayList<>();
    for (Map.Entry<ToolUseBlock, ToolResultBlock> pair : pendingPairs) {
        content.add(pair.getKey());   // ToolUseBlock (含完整参数)
        content.add(pair.getValue()); // ToolResultBlock (suspended 标记)
    }
    return Msg.builder()
        .name(getName())
        .role(MsgRole.ASSISTANT)
        .content(content)
        .generateReason(GenerateReason.TOOL_SUSPENDED)
        .build();
}
```

消息中同时包含 `ToolUseBlock` 和 `ToolResultBlock`，调用方可以：
- 从 `ToolUseBlock` 提取工具名、参数（用于渲染 UI）
- 从 `ToolResultBlock` 获知挂起原因（`getOutput()` 中的文本）
- 通过 `isSuspended()` 判断是否需要等待用户输入

### getPendingToolUseIds() — 挂起判定

```java
// agentscope-core/.../ReActAgent.java (lines 444-460)
private Set<String> getPendingToolUseIds() {
    Msg lastAssistant = findLastAssistantMsg();
    if (lastAssistant == null || !lastAssistant.hasContentBlocks(ToolUseBlock.class)) {
        return Set.of();
    }

    // 收集 memory 中已有的 tool result ID
    Set<String> existingResultIds = memory.getMessages().stream()
        .flatMap(m -> m.getContentBlocks(ToolResultBlock.class).stream())
        .map(ToolResultBlock::getId)
        .collect(Collectors.toSet());

    // 返回有 tool_use 但没有对应 tool_result 的 ID
    return lastAssistant.getContentBlocks(ToolUseBlock.class).stream()
        .map(ToolUseBlock::getId)
        .filter(id -> !existingResultIds.contains(id))
        .collect(Collectors.toSet());
}
```

挂起的判定完全基于 **memory 中的消息状态**：如果一个工具调用 ID 在 memory 中没有对应的结果消息，它就被视为"挂起"。这是恢复机制能工作的关键——memory 是持久化的，即使 Agent 实例被重新创建，只要加载了 memory 就可以接续执行。

---

## 恢复机制详解

### 恢复路径一览

```
                       ┌── REASONING_STOP_REQUESTED ──→ agent.call() → 进入 acting 执行工具
用户审查后恢复 ──┤
                       └── TOOL_SUSPENDED ──→ agent.call(toolResults) → 验证结果 → acting/推理
```

### 恢复路径 1：Hook 暂停后恢复（REASONING_STOP_REQUESTED）

调用方检测 `response.getGenerateReason() == GenerateReason.REASONING_STOP_REQUESTED` 或 `response.hasContentBlocks(ToolUseBlock.class)`，然后：

- **确认执行**：`agent.call()`（无参）→ 进入 `doCall()` 的情况2 → 直接 `acting(0)`
- **拒绝执行**：`agent.call(cancelResultMsg)` → 进入情况3 → `validateAndAddToolResults()` → 继续推理

```java
// 示例：HookStopAgentExample
Msg response = agent.call(userMsg).block();

while (response != null && response.hasContentBlocks(ToolUseBlock.class)) {
    // 显示待审批的工具调用
    displayPendingToolCalls(response);

    if (userConfirms) {
        response = agent.call().block();           // 恢复，执行工具
    } else {
        Msg cancelResult = createCancelledToolResults(response, agent.getName());
        response = agent.call(cancelResult).block(); // 注入取消结果
    }
}
```

### 恢复路径 2：工具挂起后恢复（TOOL_SUSPENDED）

调用方检测 `response.getGenerateReason() == GenerateReason.TOOL_SUSPENDED`，注意此时 `isSuspended()` 为 true。调用方执行外部工具后提交结果：

```java
// 构造 tool result
ToolResultBlock result = ToolResultBlock.of(
    toolUse.getId(), toolUse.getName(),
    TextBlock.builder().text("执行结果...").build());

Msg resultMsg = Msg.builder()
    .name("user")
    .role(MsgRole.TOOL)
    .content(result)
    .build();

// 恢复执行
Msg resumeResponse = agent.call(resultMsg).block();
```

### 恢复路径 3：会话持久化恢复

Agent 支持通过 Session 持久化 memory、toolkit 状态、PlanNotebook 等：

```java
// 保存
agent.saveTo(session, sessionKey);

// 恢复（新进程或新请求中）
ReActAgent agent = ReActAgent.builder()...build();
agent.loadFrom(session, sessionKey);

// 继续执行挂起的工具
Msg resumeResponse = agent.call().block();
```

`loadFrom()` 恢复 memory 后，`getPendingToolUseIds()` 能正确识别挂起状态，`agent.call()` 无参调用即可接续执行。**这是实现跨请求/跨进程 HITL 的基础**。

### PendingToolRecoveryHook — 孤儿工具自动恢复

```java
// agentscope-core/.../hook/PendingToolRecoveryHook.java
private Mono<PreCallEvent> handlePreCall(PreCallEvent event) {
    // 只在 ReActAgent 且有挂起工具时生效
    Set<String> pendingIds = findPendingToolUseIds(memory);
    if (pendingIds.isEmpty()) return Mono.just(event);

    // 关键判断：input 为空 → 用户在恢复 → 不修补
    if (inputMessages == null || inputMessages.isEmpty()) {
        return Mono.just(event);  // 放行，让 ReActAgent.doCall() 处理恢复
    }

    // input 不为空但无 tool result → 孤儿状态 → 自动生成 error result
    patchPendingToolCalls(reactAgent, memory, pendingIds);
}
```

这个 Hook 在 `PreCallEvent` 时运行（优先级 10，高优先级），处理异常场景：
- 工具因超时/崩溃没有返回结果
- memory 中存在无对应结果的 tool_use
- 用户没有提供结果
→ 自动生成 `[ERROR] Previous tool execution failed` 结果，让 Agent 能继续运行而不是抛出 `IllegalStateException`

---

## GenerateReason 枚举 — Agent 状态信号

```java
// agentscope-core/.../message/GenerateReason.java
public enum GenerateReason {
    MODEL_STOP,                  // 模型正常结束
    TOOL_CALLS,                  // 模型返回工具调用（内部工具，框架继续执行）
    STRUCTURED_OUTPUT,           // 结构化输出完成
    TOOL_SUSPENDED,              // 工具执行挂起，等待用户提供结果
    REASONING_STOP_REQUESTED,    // Hook 在推理后暂停
    ACTING_STOP_REQUESTED,       // Hook 在执行后暂停
    INTERRUPTED,                 // Agent 被中断
    MAX_ITERATIONS               // 达到最大迭代次数
}
```

调用方通过检查 `Msg.getGenerateReason()` 判断 Agent 状态并决定恢复策略。`TOOL_SUSPENDED` 和 `REASONING_STOP_REQUESTED` / `ACTING_STOP_REQUESTED` 是 HITL 的两个核心信号。

---

## 架构总结

```
                         ┌──────────────────────────────────┐
                         │         ReActAgent.doCall()       │
                         │   (挂起检测 + 路由分发)            │
                         └──────────┬───────────────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            ▼                       ▼                       ▼
    ┌───────────────┐     ┌─────────────────┐     ┌──────────────────┐
    │ 无挂起 → 正常  │     │ 有挂起 + 无输入  │     │ 有挂起 + 有输入   │
    │ executeIter() │     │ → acting() 恢复  │     │ → 验证tool result │
    └───────────────┘     └─────────────────┘     └──────────────────┘
                                    │
                          ┌─────────┴─────────┐
                          ▼                   ▼
                  ┌──────────────┐    ┌──────────────────┐
                  │ reasoning()  │    │    acting()       │
                  │              │    │  执行挂起工具       │
                  │ PostReasoning│    │  分离 success/     │
                  │   .stopAgent │    │  suspended 结果    │
                  └──────┬───────┘    └────────┬─────────┘
                         │                     │
                  REASONING_             TOOL_SUSPENDED
                  STOP_REQUESTED         ACTING_STOP_REQUESTED
                         │                     │
                         └─────────┬───────────┘
                                   ▼
                         ┌──────────────────┐
                         │   返回 Msg 给调用方 │
                         │   等待用户决策/输入  │
                         └────────┬─────────┘
                                  │
                    agent.call() 或 agent.call(toolResults)
                                  │
                                  ▼
                         ┌──────────────────┐
                         │  doCall() 再次进入 │
                         │  情况2/3 恢复执行   │
                         └──────────────────┘
```

### 关键设计决策

1. **挂起状态通过消息元数据而非异常传播**：`ToolSuspendException` 在 `ToolExecutor` 层就被转为 `ToolResultBlock(metadata.suspended=true)`，上层完全通过消息状态机来处理挂起/恢复。

2. **恢复判定完全基于 memory 状态**：`getPendingToolUseIds()` 通过比较 memory 中的 `ToolUseBlock` 和 `ToolResultBlock` ID 集来确定哪些工具需要执行/等待。这使得状态可以被持久化、跨进程恢复。

3. **`agent.call()` 无参 = 恢复信号**：不需要额外的 resume API。无参调用时，`doCall()` 检测到有挂起工具且无输入，自动进入 acting 执行阶段。

4. **双层机制互补**：`ToolSuspendException` 适合"这个工具需要外部执行"的场景（如 `SchemaOnlyTool`、`UserInteractionTool`），`StopAgent` 适合"执行前需要审查"的场景（如敏感操作审批）。两者可以在同一个 Agent 中共存。
