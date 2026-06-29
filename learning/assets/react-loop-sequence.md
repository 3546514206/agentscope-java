# ReAct 主循环时序图

> 引用自 `agentscope-core/src/main/java/io/agentscope/core/ReActAgent.java:795 buildAgentStream`（**注意：不是 L769 callInternal** —— `callInternal` 只有 5 行壳，本图描述的是它驱动的真实 7 步事件流）

## 整体一次 `agent.call(msg, ctx)` 的事件流

```mermaid
sequenceDiagram
    autonumber
    participant User as 用户/调用方
    participant Agent as AgentBase
    participant ReAct as ReActAgent.callInternal
    participant Hook as HookDispatcher
    participant MW as MiddlewareChain
    participant Model as ChatModelBase
    participant Toolkit as Toolkit
    participant Tool as ToolBase
    participant State as AgentState

    User->>Agent: call(msg, ctx)
    Agent->>Agent: 加载/恢复 state (key=userId+sessionId)
    Agent->>ReAct: callInternal(state, msg, ctx)
    ReAct->>Hook: firePreCall
    ReAct->>ReAct: 追加 msg 到 state.context

    loop iter < maxIters
        ReAct->>Hook: firePreReasoning
        Hook-->>ReAct: 可能的 GenerateOptions / messages 覆盖
        ReAct->>MW: build(ReasoningInput)
        MW->>Model: doStream(messages, tools, options)
        Model-->>MW: Flux<ChatResponse>
        MW-->>ReAct: Flux<AgentEvent>
        ReAct->>ReAct: ReasoningContext 累积
        ReAct->>Hook: firePostReasoning

        alt 响应包含 ToolUseBlock
            ReAct->>Hook: firePreActing
            loop 每个 toolCall
                ReAct->>Toolkit: executeToolCall(param)
                Toolkit->>Tool: callAsync(param)
                Tool-->>Toolkit: Mono<ToolResultBlock>
                Toolkit-->>ReAct: 携带权限/并发控制
            end
            ReAct->>Hook: firePostActing
            ReAct->>ReAct: 拼装 ToolResultMsg 追加 state
        else 纯文本响应
            ReAct->>ReAct: break（不再进入下一轮）
        end
    end

    ReAct->>Hook: firePreSummary (iter >= maxIters)
    ReAct->>Model: summaryModelCallStream(...)
    Model-->>ReAct: 最终回答
    ReAct->>Hook: firePostSummary
    ReAct->>State: 持久化
    ReAct->>Hook: firePostCall
    ReAct-->>User: Mono<Msg>（终态消息）
```

## 中间件与 Hook 的差异

```mermaid
graph LR
    Input[输入] --> PreHook[Pre* Hook<br/>观察+可修改]
    PreHook --> PreMW[Pre* Middleware<br/>包裹+可短路]
    PreMW --> Core[核心逻辑]
    Core --> PostMW[Post* Middleware<br/>包裹+可转换]
    PostMW --> PostHook[Post* Hook<br/>观察+可修改]
    PostHook --> Output[输出]

    style PreHook fill:#fef3c7
    style PreMW fill:#dbeafe
    style Core fill:#dcfce7
    style PostMW fill:#dbeafe
    style PostHook fill:#fef3c7
```

- **Hook**：事件订阅者，**可观察 + 可修改**事件载荷，不控制流（不能短路）
- **Middleware**：AOP 拦截器，**可包裹**核心逻辑，**可短路 / 转换**输入输出

## 工具调用一次往返

```mermaid
sequenceDiagram
    autonumber
    participant ReAct as ReActAgent
    participant TK as Toolkit
    participant Perm as PermissionEngine
    participant Invoker as ToolMethodInvoker
    participant Tool as ToolBase

    ReAct->>TK: executeToolCall(ToolCallParam)
    TK->>Perm: checkPermissions(input, ctx)
    Perm-->>TK: PermissionDecision(allow/deny/ask)
    alt allow
        TK->>Invoker: invoke(method, args)
        Invoker->>Tool: callAsync(param)
        Tool-->>Invoker: Mono<ToolResultBlock>
        Invoker-->>TK: 结果
    else ask
        TK->>ReAct: RequireUserConfirmEvent
        ReAct-->>User: 等待 ConfirmResult
    else deny
        TK-->>ReAct: ToolResultBlock.error("denied")
    end
    TK-->>ReAct: ToolResultBlock
    ReAct->>ReAct: 包装为 ToolResultMessage 追加 state
```
