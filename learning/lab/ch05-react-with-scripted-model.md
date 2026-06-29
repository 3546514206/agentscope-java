# Ch05 · 实验：用 ScriptedModel 跑通两轮 ReAct

> 配套章节：[ch05-react-loop-deep-dive.md](../chapters/ch05-react-loop-deep-dive.md)
> 预计时长：60 分钟
> 前置：Ch01-Ch04 完成

## 目标

1. 看到完整 ReAct：reasoning → acting → reasoning → 文本 → summarizing
2. 打印所有 `AgentEvent`，验证 Ch05 §3.5 的事件时序
3. 验证 pending tool calls 机制
4. 理解 `Accumulator` 在流式输出中的作用

## 实验 1：两轮 ScriptedModel + 简单 Tool

`/tmp/agent-lab/src/main/java/lab/ReactLoop.java`：

```java
package lab;

import io.agentscope.core.ReActAgent;
import io.agentscope.core.agent.RuntimeContext;
import io.agentscope.core.message.*;
import io.agentscope.core.model.*;
import io.agentscope.core.tool.*;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

import java.util.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Supplier;

public class ReactLoop {

    // ============ ScriptedModel ============
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
            System.out.println("[model] call #" + i + " (msgs=" + messages.size() + ", tools=" + tools.size() + ")");
            if (i >= scripts.size()) {
                return Flux.just(textResponse("[no more scripts]"));
            }
            return scripts.get(i).get();
        }

        public static ChatResponse textResponse(String text) {
            return ChatResponse.builder()
                .content(List.<ContentBlock>of(TextBlock.builder().text(text).build()))
                .build();
        }

        public static ChatResponse toolUseResponse(String id, String name, Map<String,Object> input) {
            return ChatResponse.builder()
                .content(List.<ContentBlock>of(
                    ToolUseBlock.builder().id(id).name(name).input(input).build()))
                .build();
        }
    }

    // ============ EchoTool ============
    public static class EchoTool extends ToolBase {
        public EchoTool() {
            // 注意：9 参数顺序是 (name, desc, schema, readOnly, concurrencySafe, mcp, mcpName, externalTool, stateInjected)
            // 之前报告里写的 (name, desc, schema, true, true, false, null, false, false) 顺序错了
            // 正确顺序：mcp 必须先于 mcpName；externalTool 在 stateInjected 之前
            super("echo", "echoes back the input",
                Map.of("type", "object",
                       "properties", Map.of("text", Map.of("type", "string")),
                       "required", List.of("text")),
                true,    // readOnly
                true,    // concurrencySafe
                false,   // mcp
                null,    // mcpName
                false,   // externalTool
                false);  // stateInjected
            // 业务上更推荐 Builder 写法：
            // super(ToolBase.builder()
            //     .name("echo").description("echoes back the input")
            //     .inputSchema(schema).readOnly(true).concurrencySafe(true)
            //     .build());
        }

        @Override
        public Mono<ToolResultBlock> callAsync(ToolCallParam param) {
            Object text = param.getInput().get("text");
            System.out.println("[tool]  echo(" + text + ")");
            return Mono.just(ToolResultBlock.text("ECHO:" + text));
        }
    }

    public static void main(String[] args) {
        // 1. 准备脚本：
        //    轮 0：发工具调用 echo("hello")
        //    轮 1：纯文本 "完成。"
        ScriptedModel model = new ScriptedModel(List.of(
            () -> Flux.just(ScriptedModel.toolUseResponse(
                "call_001", "echo", Map.of("text", "hello"))),
            () -> Flux.just(ScriptedModel.textResponse("完成。"))
        ));

        // 2. 构造 Toolkit
        Toolkit toolkit = new Toolkit();
        toolkit.registerAgentTool(new EchoTool());

        // 3. 构造 Agent
        ReActAgent agent = ReActAgent.builder()
            .name("react-loop")
            .sysPrompt("你使用 echo 工具回显。")
            .model(model)
            .toolkit(toolkit)
            .maxIters(5)
            .build();

        // 4. 订阅事件流
        RuntimeContext ctx = RuntimeContext.builder()
            .sessionId("react-demo")
            .userId("alice")
            .build();

        Msg userMsg = Msg.builder()
            .role(MsgRole.USER)
            .textContent("请 echo 一条 hello")
            .build();

        System.out.println("\n========== streamEvents() ==========");
        agent.streamEvents(userMsg, ctx)
            .doOnNext(event -> System.out.println("[event] " + event.getClass().getSimpleName()))
            .blockLast();

        System.out.println("\n========== final state ==========");
        var state = agent.getAgentState();
        for (int i = 0; i < state.getContext().size(); i++) {
            Msg m = state.getContext().get(i);
            System.out.printf("[%d] %s: %s%n",
                i, m.getRole(), abbrev(m.toString()));
        }
    }

    private static String abbrev(String s) {
        return s.length() > 80 ? s.substring(0, 77) + "..." : s;
    }
}
```

