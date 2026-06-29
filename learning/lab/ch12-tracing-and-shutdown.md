# Ch12 · 实验：Trace 导出与优雅停机

> 配套章节：[ch12-production-observability.md](../chapters/ch12-production-observability.md)
> 预计时长：60 分钟
> 前置：Ch11 完成

## 目标

1. 注册 `JsonlTraceExporter` 把所有事件写到文件
2. 触发 `GracefulShutdownManager.shutdown()`，观察活跃请求被等待
3. 跑一个 `STRICT` 模式权限实验
4. 验证 trace 文件可读、可分析

## 实验 1：Jsonl Trace 导出

`/tmp/agent-lab/src/main/java/lab/TraceDemo.java`：

```java
package lab;

import io.agentscope.core.ReActAgent;
import io.agentscope.core.agent.RuntimeContext;
import io.agentscope.core.hook.*;
import io.agentscope.core.event.*;
import io.agentscope.core.message.*;
import io.agentscope.core.model.*;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;
import java.nio.file.*;
import java.util.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Supplier;

public class TraceDemo {

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
        Path traceFile = Path.of("/tmp/agent-lab/trace.jsonl");
        Files.deleteIfExists(traceFile);

        // 1. 注册 JsonlTraceExporter（注意：v2 推荐用 MiddlewareBase 重写）
        // 这里是简化版 Hook 演示
        Hook traceHook = new Hook() {
            @Override public String getName() { return "trace"; }

            // 正确签名：<T extends HookEvent> Mono<T> onEvent(T event) —— 单参数泛型
            // 之前报告里写的 onEvent(AgentEvent event, HookEventType type) 双参数不能编译
            @Override
            public <T extends HookEvent> Mono<T> onEvent(T event) {
                String evName = event.getClass().getSimpleName();
                return Mono.fromRunnable(() -> {
                    try {
                        String line = String.format(
                            "{\"event\":\"%s\"}", evName);
                        Files.writeString(traceFile, line + "\n",
                            StandardOpenOption.CREATE, StandardOpenOption.APPEND);
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }).then(Mono.just(event));
            }
        };

        ScriptedModel model = new ScriptedModel(List.of(
            () -> Flux.just(ScriptedModel.textResponse("done"))
        ));

        ReActAgent agent = ReActAgent.builder()
            .name("trace-demo")
            .sysPrompt("...")
            .model(model)
            .hook(traceHook)  // 注意：v1 Hook 已 deprecated，生产用 Middleware
            .build();

        RuntimeContext ctx = RuntimeContext.builder()
            .sessionId("trace").userId("u1").build();

        agent.call(
            Msg.builder().role(MsgRole.USER).textContent("hi").build(),
            ctx
        ).block();

        // 2. 打印 trace 文件
        System.out.println("\n=== trace.jsonl ===");
        Files.lines(traceFile).forEach(System.out::println);
    }
}
```

**注意**：

- v2 推荐用 `MiddlewareBase` 重写 trace 收集（更强大）
- 本实验为简化展示，**实际生产用 `tracing/OtelTracingMiddleware` 或更新的 exporter**

## 实验 2：优雅停机

`/tmp/agent-lab/src/main/java/lab/GracefulShutdownDemo.java`：

