# LobeHub — Agent 生命周期

## Agent 执行引擎架构

**设计理念**：Runtime（引擎）+ Agent（大脑）分离。Agent 负责产生指令，Runtime 负责执行。

### 六态状态机

```
idle → running → waiting_for_human → done
                  ↓                   ↑
              interrupted ←───────────┘
```

## Plan → Execute 循环

### 13 种指令类型

```typescript
type AgentInstructionType =
  | 'call_llm'              // 调用 LLM
  | 'call_tool'             // 执行单个工具
  | 'call_tools_batch'      // 批量执行工具
  | 'resolve_aborted_tools' // 解决已中止的工具
  | 'exec_sub_agent'        // 执行子 Agent
  | 'exec_sub_agents'       // 批量执行子 Agent
  | 'exec_client_sub_agent' // 客户端子 Agent
  | 'exec_client_sub_agents'// 批量客户端子 Agent
  | 'request_human_prompt'  // 请求人工输入
  | 'request_human_select'  // 请求人工选择
  | 'request_human_approve' // 请求人工审批
  | 'compress_context'      // 压缩上下文
  | 'finish';               // 结束
```

### step() 核心逻辑

```
step(state, context):
  1. structuredClone(state) + stepCount += 1
  2. 检查 maxSteps:
     - 超限 → forceFinish = true (允许已完成工具，下次 LLM 只产最终文本)
  3. 根据 phase 分流:
     - human_approved_tool → 直接创建 call_tool 指令
     - 其他 → agent.runner(runtimeContext, newState) 获取指令
  4. 标准化旧格式指令
  5. 顺序执行所有指令:
     - call_llm → LLM 流式调用 → llm_start/stream/result 事件
     - call_tool → ToolRegistry 查找 handler → 执行 → tool_result
     - call_tools_batch → pMap 并发执行 + mergeToolResults
     - request_human_* → status = 'waiting_for_human' + 发射事件
     - compress_context → 压缩 → compression_complete
     - finish → status = 'done' + done 事件
  6. 每次执行后累加事件 + 更新状态
  7. waiting_for_human / interrupted → 停止执行
  8. finish 指令不计入 stepCount
```

### call_llm executor 详细流程

```
1. status = 'running'
2. 发射 llm_start 事件
3. modelRuntime.chat(payload) 流式迭代:
   - 每个 chunk → llm_stream 事件
   - 累加 assistantContent + toolCalls
4. 完成 → llm_result 事件
5. calculateUsage + calculateCost
6. 成本限制检查:
   - stop: 立即完成 done
   - interrupt: 中断（可恢复）
   - 默认: 警告 + 继续
7. 返回 nextContext (phase='llm_result')
```

### call_tool executor 详细流程

```
1. 按 apiName 从 ToolRegistry 查找 handler
2. handler.exec(args)
3. 结果追加到 messages
4. 发射 tool_result 事件
5. 更新 usage/cost，检查成本限制
6. 返回 nextContext (phase='tool_result')
```

## 中断和恢复

### interrupt()

```
1. status = 'interrupted'
2. 存储 interruption { reason, interruptedAt, interruptedInstruction?, canResume }
3. 发射 interrupted 事件
```

### resume()

```
1. 验证 status == 'interrupted' && canResume
2. 清空 interruption → status = 'running'
3. 发射 resumed 事件
4. 如有 context → 继续 step()
```

### approveToolCall()

```
1. 创建 phase='human_approved_tool' 的 context
2. step(state, context)
```

## 完整 Plan → Execute 循环

```
[用户输入]
    ↓
AgentRuntime.step()
    ├── Agent.runner() → [指令数组]          ← Plan
    │      ├── call_llm    → LLM 流式调用
    │      ├── call_tool   → 工具执行
    │      ├── call_tools_batch → 并发工具
    │      ├── request_human_* → 等待用户
    │      ├── exec_sub_agent  → 子 Agent
    │      ├── compress_context → 压缩
    │      └── finish      → 完成
    │
    └── 返回 events[], newState, nextContext
            ↓ (循环)
    [nextContext → 下一轮 Agent.runner()]
            ↓
    ... 直到 status = 'done' / 'error' / 'waiting_for_human'
```

## 完成原因（11 种）

```typescript
type FinishReason =
  | 'completed'               // 正常完成
  | 'user_requested'          // 用户请求结束
  | 'user_aborted'            // 用户中止
  | 'max_steps_exceeded'      // 超过最大步数
  | 'max_steps_completed'     // 达到最大步数完成
  | 'cost_limit_exceeded'     // 超过成本限制
  | 'timeout'                 // 执行超时
  | 'agent_decision'          // Agent 决定结束
  | 'queued_message_interrupt'// 排队消息软中断
  | 'error_recovery'          // 不可恢复错误
  | 'system_shutdown';        // 系统关闭
```
