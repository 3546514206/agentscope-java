# Ch10 · 实验：多 Agent 协作

> 配套章节：[ch10-multi-agent-and-harness.md](../chapters/ch10-multi-agent-and-harness.md)
> 预计时长：60 分钟
> 前置：Ch09 完成

## 目标

1. 构造两个独立 ReActAgent：researcher + writer
2. 用 `SubAgentConfig` 把 researcher 暴露为主 Agent 的工具
3. 主 Agent 通过 `call_researcher` 调子 Agent
4. 验证 `conversation_id` 实现多轮

## 实验 1：构造两个子 Agent

`/tmp/agent-lab/src/main/java/lab/SubAgentDemo.java`：

```java
package lab;

import io.agentscope.core.ReActAgent;
import io.agentscope.core.agent.RuntimeContext;
import io.agentscope.core.message.*;
import io.agentscope.core.model.*;
import io.agentscope.core.tool.*;
import io.agentscope.core.tool.subagent.SubAgentConfig;
import reactor.core.publisher.Flux;
import java.util.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Supplier;

public class SubAgentDemo {

    public static class ScriptedModel extends ChatModelBase {
        private final List<Supplier<Flux<ChatResponse>>> scripts;
        private final AtomicInteger idx = new AtomicInteger(0);
        public ScriptedModel(String role, List<Supplier<Flux<ChatResponse>>> s) {
            super();
            this.scripts = s;
        }
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

    public static void main(String[] args) {
        // ============= 子 Agent: researcher =============
        ScriptedModel researcherModel = new ScriptedModel("r", List.of(
            () -> Flux.just(ScriptedModel.textResponse("研究结果：ReAct 是一种推理范式。")),
            () -> Flux.just(ScriptedModel.textResponse("补充：ReAct 由 Yao et al. 2022 提出。"))
        ));

        ReActAgent researcher = ReActAgent.builder()
            .name("Researcher")
            .description("负责检索资料")
            .sysPrompt("你是研究员，给出事实性回答。")
            .model(researcherModel)
            .build();

        // ============= 子 Agent: writer =============
        ScriptedModel writerModel = new ScriptedModel("w", List.of(
            () -> Flux.just(ScriptedModel.textResponse("文章草稿：ReAct 推理范式。"))
        ));

        ReActAgent writer = ReActAgent.builder()
            .name("Writer")
            .description("负责写作")
            .sysPrompt("你是写作者，根据材料写文章。")
            .model(writerModel)
            .build();

        // ============= 主 Agent =============
        ScriptedModel mainModel = new ScriptedModel("m", List.of(
            // 主 Agent 第一轮：调 researcher
            () -> Flux.just(ScriptedModel.textResponse(
                "我已经收集到材料，现在写文章。")),
            // 主 Agent 第二轮：调 writer
            () -> Flux.just(ScriptedModel.textResponse("文章完成。"))
        ));

        // 把 researcher 注册为主 Agent 的工具
        Toolkit mainTk = new Toolkit();
        mainTk.registerSubAgent(SubAgentConfig.builder()
            .name("researcher")
            .agent(researcher)
            .build());

        ReActAgent mainAgent = ReActAgent.builder()
            .name("MainAgent")
            .sysPrompt("你协调研究员和写作者。")
            .model(mainModel)
            .toolkit(mainTk)
            .build();

        // ============= 跑一次 =============
        RuntimeContext ctx = RuntimeContext.builder()
            .sessionId("orchestration").userId("u1").build();

        Msg resp = mainAgent.call(
            Msg.builder().role(MsgRole.USER)
                .textContent("写一篇关于 ReAct 的短文").build(),
            ctx
        ).block();

        System.out.println("\n[final] " + resp.getTextContent());
    }
}
```

```bash
mvn -q compile exec:java -Dexec.mainClass=lab.SubAgentDemo
```

**观察**：

- 主 Agent 看到工具列表里有 `call_researcher`
- 它通过 `tool_use` 触发子 Agent
- 子 Agent 独立 ReAct 循环，返回结果
- 主 Agent 收到结果继续推理

## 实验 2：conversation_id 多轮

修改子 Agent 的脚本，让它"记住"之前的话：

```java
ScriptedModel statefulModel = new ScriptedModel("r", List.of(
    () -> Flux.just(ScriptedModel.textResponse("我记住了第一句。"))
    // 同 sessionId 第二次调用会继续累积 state
));
```

主 Agent 两次调 `call_researcher`（同 `conversation_id`）：

```java
// 第一次：conversation_id="c1"
// 第二次：conversation_id="c1"（同会话）
// → researcher 看到自己上一轮输出
```

## 实验 3：打印所有事件观察嵌套

```java
mainAgent.streamEvents(userMsg, ctx)
    .doOnNext(e -> System.out.println("[event] " + e.getClass().getSimpleName()))
    .blockLast();
```

**预期**：

- 主 Agent 的 Pre/Post Call 事件
- 主 Agent 的 Pre/Post Reasoning
- 子 Agent 调用的 ToolCallStart/End（**子 Agent 的事件被 `forwardEvents=true` 转发**）
- 主 Agent 的 Post Acting
- 主 Agent 的 Pre/Post Reasoning（第二轮）
- ...

## 实验 4：HarnessAgent 最小实例（可选）

`HarnessAgent` 需要 workspace 目录。先建：

```bash
mkdir -p /tmp/harness-ws/agents/main-agent
echo "# AGENTS" > /tmp/harness-ws/AGENTS.md
```

```java
// 引入 harness 依赖
HarnessAgent harness = HarnessAgent.builder()
    .name("main-agent")
    .model("dashscope:qwen-plus")  // 需要 API Key
    .workspace(Path.of("/tmp/harness-ws"))
    .build();
```

> 完整 HarnessAgent 实验需要真实 LLM，Ch12 后做完整实践。

## 验收标准

- [ ] 你能在 30 秒内解释子 Agent = 工具的设计
- [ ] 你能用 `SubAgentConfig` 注册子 Agent
- [ ] 你能让主 Agent 触发子 Agent 调用
- [ ] 你能用 `conversation_id` 实现多轮
- [ ] 你能区分 `forwardEvents=true/false` 的效果

## 思考题答案提示

1. **嵌套调用**：框架通过 `SubagentEventBus` 隔离事件流，**不会**无限递归（除非逻辑上确实无限）
2. **同一 LLM**：通常各 Agent 独立配置；研究/写作类可用更大模型，工具类可用小模型降本
3. **forwardEvents 影响**：调试期 true（看到完整链路），生产期 false（降噪 + 性能）

## 下一步

完成 Ch10 后你已经能构建多 Agent 系统。Ch11 进入**协议集成**（MCP / A2A）。
