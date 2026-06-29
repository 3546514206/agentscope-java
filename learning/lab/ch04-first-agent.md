# Ch04 · 实验：第一个 Agent（基于 ScriptedModel）

> 配套章节：[ch04-agent-and-state.md](../chapters/ch04-agent-and-state.md)
> 预计时长：40 分钟
> 前置：Ch01-Ch03 完成

## 目标

1. 复制 `ScriptedModel` 脚手架并能复述其结构
2. 跑通 `ReActAgent.call()` 最小流程
3. 验证 state 在多次 `call` 间累积
4. 第一次直观感受 Hook 事件流

## 实验 1：复制 ScriptedModel

创建 `/tmp/agent-lab/src/main/java/lab/FirstAgent.java`：

```java
package lab;

import io.agentscope.core.ReActAgent;
import io.agentscope.core.agent.RuntimeContext;
import io.agentscope.core.message.Msg;
import io.agentscope.core.message.MsgRole;
import io.agentscope.core.message.TextBlock;
import io.agentscope.core.message.ContentBlock;
import io.agentscope.core.model.*;
import reactor.core.publisher.Flux;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Supplier;

public class FirstAgent {

    // 1. ScriptedModel：脚本驱动的假模型
    public static class ScriptedModel extends ChatModelBase {
        private final List<Supplier<Flux<ChatResponse>>> scripts;
        private final AtomicInteger idx = new AtomicInteger(0);

        public ScriptedModel(List<Supplier<Flux<ChatResponse>>> scripts) {
            this.scripts = scripts;
        }

        @Override public String getModelName() { return "scripted"; }

        @Override
        protected Flux<ChatResponse> doStream(
                List<Msg> messages, List<ToolSchema> tools, GenerateOptions options) {
            int i = idx.getAndIncrement();
            if (i >= scripts.size()) {
                return Flux.just(textResponse("[脚本耗尽]"));
            }
            return scripts.get(i).get();
        }

        private static ChatResponse textResponse(String text) {
            return ChatResponse.builder()
                .content(List.<ContentBlock>of(
                    TextBlock.builder().text(text).build()))
                .build();
        }
    }

    public static void main(String[] args) {
        // 2. 准备脚本：模型永远返回"你好！"（不发工具调用）
        ScriptedModel model = new ScriptedModel(List.of(
            () -> Flux.just(ScriptedModel.textResponse("你好！我是 scripted 模型。"))
        ));

        // 3. 构造最小 Agent（无工具）
        ReActAgent agent = ReActAgent.builder()
            .name("first-agent")
            .sysPrompt("你是一个测试 agent。")
            .model(model)
            .build();

        // 4. 第一次调用
        RuntimeContext ctx = RuntimeContext.builder()
            .sessionId("demo")
            .userId("alice")
            .build();

        Msg userMsg = Msg.builder()
            .role(MsgRole.USER)
            .textContent("hi")
            .build();

        Msg resp1 = agent.call(userMsg, ctx).block();
        System.out.println("[turn 1] " + resp1.getTextContent());

        // 5. 第二次调用：state 应累积，包含"hi" + "你好！"
        Msg resp2 = agent.call(
            Msg.builder().role(MsgRole.USER).textContent("再聊").build(),
            ctx
        ).block();
        System.out.println("[turn 2] " + resp2.getTextContent());

        // 6. 打印 state 验证
        var state = agent.getAgentState();
        System.out.println("\n--- state.context (size=" + state.getContext().size() + ") ---");
        for (int i = 0; i < state.getContext().size(); i++) {
            Msg m = state.getContext().get(i);
            System.out.println("  [" + i + "] " + m.getRole() + ": " + m.getTextContent());
        }
    }
}
```

`pom.xml`（参考 ch03 模板）：

```xml
<dependencies>
    <dependency>
        <groupId>io.agentscope</groupId>
        <artifactId>agentscope-core</artifactId>
        <version>1.0.12</version>
    </dependency>
</dependencies>
```

```bash
cd /tmp/agent-lab
mvn -q compile exec:java -Dexec.mainClass=lab.FirstAgent
```

**预期输出**：

