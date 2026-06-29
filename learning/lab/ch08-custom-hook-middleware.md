# Ch08 · 实验：自定义 Middleware 四件套

> 配套章节：[ch08-middleware-and-hooks.md](../chapters/ch08-middleware-and-hooks.md)
> 预计时长：60 分钟
> 前置：Ch07 完成

## 目标

1. 写 4 个 Middleware：Timing / TokenCounter / ContextSanitizer / SkillInjector
2. 验证洋葱模型的执行顺序
3. 体验 Middleware 的**短路**能力

## 实验 1：Timing Middleware

`/tmp/agent-lab/src/main/java/lab/TimingMiddleware.java`：

```java
package lab;

import io.agentscope.core.ReActAgent;
import io.agentscope.core.agent.Agent;
import io.agentscope.core.agent.RuntimeContext;
import io.agentscope.core.middleware.*;
import io.agentscope.core.event.AgentEvent;
import reactor.core.publisher.Flux;

import java.util.function.Function;

public class TimingMiddleware implements MiddlewareBase {

    @Override
    public Flux<AgentEvent> onAgent(
            Agent agent, RuntimeContext ctx, AgentInput input,
            Function<AgentInput, Flux<AgentEvent>> next) {
        long t0 = System.currentTimeMillis();
        return next.apply(input)
            .doOnComplete(() ->
                System.out.println("[timing] session=" + ctx.getSessionId()
                    + " elapsed=" + (System.currentTimeMillis() - t0) + "ms"));
    }
}
```

注册：

```java
ReActAgent agent = ReActAgent.builder()
    // ...
    .middleware(new TimingMiddleware())
    .build();
```

## 实验 2：Token 统计 Middleware

`/tmp/agent-lab/src/main/java/lab/TokenCounterMiddleware.java`：

```java
package lab;

import io.agentscope.core.agent.Agent;
import io.agentscope.core.agent.RuntimeContext;
import io.agentscope.core.middleware.*;
import io.agentscope.core.message.ContentBlock;
import io.agentscope.core.message.TextBlock;
import io.agentscope.core.model.*;
import reactor.core.publisher.Flux;

import java.util.concurrent.atomic.AtomicLong;
import java.util.function.Function;

public class TokenCounterMiddleware implements MiddlewareBase {

    private final AtomicLong totalTokens = new AtomicLong(0);

    @Override
    public Flux<ChatResponse> onModelCall(
            Agent agent, RuntimeContext ctx, ModelCallInput input,
            Function<ModelCallInput, Flux<ChatResponse>> next) {
        return next.apply(input)
            .doOnNext(resp -> {
                // 简化估算：每 4 字符 = 1 token
                long tokens = resp.getContent().stream()
                    .filter(b -> b instanceof TextBlock)
                    .mapToLong(b -> ((TextBlock) b).getText().length() / 4)
                    .sum();
                long cum = totalTokens.addAndGet(tokens);
                System.out.println("[tokens] +" + tokens + " cum=" + cum);
            });
    }

    public long getTotal() { return totalTokens.get(); }
}
```

## 实验 3：脱敏 Middleware

`/tmp/agent-lab/src/main/java/lab/SanitizerMiddleware.java`：

```java
package lab;

import io.agentscope.core.agent.Agent;
import io.agentscope.core.agent.RuntimeContext;
import io.agentscope.core.middleware.*;
import io.agentscope.core.message.*;
import reactor.core.publisher.Flux;

import java.util.List;
import java.util.function.Function;
import java.util.regex.Pattern;

public class SanitizerMiddleware implements MiddlewareBase {

    private static final Pattern PHONE = Pattern.compile("1[3-9]\\d{9}");

    @Override
    public Flux<AgentEvent> onReasoning(
            Agent agent, RuntimeContext ctx, ReasoningInput input,
            Function<ReasoningInput, Flux<AgentEvent>> next) {
        // 改写 input：把消息中手机号替换为 [REDACTED]
        List<Msg> sanitized = input.messages().stream()
            .map(m -> sanitize(m))
            .toList();
        ReasoningInput newInput = new ReasoningInput(
            sanitized, input.tools(), input.options()
        );
        return next.apply(newInput);
    }

    private Msg sanitize(Msg m) {
        if (m.getTextContent() == null) return m;
        String redacted = PHONE.matcher(m.getTextContent()).replaceAll("[REDACTED]");
        return m.withTextContent(redacted);
    }
}
```

## 实验 4：注入系统提示

`/tmp/agent-lab/src/main/java/lab/SkillInjector.java`：

```java
package lab;

import io.agentscope.core.agent.Agent;
import io.agentscope.core.agent.RuntimeContext;
import io.agentscope.core.middleware.MiddlewareBase;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class SkillInjector implements MiddlewareBase {

    @Override
    public String onSystemPrompt(Agent agent, RuntimeContext ctx, String systemPrompt) {
        String ts = LocalDateTime.now().format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);
        return systemPrompt + "\n\n[当前时间] " + ts;
    }
}
```

## 实验 5：洋葱顺序验证

注册多个 Middleware：

```java
.middleware(new Mw1())  // 第一个注册
.middleware(new Mw2())  // 第二个
.middleware(new Mw3())  // 第三个
```

每个 Middleware 的 `onAgent` 打印 enter / exit：

```java
public class Mw1 implements MiddlewareBase {
    @Override
    public Flux<AgentEvent> onAgent(Agent a, RuntimeContext c, AgentInput i,
            Function<AgentInput, Flux<AgentEvent>> n) {
        System.out.println("Mw1 enter");
        return n.apply(i).doOnComplete(() -> System.out.println("Mw1 exit"));
    }
}
```

**预期执行顺序**：

```
Mw1 enter
Mw2 enter
Mw3 enter
[core runs]
Mw3 exit
Mw2 exit
Mw1 exit
```

**结论**：第一个注册的**最外层**（先入后出）。

## 实验 6：短路 Middleware

```java
public class BlocklistMiddleware implements MiddlewareBase {
    @Override
    public Flux<AgentEvent> onAgent(Agent a, RuntimeContext c, AgentInput i,
            Function<AgentInput, Flux<AgentEvent>> n) {
        if (i.messages().get(0).getTextContent().contains("禁止")) {
            // 短路：不调 next，直接返回固定结果
            return Flux.just(/* 一个 AgentResultEvent 携带错误消息 */);
        }
        return n.apply(i);
    }
}
```

> **注意**：`AgentResultEvent` 是框架内部事件，构造它需要正确签名。看 `event/AgentResultEvent.java`。

## 验收标准

- [ ] 你能在 30 秒内描述洋葱模型执行顺序
- [ ] 你能写一个 `onAgent` Middleware 打印耗时
- [ ] 你能写一个 `onReasoning` Middleware 改写 messages
- [ ] 你能写一个 `onSystemPrompt` Middleware 追加内容
- [ ] 你能解释为什么 v1 Hook 已 deprecated

## 思考题答案提示

1. **最外层**：先注册的最外（最后出）
2. **onModelCall 拿到原始**：`onModelCall` 是**最里层**（最接近 LLM），拿到的是**未被后续中间件修改**的流
3. **敏感词拒绝**：`onAgent` 最合适，可在最外层直接短路

## 下一步

完成 Ch08 后你已经掌握框架的两个核心扩展点（Middleware 优先，Hook 已 deprecated）。Ch09 进入**结构化输出**。
