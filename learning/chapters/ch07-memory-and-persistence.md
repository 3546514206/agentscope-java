# Ch07 · 持久化（v2 现行范式）+ 旧 v1 长期记忆（已 deprecated）

> 状态：🔲 · 预计时长：2.5h · 前置：Ch06

## ⚠️ 重要：本章范围在 v2 起发生重大变更

**`LongTermMemory` 接口在 v2 起已被声明 `@Deprecated(forRemoval = true, since = "2.0.0")`**（见 `agentscope-core/src/main/java/io/agentscope/core/memory/LongTermMemory.java` 顶部注释）：

> "Long-term memory is removed; conversation context now lives on `AgentState#getContext()`. Integrate any cross-session persistence at the application layer."

也就是说：

- v2 起**框架级长期记忆抽象已废弃**，未来版本将彻底删除
- 跨会话持久化**由应用层自己实现**（用 `AgentStateStore` + `StateBackedMemory` 之类）
- 本章按"持久化（v2 现行）→ 旧 v1 长期记忆（已 deprecated）"两部分组织

## 1. 本章目标

- 理解 `AgentStateStore` 抽象与两个内置实现（`InMemoryAgentStateStore` / `JsonFileAgentStateStore`）
- 理解 v1 `Memory` → v2 `AgentState` 的迁移路径
- ~~掌握 `LongTermMemory` 与 `LongTermMemoryMode` 三种模式~~ **（已 deprecated，仅作历史背景）**
- 掌握 `StateBackedMemory` 适配器（旧接口兼容）

## 2. 核心概念

### 2.1 三层记忆体系

```mermaid
graph TB
    A[短期消息历史<br/>state.context] --> B[AgentState<br/>不可变聚合]
    C[长期事实<br/>LongTermMemory] --> D[语义检索]
    B --> E[AgentStateStore<br/>持久化]
    D --> E
    E --> F[(Redis/JSON File/DB)]
```

| 层 | 抽象 | 存储位置 |
|---|---|---|
| 短期 | `AgentState.context: List<Msg>` | `AgentStateStore` |
| 长期 | `LongTermMemory` | 独立存储 + 语义索引 |
| 工作 | `AgentState` 其他字段 | `AgentStateStore` |

### 2.2 `AgentStateStore` 接口

`agentscope-core/src/main/java/io/agentscope/core/state/AgentStateStore.java:61`：

```java
public interface AgentStateStore {
    // 加载单个 state
    <T extends State> Mono<T> load(String userId, String sessionId, String key, Class<T> type);

    // 加载某个 session 的所有 state
    <T extends State> Flux<T> loadAll(String userId, String sessionId, String key, Class<T> type);

    // 持久化单个
    Mono<Void> save(String userId, String sessionId, String key, State value);

    // 持久化多个
    Mono<Void> save(String userId, String sessionId, String key, List<? extends State> values);
}
```

**设计要点**：

- 4 个方法的 `Mono` / `Flux` 返回 —— **异步**，适配 Redis / DB
- 键是 `(userId, sessionId, key)` 三元组
- `State` 是一个标记接口，`AgentState` 实现它

### 2.3 内置实现

#### `InMemoryAgentStateStore`

`state/InMemoryAgentStateStore.java`：

- 内部用**嵌套** `ConcurrentHashMap<String, Map<String, SessionData>>`（不是简单的 `ConcurrentHashMap<String, State>`）
  - 外层 key：`userId`
  - 内层 value：`SessionData`，含 `Map<String, State> singleStates` 和 `Map<String, List<State>> listStates`
- 纯内存，进程重启**丢失**
- 适合单元测试

#### `JsonFileAgentStateStore`

`state/JsonFileAgentStateStore.java`：

- 写到本地 JSON 文件，默认路径 `~/.agentscope/state/<agentId>/<userId>/<sessionId>/`
- 用 Jackson 序列化整个 `AgentState`
- 单机部署可用

**默认路径结构**（**注意：没有 `<agentId>` 段**）：

```
~/.agentscope/state/
└── <userId>/
    └── <sessionId>/
        └── <key>.json
```

**关键纠正（与之前报告相比）**：
- 路径只有 `<userId>/<sessionId>/<key>.json` 三段，**没有 `<agentId>` 段**
- `<key>` 对应 `save(userId, sessionId, key, value)` 方法的 `key` 参数（默认是 `"agent_state"`）

**为什么没有 `<agentId>`**：`JsonFileAgentStateStore` 不知道也不关心"哪个 agent 在用"——它只按 `(userId, sessionId, key)` 三个维度组织数据。如果想按 agent 隔离，由 `key` 命名空间（`my_agent:agent_state`）自己处理。

#### `RedisAgentStateStore`（extensions-redis）

