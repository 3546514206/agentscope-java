# Ch07 · 实验：持久化与长期记忆

> 配套章节：[ch07-memory-and-persistence.md](../chapters/ch07-memory-and-persistence.md)
> 预计时长：45 分钟
> 前置：Ch06 完成

## 目标

1. 用 `JsonFileAgentStateStore` 实现 state 落盘
2. 验证进程重启后 state 恢复
3. 跑通 `LongTermMemoryMode.BOTH` 模式
4. 查看磁盘上 JSON 文件结构

## 实验 1：基本持久化

`/tmp/agent-lab/src/main/java/lab/PersistenceDemo.java`：

```java
package lab;

import io.agentscope.core.ReActAgent;
import io.agentscope.core.agent.RuntimeContext;
import io.agentscope.core.message.*;
import io.agentscope.core.model.*;
import io.agentscope.core.state.AgentStateStore;
import io.agentscope.core.state.JsonFileAgentStateStore;
import reactor.core.publisher.Flux;
import java.nio.file.Path;
import java.util.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Supplier;

public class PersistenceDemo {

    public static class ScriptedModel extends ChatModelBase {
        private final List<Supplier<Flux<ChatResponse>>> scripts;
        private final AtomicInteger idx = new AtomicInteger(0);
        public ScriptedModel(List<Supplier<Flux<ChatResponse>>> s) { this.scripts = s; }
        @Override public String getModelName() { return "scripted"; }
        @Override
        protected Flux<ChatResponse> doStream(
                List<Msg> messages, List<ToolSchema> tools, GenerateOptions options) {
            int i = idx.getAndIncrement();
            if (i >= scripts.size()) return Flux.just(textResponse("[end]"));
            return scripts.get(i).get();
        }
        public static ChatResponse textResponse(String t) {
            return ChatResponse.builder()
                .content(List.<ContentBlock>of(TextBlock.builder().text(t).build())).build();
        }
    }

    public static void main(String[] args) throws Exception {
        // 1. 显式配置 store（不配置则用 InMemoryAgentStateStore 默认）
        AgentStateStore store = new JsonFileAgentStateStore(Path.of("/tmp/agentscope-state"));

        ReActAgent agent = ReActAgent.builder()
            .name("persistence-demo")
            .sysPrompt("...")
            .model(new ScriptedModel(List.of(
                () -> Flux.just(ScriptedModel.textResponse("round 1 done"))
            )))
            .stateStore(store)
            .build();

        RuntimeContext ctx = RuntimeContext.builder()
            .sessionId("persist-demo")
            .userId("alice")
            .build();

        // 2. 第一次调用
        agent.call(Msg.builder().role(MsgRole.USER)
            .textContent("first turn").build(), ctx).block();

        // 3. 打印落盘文件
        Path stateFile = Path.of("/tmp/agentscope-state/persistence-demo/alice/persist-demo/agent_state.json");
        System.out.println("[file exists] " + java.nio.file.Files.exists(stateFile));
        System.out.println("[file content] " + java.nio.file.Files.readString(stateFile));
    }
}
```

```bash
mvn -q compile exec:java -Dexec.mainClass=lab.PersistenceDemo
cat /tmp/agentscope-state/persistence-demo/alice/persist-demo/agent_state.json | head -50
```

**观察**：

- `agent_state.json` 是标准 JSON，可读
- 包含 `context: [...]`、`sessionId`、`userId` 等

## 实验 2：进程重启后恢复

```java
// 注释掉第一次调用，模拟"新进程"
// 1. agent.call(...)  // 假设之前已经 call 过
// 2. 模拟"新进程"
ReActAgent newAgent = ReActAgent.builder()
    .name("persistence-demo")  // 同名 agent
    .sysPrompt("...")
    .model(new ScriptedModel(List.of(
        () -> Flux.just(ScriptedModel.textResponse("round 2 done"))
    )))
    .stateStore(store)         // 同 store
    .build();

// 第二次调用应该看到完整历史
newAgent.call(Msg.builder().role(MsgRole.USER)
    .textContent("second turn").build(), ctx).block();

// 打印 state 验证
var state = newAgent.getAgentState();
System.out.println("[recovered state size] " + state.getContext().size());
```

**预期**：`state.context` 包含 4 条消息（USER/ASSISTANT/USER/ASSISTANT）。

## 实验 3：LongTermMemory BOTH 模式

`/tmp/agent-lab/src/main/java/lab/LongTermDemo.java`：

```java
package lab;

import io.agentscope.core.memory.*;
// 注：完整 LongTermMemory 实现位于 agentscope-extensions-mem 模块
// 本实验只演示 Hook 注册逻辑

public class LongTermDemo {
    public static void main(String[] args) {
        // 1. 框架自带：StaticLongTermMemoryHook（STATIC_CONTROL）
        //    通过 .staticLongTermMemoryHook(...) 注册
        // 2. 框架自带：LongTermMemoryTools（AGENT_CONTROL）
        //    通过 .longTermMemoryTools(...) 注册，让 Agent 能调工具
        // 3. 组合：BOTH
        //
        // 详见：ReActAgent.builder() 的 longTermMemory() / longTermMemoryMode() 方法

        // 自定义：实现一个简单的 InMemoryLongTermMemory
        LongTermMemory inMem = new LongTermMemory() {
            private final List<String> records = new ArrayList<>();
            @Override
            public Mono<Void> record(Msg message) {
                records.add(message.getTextContent());
                return Mono.empty();
            }
            @Override
            public Flux<Knowledge> retrieve(String query, int limit) {
                return Flux.fromIterable(records.stream()
                    .filter(r -> r.contains(query))
                    .map(r -> new Knowledge(r, 1.0))
                    .limit(limit)
                    .toList());
            }
            @Override
            public Mono<Void> clear() {
                records.clear();
                return Mono.empty();
            }
        };

        // 接入 Agent
        // ReActAgent.builder()
        //     .longTermMemory(inMem)
        //     .longTermMemoryMode(LongTermMemoryMode.BOTH)
        //     .build();
    }
}
```

> 完整能跑的 LongTermMemory 实验需引入 `agentscope-extensions-mem` 模块。本实验演示接口契约。

## 实验 4：查看 JSON 结构

```bash
# 树形打印
find /tmp/agentscope-state -type f -name "*.json" | xargs -I {} sh -c 'echo "=== {} ==="; cat {}; echo'
```

**预期结构**：

```json
{
  "sessionId": "persist-demo",
  "userId": "alice",
  "summary": null,
  "curIter": 0,
  "context": [
    {"id": "...", "role": "USER", "content": [...]},
    {"id": "...", "role": "ASSISTANT", "content": [...]}
  ],
  "shutdownInterrupted": false,
  "permissionContext": {...},
  "toolContext": {...},
  "tasksContext": {...}
}
```

## 验收标准

- [ ] 你能在 30 秒内解释 `JsonFileAgentStateStore` 的文件布局
- [ ] 你能跑通"两次 call 跨进程"并看到 state 恢复
- [ ] 你能区分 `STATIC_CONTROL` / `AGENT_CONTROL` / `BOTH` 三种模式
- [ ] 你能读懂 `agent_state.json` 的字段含义

## 思考题答案提示

1. **多实例**：会读写冲突，需切换到 `RedisAgentStateStore`（用 Redis 原子写）
2. **权限污染**：`userId` 在 store key 中隔离，不会污染
3. **去重**：通常靠 embedding 相似度阈值 + 时间衰减

## 下一步

完成 Ch07 后你已掌握 Agent 的『记忆系统』全貌。Ch08 进入**中间件与 Hook** —— 框架的扩展点。