```
[turn 1] 你好！我是 scripted 模型。
[turn 2] 你好！我是 scripted 模型。
--- state.context (size=4) ---
  [0] USER: hi
  [1] ASSISTANT: 你好！我是 scripted 模型。
  [2] USER: 再聊
  [3] ASSISTANT: 你好！我是 scripted 模型。
```

**观察**：

- ✅ 同一 `RuntimeContext`（同 user/session）累积了 4 条消息
- ✅ 模型只被调用 2 次（每次 call 一次）
- ✅ `agent.getAgentState()` 拿到的是当前 state 视图

## 实验 2：观察 Hook 事件

`Hook` 系统会在 ReAct 关键点触发。改写 `FirstAgent.java`，加一个 `onEventReceived` Hook：

```java
import io.agentscope.core.hook.*;
import io.agentscope.core.event.*;

// ... 在 main 中
ReActAgent agent = ReActAgent.builder()
    .name("first-agent")
    .sysPrompt("...")
    .model(model)
    .hook(new Hook() {
        @Override
        public String getName() { return "log-everything"; }

        // 正确签名：<T extends HookEvent> Mono<T> onEvent(T event) —— 1 个泛型参数
        // 之前报告里写的 onEvent(AgentEvent event, HookEventType type) 不能编译
        @Override
        public <T extends HookEvent> Mono<T> onEvent(T event) {
            System.out.println("[hook] " + event.getClass().getSimpleName());
            return Mono.just(event);
        }
    })
    .build();
```

**注意**：`Hook.onEvent` 是**单参数泛型签名**（`Hook.java:152`），不是双参数 `(AgentEvent, HookEventType)`。事件类型通过 `switch` 模式匹配（Java 21+）或 `instanceof` 判断。

## 实验 3：尝试不同 RuntimeContext 隔离

```java
// ctx1
RuntimeContext ctx1 = RuntimeContext.builder().sessionId("a").userId("u1").build();
agent.call(Msg.builder().role(MsgRole.USER).textContent("ctx1 msg").build(), ctx1).block();

// ctx2
RuntimeContext ctx2 = RuntimeContext.builder().sessionId("b").userId("u1").build();
agent.call(Msg.builder().role(MsgRole.USER).textContent("ctx2 msg").build(), ctx2).block();

// 打印 agent.getAgentState() —— 但注意 getAgentState() 返回的是默认 state，
// 多 context 时需要用 agent.getState(ctx) 或类似接口。
```

> **重要发现**：`getAgentState()` 默认返回 null 或者是按某一 key 的 state。多 context 时需要用对应 API。读 `AgentBase.java:97` 验证。

## 实验 4：interrupt 测试（可选）

```java
// 在 agent.call 前另起线程调用
new Thread(() -> {
    try { Thread.sleep(100); } catch (InterruptedException ignored) {}
    agent.interrupt(Msg.builder()
        .role(MsgRole.USER)
        .textContent("【用户打断】")
        .build());
}).start();

agent.call(longRunningMsg, ctx).block();
```

观察：

- `agent.interrupt(msg)` 是协作式 —— 它设置一个标志
- 当前 ReAct 循环在检查点（`checkInterrupted()`）会响应
- 不会强制 kill 线程

## 验收标准

- [ ] 你能在 30 秒内解释 `ScriptedModel` 的设计目的
- [ ] 你能跑通最小 ReActAgent 并验证 state 累积
- [ ] 你能在 30 秒内说出 `call()` vs `streamEvents()` vs `observe()` 的区别
- [ ] 你能用 `RuntimeContext` 实现多用户隔离

## 思考题答案提示

1. **清理逻辑插入点**：在 `ReActAgent.callInternal` 中加 Hook（`PostActingEvent` 后），不修改 `callInternal` 本身
2. **observe vs call 共享 state**：是的，共享 —— 同一个 `loadOrCreateState(key)` 取的是同一个对象
3. **userId 为空**：`stateStore.load(null, ...)` 通常返回 empty，新建一个 userId=null 的 state（生产中会出 bug，要校验）

## 下一步

完成 Ch04 后你已经能跑通『用户 → Agent → 假模型 → 回复』最小回路。Ch05 进入**核心**：ReAct 主循环源码精读。