`agentscope-extensions/agentscope-extensions-redis/`：

- 分布式场景
- Key 设计同上

### 2.4 v1 `Memory` 迁移

```java
// v1
Memory mem = new InMemoryMemory();
mem.saveTo(stateStore, "alice", "demo");
mem.loadFrom(stateStore, "alice", "demo");
List<Msg> msgs = mem.getMessages();

// v2 推荐
AgentStateStore store = new JsonFileAgentStateStore();
Mono<AgentState> state = store.load("alice", "demo", "agent_state", AgentState.class);
```

`StateBackedMemory` 是过渡期的**桥接类**：

```java
public class StateBackedMemory implements Memory {
    private final AgentStateStore store;
    private final String userId, sessionId;

    public List<Msg> getMessages() {
        // 实际从 store.load(...) 取
    }
}
```

### 2.5 ⚠️ 旧 v1 `LongTermMemory` 长期记忆（**已 deprecated**）

> **本节仅作历史背景**——v2 起新代码不应再用此 API。

`memory/LongTermMemory.java:71`（接口**本身**）：

```java
@Deprecated(forRemoval = true, since = "2.0.0")
public interface LongTermMemory {
    Mono<Void> record(List<Msg> msgs);     // 记录一条或多条事实（接收 List，不是单条）
    Mono<String> retrieve(Msg msg);         // 语义检索（入参是 Msg，输出 String）
    // 注意：没有 clear() 方法 —— 如需清空需通过 record 覆盖或扩展实现
}
```

**关键纠正**（与之前报告相比）：
- `record` 接 `List<Msg>`，不是 `Msg`
- `retrieve` 接 `Msg` 返回 `Mono<String>`，不是 `Flux<Knowledge>` 也不是 `(String, int)`
- **没有 `clear()` 方法**
- **整个接口 `@Deprecated forRemoval`** —— 见接口顶部的 Javadoc

**实现位置（历史，已不推荐）**：

- `agentscope-extensions/agentscope-extensions-mem/agentscope-extensions-mem0/`
- `agentscope-extensions/agentscope-extensions-mem/agentscope-extensions-memory-bailian/`
- `agentscope-extensions/agentscope-extensions-mem/agentscope-extensions-reme/`

**对应 `Builder.longTermMemory(LongTermMemory)` 和 `Builder.longTermMemoryMode(LongTermMemoryMode)`**（`ReActAgent.java:3929`, `:3938`）**也都标注 `@Deprecated since 2.0.0`**。

### 2.6 ⚠️ 旧 v1 `LongTermMemoryMode` 三种模式（**已 deprecated**）

> **本节仅作历史背景**——v2 起新代码不应再用此 API。

`memory/LongTermMemoryMode.java:54`（**枚举类声明**）：

```java
// L54 是 enum 声明
public enum LongTermMemoryMode {
    AGENT_CONTROL,   // L59
    STATIC_CONTROL,  // L64
    BOTH             // L69
}
```

**关键纠正**：
- L54 是 `enum` 类声明行，**实际枚举常量在 L59 / L64 / L69**
- 整个枚举配套 `LongTermMemory` 接口**一起进入 deprecated 通道**

| 模式 | Agent 能调用工具吗 | 框架自动记录吗 | 适用 | 状态 |
|---|---|---|---|---|
| `AGENT_CONTROL` | ✅ | ❌ | 高级 Agent，模型理解何时该记 | **deprecated** |
| `STATIC_CONTROL` | ❌ | ✅ | 简单场景，强制全量记录 | **deprecated** |
| `BOTH` | ✅ | ✅ | 旧版"推荐默认" | **deprecated** |

**v2 推荐的迁移路径**：跨会话持久化用 `AgentStateStore`（`JsonFileAgentStateStore` / `RedisAgentStateStore` / `MySQLAgentStateStore` 等等），按 `(userId, sessionId, key)` 维度组织数据；语义检索需求用 RAG 扩展（`agentscope-extensions-rag`）替代。

## 3. 源码精读

### 3.1 `JsonFileAgentStateStore` 的文件布局

读 `state/JsonFileAgentStateStore.java`（实际 396 行）：

```java
// 关键：save 是同步 void 方法，不是 Mono.fromCallable
public void save(String userId, String sessionId, String key, State value) {
    Path dir = resolveDir(userId, sessionId, key);  // ~/.agentscope/state/<userId>/<sessionId>/
    Files.createDirectories(dir);
    Path file = dir.resolve(key + ".json");

    // 写时复制：写到临时文件，原子重命名
    Path tmp = file.resolveSibling(file.getFileName() + ".tmp");
    Files.writeString(tmp, JsonCodec.encode(value));
    Files.move(tmp, file, StandardCopyOption.ATOMIC_MOVE, StandardCopyOption.REPLACE_EXISTING);
}
```

