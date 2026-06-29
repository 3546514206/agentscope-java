# Ch04 · Agent 抽象与 `AgentState`

> 状态：🔲 · 预计时长：2h · 前置：Ch03

## 1. 本章目标

- 理解 `Agent` 接口的"三合一"设计：`CallableAgent` + `StreamableAgent` + `ObservableAgent`
- 掌握 `AgentBase` 提供的通用能力（事件总线、Hook 调度、状态加载）
- 理解 `AgentState` 替代旧 `Memory` 的设计意图
- 能用 `ScriptedModel` 跑通一个**最小可工作**的 ReActAgent

## 2. 核心概念

### 2.1 `Agent` 接口的复合设计

读 `agentscope-core/src/main/java/io/agentscope/core/agent/Agent.java:47`：

```java
public interface Agent
        extends CallableAgent, StreamableAgent, ObservableAgent {
    String getAgentId();
    String getName();
    default String getDescription() { return "Agent(" + getAgentId() + ") " + getName(); }
    void interrupt();
    void interrupt(Msg msg);
    default AgentState getAgentState() { return null; }
    default Toolkit getToolkit() { return null; }
}
```

三个子接口分别负责：

| 接口 | 方法 | 场景 |
|---|---|---|
| `CallableAgent` | `Mono<Msg> call(Msg)` | 同步拿最终回复 |
| `StreamableAgent` | `Flux<AgentEvent> streamEvents(Msg)` | UI 流式渲染 |
| `ObservableAgent` | `Mono<Void> observe(Msg)` | 多 Agent 协作时只听不回 |

**设计意图**：

- `call` 和 `streamEvents` 是同一调用的两个**视图** —— `call` 只取终态 `Msg`，`streamEvents` 暴露完整事件流；二者底层走同一管线，但只暴露其一（看 `Agent.java:42-46` 的 "Reply contract" Javadoc 注释）
- `observe` 实现"**订阅但不响应**"，是 multi-agent 编排的基础（主 Agent 监听子 Agent 的输出但不立即回话）
- `interrupt()` 是**协作式中断**（检查点机制，非强制 kill）

**关键纠正**（与之前报告相比）：
- `Agent.java:42-46` 的注释是 **"Reply contract"**（说明一次 `call(...)` 只产生一个终态 `Msg`），**不是**"互斥视图"
- 之前报告里写的"互斥视图"原话在该文件中**不存在**

### 2.2 `AgentBase` 的通用能力

读 `agentscope-core/src/main/java/io/agentscope/core/agent/AgentBase.java:92`（**类声明**）：

```java
// AgentBase.java:92 - 类声明
public abstract class AgentBase implements Agent {

    // 1. 身份字段
    protected final String agentId;
    protected final String name;
    protected final String description;

    // 2. Hook 列表（不是 HookDispatcher）
    private final List<Hook> hooks;
    private static final List<Hook> systemHooks = new CopyOnWriteArrayList<>();
    private final Map<String, List<AgentBase>> hubSubscribers = new ConcurrentHashMap<>();

    // 3. 同 key 串行化（用 Mono<Void> 链式追加实现，不是 Semaphore）
    private final ConcurrentMap<Object, Mono<Void>> callGates = new ConcurrentHashMap<>();
    private final List<Hook> runtimeContextAwareHooks = new CopyOnWriteArrayList<>();
    // （L92-110 范围都是字段声明，方法从 L120+ 才开始）

    // 4. 关键方法：runLifecycle 模板（**抽象**）
    protected abstract Mono<Msg> runLifecycle(List<Msg> msgs, Function<List<Msg>, Mono<Msg>> doCallFn);
    // 实际是 protected 而非 abstract —— 但每个子类必须实现

    // 5. callInternal **没有默认实现**（由各子类按需覆盖）
    //   ReActAgent.java:769 才有 5 行的具体实现
    protected Mono<Msg> callInternal(List<Msg> msgs, RuntimeContext context, Function<List<Msg>, Mono<Msg>> doCallFn) { ... }
}
```

**关键纠正（与之前报告相比）**：

- **L91-216 不是"通用能力"**——L91 是 Javadoc 收尾段（`* }` 之类），L92 才是类声明。整个 AgentBase 通用能力分布在 **L92-1050**（共 1035 行），不要按 91-216 这段去找
- **没有"callInternal 默认实现（L210-216）"**——`AgentBase` 中没有 L210-216 的默认 `callInternal`；实际是 `ReActAgent.java:769` 才有 5 行具体实现
- `callGates` **不是 Semaphore**，是 `ConcurrentMap<Object, Mono<Void>>`（每个 key 持一个 `Mono<Void>` 链）

**关键观察 1**：`call` 方法是 `final` 模板方法（位于 `AgentBase`，约 L630-660 范围）；`callInternal` 由各子类按需覆盖。这是 Template Method 模式的应用。

