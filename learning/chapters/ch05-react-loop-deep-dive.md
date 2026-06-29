# Ch05 · ReAct 主循环源码精读

> 状态：🔲 · 预计时长：3h · 前置：Ch04

## 1. 本章目标

- 精读 `ReActAgent.java` 的核心方法：`buildAgentStream` / `doCall` / `executeIteration` / `reasoning` / `acting` / `summarizing`
- 理解"Reasoning → Acting → Reasoning → ... → Summarizing"的迭代模型
- 能用 `ScriptedModel` 模拟"模型发工具调用 → 工具返回 → 模型再推理"的两轮 ReAct
- 掌握 `Accumulator` 系列在流式聚合中的作用

## 2. 核心概念

### 2.1 ReAct 范式

ReAct = **Re**asoning + **Act**ing，由 Yao et al. 2022 提出：

```
Thought 1 → Action 1 → Observation 1 → Thought 2 → Action 2 → ... → Final Answer
```

在 `agentscope-java` 中对应：

| 阶段 | ReActAgent 方法 | 产物 |
|---|---|---|
| Reason | `reasoning(iter, false)` | `Msg`（含 `TextBlock` + 可能的 `ToolUseBlock`） |
| Act | `acting(iter)` | 工具执行结果 `ToolResultBlock`，追加为 `Msg(role=TOOL)` |
| Repeat | `executeIteration(iter+1)` | 重新 reason |
| Summarize | `summarizing()` | 最终回答 |
| Cap | `iter >= maxIters` | 强制走 summarize |

### 2.2 迭代终止条件

`ReActAgent.java:1835-1839`：

```java
private Mono<Msg> reasoning(int iter, boolean ignoreMaxIters) {
    if (!ignoreMaxIters && iter >= maxIters) {
        return summarizing();
    }
    // ...
}
```

终止方式有 4 种：

1. **模型不再发工具调用** —— `extractPendingToolCalls()` 返回空 → `acting()` 不做事 → `executeIteration(iter+1)` → 下一轮 reasoning
2. **达到 maxIters** —— 默认 10，可调
3. **HITL 人工中断** —— `RequireUserConfirmEvent` 后用户拒绝
4. **Middleware 主动停止** —— `RequestStopEvent`（带 `GenerateReason`）

### 2.3 `ActingInput` / `ReasoningInput` / `AgentInput` —— 中间件协议

`agentscope-core/src/main/java/io/agentscope/core/middleware/`：

```java
public record ReasoningInput(List<Msg> messages, List<ToolSchema> tools, GenerateOptions options) {}
public record ActingInput(List<ToolUseBlock> toolCalls) {}
public record AgentInput(List<Msg> messages) {}  // 最外层
public record ModelCallInput(...) {}  // 模型调用前
public record TaskReminderInput(...) {}  // 任务提醒
```

**设计意图**：每个阶段给中间件一个**强类型**的输入 / 输出契约，比 `Map<String,Object>` 友好得多。

### 2.4 累积器 `Accumulator`

`agentscope-core/src/main/java/io/agentscope/core/agent/accumulator/`：

| 累积器 | 作用 |
|---|---|
| `TextAccumulator` | 合并 `TextBlockDeltaEvent` 流为完整 `TextBlock` |
| `ThinkingAccumulator` | 合并 `ThinkingBlockDeltaEvent` 为完整 `ThinkingBlock` |
| `ToolCallsAccumulator` | 合并 `ToolCallDeltaEvent` 为完整 `ToolUseBlock` |
| `ContentAccumulator` | 上面三个的总入口，输出 `Msg` |

LLM 流式返回时，每个 token 一个 `*DeltaEvent`，累积器把它们**拼起来**形成最终 `Msg`。

## 3. 源码精读

### 3.1 入口：`buildAgentStream`

`ReActAgent.java:795-850`：

```java
private Flux<AgentEvent> buildAgentStream(
        List<Msg> msgs, RuntimeContext context, Function<List<Msg>, Mono<Msg>> doCallFn) {
    String replyId = UUID.randomUUID().toString().replace("-", "");
    Function<AgentInput, Flux<AgentEvent>> core = input -> Flux.create(sink -> {
        sink.next(new AgentStartEvent(null, replyId, getName()));
        // ...
        Mono<Msg> lifecycle = runLifecycle(input.msgs(), doCallFn);
        if (context != null) {
            lifecycle = lifecycle.contextWrite(c -> c.put(RUNTIME_CONTEXT_KEY, context));
        }
        // ...
        lifecycle
            .contextWrite(c -> c.put(EVENT_SINK_KEY, sink))
            // ...
            .doFinally(signal -> {
                sink.next(new AgentEndEvent(replyId));
                sink.complete();
            })
            .subscribe(
                finalMsg -> sink.next(new AgentResultEvent(finalMsg)),
                sink::error
            );
    }, FluxSink.OverflowStrategy.BUFFER);

    return MiddlewareChain.build(middlewares, this, context, MiddlewareBase::onAgent, core)
            .apply(new AgentInput(msgs == null ? List.of() : msgs));
}
```

