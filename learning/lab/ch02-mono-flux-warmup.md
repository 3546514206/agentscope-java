# Ch02 · 实验：Mono / Flux 暖身

> 配套章节：[ch02-reactive-foundation.md](../chapters/ch02-reactive-foundation.md)
> 预计时长：45 分钟
> 前置：JDK 17+、Maven 3.9+、Ch01 环境就绪

## 目标

1. 5 个必会操作符的最小可运行示例
2. 用 Reactor Context 模拟 `RuntimeContext` 透传
3. 用 `Flux` 模拟 LLM 的 SSE 流式 token
4. 验证 `Thread.sleep` 在 Reactor 中的问题

## 实验 1：基础 Mono 链

新建临时项目（不污染 agentscope-java）：

```bash
mkdir -p /tmp/reactor-lab && cd /tmp/reactor-lab
```

`pom.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    <groupId>lab</groupId>
    <artifactId>reactor-lab</artifactId>
    <version>1.0</version>
    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
    </properties>
    <dependencies>
        <dependency>
            <groupId>io.projectreactor</groupId>
            <artifactId>reactor-core</artifactId>
            <version>3.7.x</version>  <!-- 与 agentscope-core 保持一致即可 -->
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-simple</artifactId>
            <version>2.0.16</version>
        </dependency>
    </dependencies>
</project>
```

`src/main/java/lab/BasicMono.java`：

```java
package lab;

import reactor.core.publisher.Mono;
import java.time.Duration;

public class BasicMono {
    public static void main(String[] args) {
        System.out.println("[start] " + Thread.currentThread().getName());

        Mono<String> m = Mono.just("hello")
                .doOnNext(s -> System.out.println("doOnNext: " + s + " on " + Thread.currentThread().getName()))
                .map(String::toUpperCase)
                .delayElement(Duration.ofMillis(500))   // 模拟异步等待
                .doOnNext(s -> System.out.println("after delay: " + s + " on " + Thread.currentThread().getName()));

        // 订阅才触发
        m.block();

        System.out.println("[end]   " + Thread.currentThread().getName());
    }
}
```

```bash
mvn -q compile exec:java -Dexec.mainClass=lab.BasicMono
```

**观察**：

- `doOnNext` 第一次在 `main`，延迟后在 `parallel-1`（Reactor 线程池）
- `block()` 让 main 线程同步等结果（**生产中禁用**）
- "hello" → "HELLO"：map 同步转换

## 实验 2：Flux 模拟 LLM 流式返回

`src/main/java/lab/SimulatedLLM.java`：

```java
package lab;

import reactor.core.publisher.Flux;
import java.time.Duration;

public class SimulatedLLM {
    public static void main(String[] args) {
        Flux<String> tokens = Flux.just("你", "好", "，", "世", "界", "！")
                .zipWith(Flux.interval(Duration.ofMillis(100)))
                .map(t -> t.getT1())
                .doOnNext(t -> System.out.print(t))
                .doOnComplete(() -> System.out.println());

        // 模拟 UI 慢渲染
        tokens
            .onBackpressureBuffer(10)
            .delayElements(Duration.ofMillis(50))
            .blockLast();
    }
}
```

**观察**：

- 用 `zipWith(interval)` 给每个 token 加时间戳
- `onBackpressureBuffer(10)` 限制缓冲
- 慢下游时上游会自动暂停（背压生效）

## 实验 3：Mono.zip 并发工具调用

`src/main/java/lab/ParallelTools.java`：

```java
package lab;

import reactor.core.publisher.Mono;
import java.time.Duration;
import java.util.concurrent.atomic.AtomicInteger;

public class ParallelTools {
    static final AtomicInteger callCount = new AtomicInteger();

    static Mono<String> fakeTool(String name, int delayMs) {
        return Mono.fromCallable(() -> {
            int n = callCount.incrementAndGet();
            System.out.println("[call] " + name + " on " + Thread.currentThread().getName());
            try { Thread.sleep(delayMs); } catch (InterruptedException e) { /* 反模式演示 */ }
            return name + "(" + n + ")";
        }).subscribeOn(reactor.core.scheduler.Schedulers.boundedElastic());
    }

    public static void main(String[] args) {
        long t0 = System.currentTimeMillis();

        Mono.zip(
            fakeTool("weather",  300),
            fakeTool("calendar", 200),
            fakeTool("email",    400)
        ).doOnNext(t -> System.out.println("all done: " + t + " in " + (System.currentTimeMillis() - t0) + "ms"))
         .block();

        System.out.println("sequential would be: ~900ms; we got: " + (System.currentTimeMillis() - t0) + "ms");
    }
}
```