**关键观察 2**：**状态加载不在 `AgentBase` 内部** —— `eventEmitter` / `HookDispatcher` / `stateStore` / `stateCache` 这些字段都**在 `ReActAgent` 内部**（详见 §3.1）。`AgentBase` 只关心 Hook 列表和同 key 串行化。

**关键观察 3**：`callGates: ConcurrentMap<Object, Mono<Void>>` 是 `AgentBase` 唯一的并发控制原语。**没有 Semaphore、没有专门的"session 锁"** —— 串行化通过把每个 key 对应一个 `Mono<Void>` 链实现。

### 2.3 `AgentState` —— 状态的新范式

读 `agentscope-core/src/main/java/io/agentscope/core/state/AgentState.java`：

```java
public final class AgentState {
    private final String sessionId;
    private final String userId;
    private final List<Msg> context;          // 消息历史
    private final String summary;              // 长对话压缩后的摘要
    private final int curIter;                 // 当前迭代次数
    private final boolean shutdownInterrupted; // 优雅停机标记
    private final PermissionContextState permissionContext;
    private final ToolContextState toolContext;
    private final TaskContextState tasksContext;
    // ... builder 在 269 行
}
```

**vs 旧 `Memory` 接口**：

| 维度 | 旧 `Memory`（v1） | 新 `AgentState`（v2） |
|---|---|---|
| 抽象层级 | 只管消息列表 | 消息 + 摘要 + 迭代 + 权限 + 工具上下文 + 任务 |
| 持久化 | `saveTo(stateStore, userId, sessionId)` | 由 `AgentStateStore` 直接序列化整个 state |
| 线程安全 | 需要外部锁 | 不可变 + 写时复制 |
| 状态 | `@Deprecated` forRemoval | 推荐 |

迁移路径（看 `Memory.java:30-34` 的 Javadoc）：

```java
@Deprecated(forRemoval = true, since = "2.0.0")
public interface Memory {
    void saveTo(AgentStateStore stateStore, String userId, String sessionId);
    void loadFrom(AgentStateStore stateStore, String userId, String sessionId);
    // ...
}
```

v1 用户的 `Memory` 实现**作为写兼容镜像**保留，但不推荐新代码使用。

### 2.4 `RuntimeContext` —— 每次调用的钥匙

读 `agentscope-core/src/main/java/io/agentscope/core/agent/RuntimeContext.java`：

```java
public class RuntimeContext {
    private final String sessionId;    // 会话 ID（同一会话共享状态）
    private final String userId;        // 用户 ID（多租户隔离）
    private final Map<String, Object> attributes;  // 业务属性透传
    // ...
}
```

**为什么必须传**：

- 框架**不**在 Agent 实例里保存 `userId` —— 因为一个 Agent 实例服务多个用户
- 每次 `call` 通过 `RuntimeContext` 传递身份信息
- 状态查找实际是 **`ReActAgent.loadOrCreateAgentStateForSlot`（`ReActAgent.java:357`）**—— 注意**不在 `AgentBase`**；`AgentBase` 只管"按 key 串行化"，不管"按 key 加载 state"

### 2.5 `UserAgent` —— 一个具体的多 Agent 例子

读 `agentscope-core/src/main/java/io/agentscope/core/agent/user/UserAgent.java:69`：

```java
public class UserAgent extends AgentBase {
    // UserAgent 模拟"用户"，用于多 Agent 系统中把人类输入包成 Agent
    // observe() 接收外部输入；call() 返回用户消息
}
```

这印证了 `ObservableAgent` 的设计目的。

## 3. 源码精读

### 3.1 状态加载实际在 `ReActAgent` 内部

`AgentBase` **不持有状态**。状态加载在 `ReActAgent.loadOrCreateAgentStateForSlot`（`ReActAgent.java:357`）：

```java
private Mono<AgentState> loadOrCreateAgentStateForSlot(RuntimeContext rc, AgentStateSlot slot) {
    String key = rc.getStateKey();
    return Mono.defer(() -> {
        // 1. 先看内存缓存（stateCache 在 ReActAgent 内）
        AgentState cached = stateCache.get(key);
        if (cached != null) return Mono.just(cached);
        // 2. 缓存未命中，从 stateStore 加载
        return stateStore.load(rc.getUserId(), rc.getSessionId(), ...)
            .switchIfEmpty(Mono.defer(() -> Mono.just(AgentState.builder()
                .sessionId(rc.getSessionId())
                .userId(rc.getUserId())
                .build())))
            .doOnNext(state -> stateCache.put(key, state));
    });
}
```

**观察 1**：`switchIfEmpty` 处理"无历史记录"情况，新建空 state。