**观察 1**：用 `Flux.create` + `OverflowStrategy.BUFFER` 手动驱动事件流，**绕开** `Reactive Streams` 自然的订阅关系，让 Hook/Middleware 能精确控制何时发射事件。

**观察 2**：`replyId` 是这次调用的"会话键"，所有事件都带它 —— 客户端可用 `replyId` 关联同一调用的所有事件。

**观察 3**：最外层套 `onAgent` 中间件链 —— 用户的整个 ReAct 循环**也可以被中间件包**（典型用法：重试、超时、降级）。

### 3.2 `runLifecycle` 模板

`ReActAgent.java` 间接继承自 `AgentBase.java:200+` 的 `runLifecycle`，负责：

1. 关闭检查（`GracefulShutdownManager`）
2. session 锁获取（同 session 串行）
3. 加载 / 创建 state
4. `firePreCall` Hook
5. 调 `doCall(msgs)`（**子类实现**）
6. 追加用户消息 + 最终回复到 state
7. `firePostCall` Hook
8. 释放锁

`ReActAgent.doCall(msgs)`（`ReActAgent.java:925`）是真正实现：

```java
protected Mono<Msg> doCall(List<Msg> msgs) {
    // 1. 把用户消息追加到 state
    state.contextMutable().addAll(msgs);
    // 2. 检查是否有 pending tool calls（恢复执行）
    if (hasPendingToolCalls(state.contextMutable())) {
        return resumeAgent();
    }
    // 3. 正常启动 ReAct 循环
    return coreAgent();
}
```

### 3.3 `coreAgent` → `executeIteration` → `reasoning` → `acting`

`ReActAgent.java:1809-1823`：

```java
private Mono<Msg> coreAgent() {
    return executeIteration(0);
}

private Mono<Msg> executeIteration(int iter) {
    return reasoning(iter, false);
}
```

`reasoning`（L1835-1962）做完一次模型推理，**判断要不要进 acting 的逻辑不在 `reasoning` 内部**，而在后续的 `runPostReasoningPipeline`（L1965-2005）：

```java
private Mono<Msg> reasoning(int iter, boolean ignoreMaxIters) {
    if (!ignoreMaxIters && iter >= maxIters) return summarizing();

    return checkInterrupted()
        .then(hookDispatcher.firePreReasoning(...))
        .flatMap(event -> {
            // 1. 调模型（套 onReasoning 中间件）
            Flux<AgentEvent> stream = MiddlewareChain.build(
                middlewares, ReActAgent.this, rc,
                MiddlewareBase::onReasoning, reasoningCore
            ).apply(new ReasoningInput(modelInput, tools, options));

            // 2. 累积 + 折叠
            return stream.doOnNext(ev -> { /* 累积到 context */ })
                         .then(Mono.defer(() -> Mono.justOrEmpty(context.buildFinalMessage())));
        });
}
```

`reasoning` 完成后由 `runPostReasoningPipeline` 决定下一步：

```java
private Mono<Msg> runPostReasoningPipeline(Msg eventMsg, int iter) {
    // 关键：先用 isFinished 判断"模型这次响应是否构成终态"
    if (isFinished(eventMsg)) {
        return Mono.just(eventMsg);   // 直接返回，不再迭代
    }
    return acting(iter);              // 否则进入 acting
}
```

`acting`（L2167-2248）内部反过来处理"无待执行工具"的情况：

```java
private Mono<Msg> acting(int iter) {
    List<ToolUseBlock> pendingToolCalls = extractPendingToolCalls();
    if (pendingToolCalls.isEmpty()) {
        // 没有工具可调 —— 推进到下一轮（这才是"iter+1"真正发生的地方）
        return executeIteration(iter + 1);
    }
    // ...
    return executeIteration(iter + 1);   // 工具执行完也总是推进一步
}
```

**观察 4（修正）**：
- "是否还要 reason" 的判断**不在 `reasoning` 内部**，而是在 `runPostReasoningPipeline` 中用 `isFinished(eventMsg)` 判定。
- "iter+1" 真正发生的位置是 **`acting` 的两个返回点**（无工具时、有工具时），不是 `reasoning` 的尾巴。
- 纯文本响应（没有 ToolUseBlock）走的是 `isFinished` → return 这条快路径，**不会**触发 iter+1。

`acting`（2167 行）：