**关键纠正**（与之前报告相比）：
- **`save` 是同步 void 方法**（L99 和 L111），**没有 Reactor** 包裹
- `Mono.fromCallable + subscribeOn(boundedElastic)` 只在 `clearAllSessions()`（L251-280）使用
- 默认路径 **`~/.agentscope/state/<userId>/<sessionId>/`** —— **没有 `<agentId>` 段**

**观察 1**：写时复制 + 原子重命名 —— 防止写到一半崩溃导致损坏。

**观察 2**：调用方负责在合适线程调用（通常是 `Schedulers.boundedElastic().schedule(() -> store.save(...))`）。

### 3.2 `AgentState` 序列化

`AgentState` 实现 `State` 接口，依赖 Jackson 注解（`@JsonProperty`）。读 `state/AgentState.java:100-200` 看字段。

**关键字段**：

- `context: List<Msg>` —— 消息历史
- `summary: String` —— 长期对话压缩后的摘要
- `curIter: int` —— 当前迭代次数（用于恢复执行）
- `permissionContext: PermissionContextState`
- `toolContext: ToolContextState`
- `tasksContext: TaskContextState`

### 3.3 ⚠️ 旧 v1 `StaticLongTermMemoryHook` 自动记录（**已 deprecated**）

`memory/StaticLongTermMemoryHook.java`（实际 300 行，已 `@Deprecated(forRemoval = true, since = "2.0.0")`）：

```java
@Deprecated(forRemoval = true, since = "2.0.0")
public class StaticLongTermMemoryHook implements Hook {
    private final LongTermMemory memory;
    private final boolean asyncRecord;
    // 注意：没有 counter 字段，没有"每 N 条"逻辑

    @Override
    public Mono<HookEvent> handlePostCall(PostCallEvent event) {
        // 每个 PostCallEvent 都**无条件**触发记录，没有节流
        if (asyncRecord) {
            return memory.record(currentMsgs)
                .subscribeOn(Schedulers.newBoundedElastic(1, 3, "long-term-memory-record"))
                .then(Mono.just(event));
        }
        return memory.record(currentMsgs).then(Mono.just(event));
    }
}
```

**关键纠正**（与之前报告相比）：
- **没有 `counter` 字段**，没有"每 10 条"节流逻辑。
- **每个 `PostCallEvent` 都无条件调用 `memory.record(...)`**。
- 异步模式用 `Schedulers.newBoundedElastic(1, 3, "long-term-memory-record")`（独占有界线程池，防止内存爆）。
- 该类已被 `@Deprecated forRemoval` 标注 —— **新代码不要用**。

**观察**：通过 Hook 实现『框架自动记录』，业务侧零侵入。但因为全量无节流，长期运行场景需要业务侧自行节流或改用 `AGENT_CONTROL` 模式。

## 4. 设计权衡

| 选择 | 原因 |
|---|---|
| v1 Memory 标记 deprecated 但保留 | 不破坏 v1 用户 |
| `State` 接口做标记 | 多态序列化 |
| 文件路径默认 `~/.agentscope/state/...` | 跨平台、隔离 workspace |
| 写时复制 + 原子重命名 | 防止崩溃损坏 |
| ~~LongTermMemory 三种模式~~ | ~~灵活度 vs 自动化~~ **（v2 起整个机制 deprecated）** |
| 持久化放 store 不放 Agent | Agent 是无状态的，可水平扩展 |

## 5. 实验任务

详见 [`lab/ch07-persistence-and-long-term-memory.md`](../lab/ch07-persistence-and-long-term-memory.md)。核心：

1. 跑两次 `agent.call(...)` 同一个 sessionId，验证 state 自动恢复
2. 杀掉进程重启，再次 `call`，验证 state 从 JSON 文件加载
3. 注册一个 `StaticLongTermMemoryHook`，跑 10 轮看自动记录

## 6. 思考题

1. 如果用 `JsonFileAgentStateStore` 部署在多实例上，会出现什么问题？
2. `AgentState` 里有 `permissionContext`，如果用户 A 的权限设置会污染用户 B 吗？
3. 长期记忆和短期记忆有重复时，如何去重？

## 7. 参考资料

- `docs/v2/en/docs/building-blocks/context.md`（约 296 行）
- `docs/v2/en/docs/harness/memory.md`（约 265 行）
- Jackson 序列化：<https://github.com/FasterXML/jackson-docs>

## 8. 学习笔记

在 `notes/ch07-my-takeaways.md` 写 3-5 条金句。

---

> 上一章：[Ch06](./ch06-toolkit-and-function-calling.md) · 下一章：[Ch08](./ch08-middleware-and-hooks.md)