```java
package lab;

import io.agentscope.core.ReActAgent;
import io.agentscope.core.agent.RuntimeContext;
import io.agentscope.core.shutdown.*;
import io.agentscope.core.message.*;
import io.agentscope.core.model.*;
import reactor.core.publisher.Flux;
import java.util.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Supplier;

public class GracefulShutdownDemo {

    public static void main(String[] args) throws Exception {
        // 1. 配置优雅停机
        GracefulShutdownManager shutdown = GracefulShutdownManager.builder()
            .timeoutMs(10_000)  // 10 秒超时
            .build();

        // 2. 启动一个慢 Agent
        ScriptedSlowModel slowModel = new ScriptedSlowModel();
        ReActAgent agent = ReActAgent.builder()
            .name("slow")
            .sysPrompt("...")
            .model(slowModel)
            .middleware(new GracefulShutdownMiddleware(shutdown))
            .build();

        RuntimeContext ctx = RuntimeContext.builder()
            .sessionId("shutdown").userId("u1").build();

        // 3. 在后台发起请求
        Thread t = new Thread(() -> {
            try {
                System.out.println("[t1] calling agent...");
                agent.call(
                    Msg.builder().role(MsgRole.USER).textContent("slow task").build(),
                    ctx
                ).block();
                System.out.println("[t1] done.");
            } catch (Exception e) {
                System.out.println("[t1] error: " + e.getMessage());
            }
        });
        t.start();

        Thread.sleep(1000);

        // 4. 同时再起一个请求（会立即被拒）
        try {
            agent.call(
                Msg.builder().role(MsgRole.USER).textContent("new task").build(),
                ctx
            ).block();
        } catch (Exception e) {
            System.out.println("[t2] rejected: " + e.getClass().getSimpleName());
        }

        // 5. 触发优雅停机
        System.out.println("[main] initiating shutdown...");
        long t0 = System.currentTimeMillis();
        shutdown.shutdown();
        System.out.println("[main] shutdown returned in " +
            (System.currentTimeMillis() - t0) + "ms");

        t.join();
    }

    static class ScriptedSlowModel extends ChatModelBase {
        @Override public String getModelName() { return "slow"; }
        @Override
        protected Flux<ChatResponse> doStream(
                List<Msg> messages, List<ToolSchema> tools, GenerateOptions options) {
            return Flux.create(sink -> {
                try { Thread.sleep(2000); } catch (InterruptedException e) {
                    sink.error(e);
                    return;
                }
                sink.next(ChatResponse.builder()
                    .content(List.<ContentBlock>of(
                        io.agentscope.core.message.TextBlock.builder()
                            .text("slow done").build()))
                    .build());
                sink.complete();
            });
        }
    }
}
```

**预期**：

```
[t1] calling agent...
[t2] rejected: AgentShuttingDownException
[main] initiating shutdown...
[main] shutdown returned in ~2000ms  (等 t1 跑完)
[t1] done.
```

## 实验 3：BYPASS 模式权限（注意：没有 STRICT 模式）

```java
PermissionContextState ctx = PermissionContextState.builder()
    .mode(PermissionMode.BYPASS)              // ← 没有 STRICT 模式！用 BYPASS 替代
    .build();

// BYPASS 模式：绕过所有权限检查
// （慎用：仅在受控环境/测试用）

// 对比：
// - DEFAULT:  按 ask 规则与工具自检判定
// - ACCEPT_EDITS:  自动接受编辑类工具
// - EXPLORE:  只读工具全部 ALLOW
// - BYPASS:  绕过所有检查
// - DONT_ASK:  不询问（按 deny/allow 规则判定）
```

**观察**：

- BYPASS 模式下所有工具调用都直接通过
- 如果要"每个工具都要求确认"，用 `DONT_ASK` 模式（把 ask 决策降级为 deny），或注册 `PermissionRule` 强制 ask

## 验收标准

- [ ] 你能在 30 秒内解释优雅停机的状态机
- [ ] 你能注册 trace 导出器并验证文件内容
- [ ] 你能区分 OTel / Jsonl 两种 trace 路径
- [ ] 你能解释权限的三级决策

## 思考题答案提示

1. **OTel parent 传递**：用 Reactor Context 携带 trace context，跨线程池自动传播
2. **Jsonl 性能瓶颈**：每事件一次 `Files.writeString`，可加 `BufferedWriter`
3. **新请求拒绝**：被 `GracefulShutdownMiddleware` 拦截，抛 `AgentShuttingDownException`

## 课程结语

完成 Ch12 后你已经具备：

- ✅ 读懂框架源码（核心 12 个子包）
- ✅ 编写自定义 Middleware / Hook
- ✅ 集成多模型、多工具、多 Agent
- ✅ 持久化、长期记忆、结构化输出
- ✅ MCP / A2A 协议接入
- ✅ 优雅停机 + 权限 + trace

**建议的下一步**：

1. 写一个完整的 `HarnessAgent` 结业项目
2. 阅读 `agentscope-examples/` 下的官方示例
3. 在公司项目里试用
4. 给框架提 PR / 写博客
