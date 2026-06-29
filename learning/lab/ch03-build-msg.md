# Ch03 · 实验：构造多模态 / 多轮消息

> 配套章节：[ch03-message-and-block.md](../chapters/ch03-message-and-block.md)
> 预计时长：30 分钟
> 前置：Ch01 环境就绪

## 目标

1. 熟练使用 `Msg.builder()` 构造各类消息
2. 验证 `Msg` 不可变性
3. 构造含工具调用的多轮对话快照
4. 打印 `toString` 验证结构

## 实验 1：基础文本消息

在仓库根目录创建临时测试类（不进 git）：

```bash
mkdir -p /tmp/msg-lab/src/main/java/lab
```

`/tmp/msg-lab/pom.xml`（参考 ch02 模板，把 `reactor-core` 也带上）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    <groupId>lab</groupId>
    <artifactId>msg-lab</artifactId>
    <version>1.0</version>
    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
    </properties>
    <dependencies>
        <!-- 复用 agentscope-java 的 core 模块 -->
        <dependency>
            <groupId>io.agentscope</groupId>
            <artifactId>agentscope-core</artifactId>
            <version>1.0.12</version>
        </dependency>
    </dependencies>
</project>
```

`/tmp/msg-lab/src/main/java/lab/MsgBuilder.java`：

```java
package lab;

import io.agentscope.core.message.*;
import java.util.List;
import java.util.Map;

public class MsgBuilder {
    public static void main(String[] args) {
        // 1. 用户消息
        Msg userMsg = Msg.builder()
            .role(MsgRole.USER)
            .textContent("北京今天天气怎么样？")
            .build();
        System.out.println("user  : " + userMsg);

        // 2. 助手消息（含文本 + 工具调用）
        Msg assistantMsg = Msg.builder()
            .role(MsgRole.ASSISTANT)
            .content(List.of(
                TextBlock.builder().text("让我查一下。").build(),
                ToolUseBlock.builder()
                    .id("call_001")
                    .name("get_weather")
                    .input(Map.of("city", "北京"))
                    .build()
            ))
            .build();
        System.out.println("assist: " + assistantMsg);

        // 3. 工具结果
        Msg toolMsg = Msg.builder()
            .role(MsgRole.TOOL)
            .content(List.of(
                ToolResultBlock.builder()
                    .toolCallId("call_001")
                    .name("get_weather")
                    .output(List.of(TextBlock.builder().text("北京：晴，25℃").build()))
                    .isError(false)
                    .build()
            ))
            .build();
        System.out.println("tool  : " + toolMsg);

        // 4. 助手最终回复
        Msg finalMsg = Msg.builder()
            .role(MsgRole.ASSISTANT)
            .textContent("北京今天晴，25℃，适合出门。")
            .build();
        System.out.println("final : " + finalMsg);

        // 5. 不可变性验证
        Msg modified = userMsg.withTextContent("上海呢？");
        System.out.println("\n[原]  " + userMsg.getTextContent());
        System.out.println("[改]  " + modified.getTextContent());
        System.out.println("[同?] " + (userMsg == modified));  // 期望 false
    }
}
```

```bash
cd /tmp/msg-lab
mvn -q compile exec:java -Dexec.mainClass=lab.MsgBuilder
```

## 实验 2：多模态消息

`/tmp/msg-lab/src/main/java/lab/Multimodal.java`：

```java
package lab;

import io.agentscope.core.message.*;
import java.util.List;

public class Multimodal {
    public static void main(String[] args) {
        // 1. 图像消息（URL 源）
        Msg imageMsg = Msg.builder()
            .role(MsgRole.USER)
            .content(List.of(
                TextBlock.builder().text("这张图里有什么？").build(),
                ImageBlock.builder()
                    .source(URLSource.builder()
                        .url("https://example.com/photo.jpg")
                        .build())
                    .build()
            ))
            .build();
        System.out.println("image msg: " + imageMsg);

        // 2. 思维链消息
        Msg thinkMsg = Msg.builder()
            .role(MsgRole.ASSISTANT)
            .content(List.of(
                ThinkingBlock.builder()
                    .thinking("用户问图里有什么，我需要先识别图...")
                    .build(),
                TextBlock.builder().text("图里有一只猫坐在窗台上。").build()
            ))
            .build();
        System.out.println("\nthink msg: " + thinkMsg);
    }
}
```

## 实验 3：完整对话快照

`/tmp/msg-lab/src/main/java/lab/ConversationSnapshot.java`：

```java
package lab;