**观察 2**：`stateCache` 是 `ReActAgent` 内部的 `ConcurrentMap`（L262），不是 `AgentBase`。同一 session 的并发请求会共享 state；同 key 串行化在 `AgentBase` 通过 `callGates` + `serializeOnKey`（L344）实现，**不是** Semaphore。

### 3.2 同 key 串行化机制

读 `AgentBase.java:344 serializeOnKey`（这是 Javadoc 的 `callSerializationKey` 方法，**L324-336** 是 Javadoc 与方法声明）：

```java
// 不同 key 并发执行；同一 key 串行执行
private final ConcurrentMap<Object, Mono<Void>> callGates = new ConcurrentHashMap<>();

private <T> Mono<T> serializeOnKey(Object key, Mono<T> work) {
    Mono<Void> gate = callGates.compute(key, (k, prev) -> {
        Mono<Void> chain = (prev == null) ? Mono.empty() : prev;
        return chain.then();  // 链式：每个新请求都追加到链尾
    });
    // ...
}
```

**意图**：保证对同一 key（典型是 `RuntimeContext` 派生的 session 标识）的并发 `call` **串行执行**，避免状态写竞争。不同 key 完全独立。**注意**：实现用的是 `Mono<Void>` 链式追加，不是 Semaphore。

### 3.3 `UserAgent` 的真实结构

读 `agentscope-core/src/main/java/io/agentscope/core/agent/user/UserAgent.java:68`（类已 `@Deprecated forRemoval since 2.0.0`）：

```java
@Deprecated(forRemoval = true, since = "2.0.0")
public class UserAgent extends AgentBase {

    private UserInputBase inputMethod;   // 可插拔输入源
    // 默认 inputMethod = StreamUserInput（System.in / System.out）

    @Override
    protected Mono<Msg> doCall(List<Msg> msgs) {
        return getUserInput(msgs, null);
    }

    public Mono<Msg> getUserInput(List<Msg> contextMessages, Class<?> structuredModel) {
        return inputMethod
            .handleInput(getAgentId(), getName(), contextMessages, structuredModel)
            .map(this::createMessageFromInput)
            .doOnNext(this::printMessage);
    }
}
```

**关键纠正**：
- `UserAgent` **没有** `Queue<Msg> pendingInputs`、没有 `pushUserInput`、没有 `Mono.fromCallable(() -> pendingInputs.take())` 这些字段/方法。
- 它通过 **`UserInputBase` 接口**（默认实现 `StreamUserInput`）拿到用户输入；想换输入源（Web UI、文件、消息队列）就实现 `UserInputBase` 注入。
- 类已被 `@Deprecated` 标记，**新代码不应再用**。

**用途**：在多 Agent 编排中，把"等待人类"建模为 Agent，主 Agent 可以 `call(userAgent)` 来阻塞等用户输入。

## 4. 设计权衡

| 选择 | 原因 |
|---|---|
| `Agent` 三合一接口 | 一个对象能 call / stream / observe，避免创建多个适配器 |
| 状态放 `AgentState` 不放 `Memory` | 状态包含迭代/权限/任务，光是消息列表不够 |
| `call` 是 final 模板 | 子类只关心核心循环，通用能力（事件、状态、Hook）由基类统一管理 |
| `RuntimeContext` 显式传 | 一个 Agent 实例可服务多用户/多会话 |
| 旧 `Memory` 标记 deprecated 但保留 | 不破坏 v1 用户代码迁移路径 |

## 5. 实验任务

详见 [`lab/ch04-first-agent.md`](../lab/ch04-first-agent.md)。核心：

1. 复制 `ScriptedModel` 脚手架
2. 构造一个**不发工具调用**的脚本（直接返回固定文本）
3. 跑通 `ReActAgent.builder()...build().call(...)` 的最小流程
4. 验证第二次 `call` 是否复用 state（消息历史累积）

## 6. 思考题

1. `call()` 是 final 模板方法 —— 如果你的业务需要"在工具调用后插入一段清理逻辑"，应该改哪里？
2. `observe()` 和 `call()` 共享同一个 state 吗？为什么？
3. 假设 `RuntimeContext` 不传 `userId`，`loadOrCreateState` 会发生什么？

## 7. 参考资料

- 官方文档 building-blocks/agent.md
- `docs/v2/en/docs/building-blocks/agent.md`（约 621 行）
- 模板方法模式：<https://refactoring.guru/design-patterns/template-method>
- 反应式背压：<https://projectreactor.io/docs/core/release/reference/#backpressure>

## 8. 学习笔记

在 `notes/ch04-my-takeaways.md` 写 3-5 条金句。

---

> 上一章：[Ch03](./ch03-message-and-block.md) · 下一章：[Ch05](./ch05-react-loop-deep-dive.md)