```bash
cd /tmp/agent-lab
mvn -q compile exec:java -Dexec.mainClass=lab.ReactLoop
```

**预期事件序列**（部分）：

```
[event] AgentStartEvent
[event] PreCallEvent
[event] PreReasoningEvent
[event] ReasoningEvent
[event] ReasoningChunkEvent
[event] ToolCallStartEvent
[event] ToolCallDeltaEvent (×N)
[event] ToolCallEndEvent
[event] PostReasoningEvent
[event] PreActingEvent
[event] ToolResultStartEvent
[event] ToolResultEndEvent
[event] PostActingEvent
[event] PreReasoningEvent
[event] ReasoningEvent
[event] TextBlockStartEvent
[event] TextBlockDeltaEvent
[event] TextBlockEndEvent
[event] PostReasoningEvent
[event] PostCallEvent
[event] AgentResultEvent
[event] AgentEndEvent
```

**state.context 应该是 4 条**：

```
[0] USER: 请 echo 一条 hello
[1] ASSISTANT: (ToolUseBlock)
[2] TOOL: (ToolResultBlock)
[3] ASSISTANT: 完成。
```

## 实验 2：pending tool calls 恢复

修改脚本，让模型**只发工具调用**，不返回文本：

```java
ScriptedModel model = new ScriptedModel(List.of(
    () -> Flux.just(ScriptedModel.toolUseResponse("c1", "echo", Map.of("text", "first"))),
    () -> Flux.just(ScriptedModel.toolUseResponse("c2", "echo", Map.of("text", "second"))),
    () -> Flux.just(ScriptedModel.textResponse("OK"))
));
```

跑一次后，state 中会看到 assistant 一直发 tool_call，tool 返回 result，直到第三轮才出文本。

## 实验 3：达到 maxIters 走 summarize

设置 `maxIters(2)` + 让模型永远发工具调用：

```java
ReActAgent agent = ReActAgent.builder()
    // ...
    .maxIters(2)
    .build();
```

**观察**：

- iter=0: reasoning → acting → executeIteration(1)
- iter=1: reasoning → acting → executeIteration(2)
- iter=2: `iter >= maxIters`，进入 summarizing
- summarizing 调一次模型，**不传 tools**，强制纯文本输出

## 实验 4：Hook 拦截观察

加一个 Hook 打印所有事件：

```java
.hook(new Hook() {
    @Override public String getName() { return "logger"; }
    @Override
    public Mono<HookEvent> onEvent(AgentEvent event, HookEventType type) {
        System.out.println("[HOOK] " + type + " <- " + event.getClass().getSimpleName());
        return Mono.just(new HookEvent(event, type));
    }
})
```

注意：`onEvent` 的具体签名按 `Hook.java` 调整。

## 验收标准

- [ ] 你能在 30 秒内口头描述 ReAct 循环的 4 个退出条件
- [ ] 你能跑通两轮 ReAct 并打印所有事件
- [ ] 你能解释 `iter+1` 即使纯文本也递增的原因
- [ ] 你能区分 `Pre*Event` / `*Event` / `*ChunkEvent` / `*StartEvent` / `*EndEvent` / `Post*Event`

## 思考题答案提示

1. **多 ToolUseBlock 并行**：默认并行（`Flux.concatMap` 内部用 `boundedElastic`），可通过工具组的 `concurrencySafe=false` 强制串行
2. **Sub-agent 递归调用**：内部用 `SubagentEventBus` 隔离事件流（看 Ch10）
3. **maxIters=0**：第一次 reasoning 就走 summarizing，模型无机会发工具调用

## 下一步

完成 Ch05 后你已经掌握 ReAct 主循环的完整流程。Ch06 进入**工具调用**专题：注解、反射、Schema 生成、权限。