import io.agentscope.core.message.*;
import io.agentscope.core.message.MessageMetadataKeys;
import java.util.List;
import java.util.Map;

public class ConversationSnapshot {
    public static void main(String[] args) {
        // 构造一个完整的 4 步 ReAct 快照
        List<Msg> conversation = List.of(
            // Step 1: 用户提问
            Msg.builder()
                .role(MsgRole.USER)
                .textContent("北京和上海今天天气分别怎样？")
                .build(),

            // Step 2: 助手并行发起两个工具调用
            Msg.builder()
                .role(MsgRole.ASSISTANT)
                .content(List.of(
                    ToolUseBlock.builder().id("c1").name("get_weather")
                        .input(Map.of("city", "北京")).build(),
                    ToolUseBlock.builder().id("c2").name("get_weather")
                        .input(Map.of("city", "上海")).build()
                ))
                .build(),

            // Step 3a: 北京的天气
            Msg.builder()
                .role(MsgRole.TOOL)
                .content(List.of(
                    ToolResultBlock.builder()
                        .toolCallId("c1").name("get_weather")
                        .output(List.of(TextBlock.builder().text("北京：晴 25℃").build()))
                        .isError(false)
                        .build()
                ))
                .build(),

            // Step 3b: 上海的天气
            Msg.builder()
                .role(MsgRole.TOOL)
                .content(List.of(
                    ToolResultBlock.builder()
                        .toolCallId("c2").name("get_weather")
                        .output(List.of(TextBlock.builder().text("上海：多云 28℃").build()))
                        .isError(false)
                        .build()
                ))
                .build(),

            // Step 4: 助手最终总结
            Msg.builder()
                .role(MsgRole.ASSISTANT)
                .textContent("北京晴 25℃，上海多云 28℃。")
                .build()
        );

        for (int i = 0; i < conversation.size(); i++) {
            Msg m = conversation.get(i);
            System.out.printf("[%d] role=%s blocks=%d text=%s%n",
                i, m.getRole(), m.getContent().size(),
                m.getTextContent() != null ? "\"" + m.getTextContent() + "\"" : "(non-text)");
        }
    }
}
```

## 实验 4：metadata 扩展

`/tmp/msg-lab/src/main/java/lab/MetadataDemo.java`：

```java
package lab;

import io.agentscope.core.message.*;
import java.util.List;
import java.util.Map;

public class MetadataDemo {
    public static void main(String[] args) {
        Msg userMsg = Msg.builder()
            .role(MsgRole.USER)
            .textContent("你好")
            .metadata("myapp.tenant_id", "tenant-42")     // 业务前缀
            .metadata("myapp.trace_id",  "abc-123")
            .build();

        System.out.println("metadata: " + userMsg.getMetadata());
        System.out.println("tenant:   " + userMsg.getMetadata().get("myapp.tenant_id"));
    }
}
```

## 验收标准

- [ ] 你能用 30 秒口头描述 `Msg` 的五个核心字段
- [ ] 你能用 30 秒解释为什么 `Msg` 要做成不可变
- [ ] 你能区分 5 种以上 `ContentBlock` 的使用场景
- [ ] 你能构造含 `TextBlock + ImageBlock` 的多模态消息
- [ ] 你能构造完整的工具调用四 Msg 序列

## 思考题答案提示

1. **多 ContentBlock 抽象**：让一个 `Msg` 同时含文本+图像+工具调用，避免类型膨胀（如 `MsgWithImage` / `MsgWithTool`）
2. **重复 id**：`ToolCallState.java` 跟踪状态，重复 id 会触发 `IllegalStateException`
3. **LocalDateTime**：可以放，但 JSON 序列化会变字符串；用 `Instant` 或 epoch 毫秒更稳