```java
private Mono<Msg> acting(int iter) {
    List<ToolUseBlock> pendingToolCalls = extractPendingToolCalls();
    if (pendingToolCalls.isEmpty()) return executeIteration(iter + 1);

    return hookDispatcher.firePreActing(pendingToolCalls, toolkit)
        .flatMap(toolCalls -> {
            // 1. 套 onActing 中间件
            Flux<AgentEvent> stream = MiddlewareChain.build(
                middlewares, ReActAgent.this, rc,
                MiddlewareBase::onActing, actingCore
            ).apply(new ActingInput(toolCalls));
            // 2. 等待所有工具完成（resultHolder 收集）
            return stream.doOnNext(/* 跟踪 RequestStopEvent */)
                         .then(Mono.defer(() -> Mono.just(resultHolder.get())));
        })
        .flatMap(results -> {
            // 3. 拼装 ToolResultMsg，追加到 state
            // 4. 进入下一轮
            return executeIteration(iter + 1);
        });
}
```

**观察 5**：`acting` 用 `resultHolder: AtomicReference` 收集所有工具结果，最后一次性取用。

**观察 6**：`acting` 完成后**总是** `executeIteration(iter+1)` —— 即使所有工具都失败，也强制 reasoning 一次（让模型看到"我之前的工具都失败了"以决定下一步）。

### 3.4 `summarizing` 阶段

- `summarizing()` 方法在 **`ReActAgent.java:2838`**（公开方法，做终止判断 + 触发 summary）
- `summaryModelCallStream(...)` 在 **`ReActAgent.java:2937`**（私有方法，做流式 summary 调用，约 70 行）

两者关系：`summarizing()` → 准备输入 → 调 `summaryModelCallStream(...)` → 折叠流到 `Mono<Msg>`。

- 触发条件：`iter >= maxIters` 或 `RequestStopEvent` 带 `GenerateReason.SUMMARIZE`
- 行为：**不传 tools**，让模型生成纯文本总结
- 作用：避免无限循环，保证 agent 最终**有输出**

### 3.5 流式事件的生命周期

`agentscope-core/src/main/java/io/agentscope/core/event/` 约 30 种事件，按时间线：

```
AgentStartEvent
   ↓
PreCallEvent
   ↓
PreReasoningEvent
   ↓
ReasoningEvent
   ↓
ReasoningChunkEvent (×N)
   ↓
ThinkingBlockStartEvent → Delta×N → EndEvent
TextBlockStartEvent → Delta×N → EndEvent
ToolCallStartEvent → Delta×N → EndEvent
   ↓
PostReasoningEvent
   ↓
PreActingEvent
   ↓
ToolResultStartEvent → Delta×N → EndEvent (×N 个工具)
   ↓
PostActingEvent
   ↓
[重复 reasoning-acting 直到 iter >= maxIters 或不再发工具调用]
   ↓
PreSummaryEvent → SummaryChunkEvent ×N → SummaryEvent
   ↓
PostSummaryEvent
   ↓
PostCallEvent
   ↓
AgentResultEvent (最终 Msg)
   ↓
AgentEndEvent
```

`streamEvents()` 把所有这些都吐出来；`call()` 只取 `AgentResultEvent` 里的 `Msg`。

## 4. 设计权衡

| 选择 | 原因 |
|---|---|
| 强制走 Summarize（即使没工具调用） | 防止无意义循环 |
| Pending tool call 触发 `resumeAgent` | 支持"工具异步 → 后续轮次回填结果"模式 |
| `AtomicReference` 跟踪中间件 stop | 反应式跨步骤传值的常见手法 |
| `iter` 每次 +1 包含纯文本 | 强制 eventual summarization |
| 三阶段（Reasoning / Acting / Summary）独立 Middleware | 精细化控制 |

## 5. 实验任务

详见 [`lab/ch05-react-with-scripted-model.md`](../lab/ch05-react-with-scripted-model.md)。核心：

1. 写一个**两轮** ScriptedModel：第一次返回 ToolUseBlock，第二次返回纯文本
2. 注册一个 EchoTool（返回固定字符串）
3. 跑通完整 ReAct：reasoning → acting(执行工具) → reasoning → 文本 → summarizing
4. 用 `streamEvents()` 打印所有事件，验证 §3.5 的生命周期

## 6. 思考题

1. 如果 `ScriptedModel` 在第一轮返回两个 ToolUseBlock，框架是串行还是并行执行？
2. 如果工具调用本身又触发模型调用（sub-agent），会发生什么？
3. `maxIters=0` 会发生什么？summarizing 会被跳过吗？

## 7. 参考资料

- 原始 ReAct 论文：<https://arxiv.org/abs/2210.03629>
- `docs/v2/en/docs/building-blocks/agent.md` 6.1 ~ 6.4 节
- Reactor `Flux.create`：<https://projectreactor.io/docs/core/release/reference/#_push_or_create_patterns>

## 8. 学习笔记

在 `notes/ch05-my-takeaways.md` 写 3-5 条金句。

---

> 上一章：[Ch04](./ch04-agent-and-state.md) · 下一章：[Ch06](./ch06-toolkit-and-function-calling.md)