**观察**：

- 三个工具**并行**执行，总耗时 ≈ 400ms（最慢那个）
- `subscribeOn(boundedElastic)` 把阻塞 IO 切到专用线程池

**注意**：本实验**不是反模式演示**。`Thread.sleep` + `subscribeOn(boundedElastic)` **就是正确用法**——阻塞操作必须放到 `boundedElastic` 线程池执行，否则会卡 reactor 线程。

**真正要避免的反模式**：

```java
// ❌ 反例：阻塞操作没切线程池（会卡 reactor 线程）
Mono<String> bad = Mono.fromCallable(() -> {
    Thread.sleep(300);  // ← 在 reactor 线程上 sleep
    return "result";
});  // ← 没 subscribeOn(boundedElastic)
```

**ch02 之前报告里把"Thread.sleep + subscribeOn"说成"反模式演示"是错的**——这恰恰是**正确用法**。

## 实验 4：Context 透传（替代 ThreadLocal）

`src/main/java/lab/ContextPropagation.java`：

```java
package lab;

import reactor.core.publisher.Mono;
import reactor.util.context.Context;

public class ContextPropagation {
    public static void main(String[] args) {
        Mono<String> m = Mono.deferContextual(ctx -> {
                    String sessionId = ctx.get("sessionId");
                    String userId = ctx.get("userId");
                    return Mono.just("hello " + userId + "@" + sessionId);
                })
                .doOnNext(s -> System.out.println("on " + Thread.currentThread().getName() + ": " + s));

        m.contextWrite(Context.of("sessionId", "demo-session", "userId", "alice")).block();
        m.contextWrite(Context.of("sessionId", "other", "userId", "bob")).block();
    }
}
```

**观察**：

- `Context.of(...)` 创建不可变上下文
- `Mono.deferContextual` 在订阅时读取
- 同一个 `Mono` 定义，两次 `contextWrite` 产出不同结果

**对比 ThreadLocal**：

```java
// 反例：值可能丢失
ThreadLocal<String> session = new ThreadLocal<>();
session.set("demo");
Mono.just("...").map(s -> s + session.get()).block();  // 可能为 null
```

## 实验 5：错误兜底

`src/main/java/lab/ErrorHandling.java`：

```java
package lab;

import reactor.core.publisher.Mono;

public class ErrorHandling {
    public static void main(String[] args) {
        Mono.just("ok")
            .flatMap(s -> Mono.<String>error(new RuntimeException("llm timeout")))
            .onErrorResume(e -> Mono.just("[fallback] " + e.getMessage()))
            .doOnNext(System.out::println)
            .block();
    }
}
```

## 验收标准

- [ ] 你能在 30 秒内解释 `Mono` vs `Flux` 的区别
- [ ] 你能在 30 秒内解释 `flatMap` 和 `map` 的区别
- [ ] 你能在 30 秒内解释为什么 `Thread.sleep` + Reactor 会有问题
- [ ] 你能用 `Mono.zip` 编排 3 个并发任务

## 思考题答案提示

1. **Mono 链 vs 直接调用**：`Mono` 描述"如何做"，订阅时才真正执行；可直接组合、缓存、超时
2. **30 秒工具调用**：Agent 主循环在等 `Mono` 完成，UI 可以渲染"工具调用中"占位，**不阻塞**
3. **AtomicReference 跟踪 stop**：因为 `RequestStopEvent` 出现在流中间，需要在折叠时取到，filter + takeUntil 不能跨阶段传递

## 下一步

回到 Ch02 正文章节"§3 源码精读"，打开 `ReActAgent.java` 对照实验感受 Mono/Flux 的实际形态。
