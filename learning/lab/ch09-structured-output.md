# Ch09 · 实验：结构化输出

> 配套章节：[ch09-structured-output-and-formatter.md](../chapters/ch09-structured-output-and-formatter.md)
> 预计时长：45 分钟
> 前置：Ch08 完成

## 目标

1. 定义 `Order` POJO 并自动生成 JSON Schema
2. 让 Agent 输出 `Order` 并反序列化
3. 体验 `StructuredOutputReminder` 的提示效果
4. 对比原生 vs 工具调用兜底

## 实验 1：定义 Order POJO

`/tmp/agent-lab/src/main/java/lab/Order.java`：

```java
package lab;

import com.fasterxml.jackson.annotation.JsonProperty;
import com.fasterxml.jackson.annotation.JsonPropertyDescription;
import java.util.List;

public class Order {

    @JsonProperty(value = "order_id", required = true)
    @JsonPropertyDescription("订单号，字符串")
    public String orderId;

    @JsonProperty(value = "customer_name", required = true)
    @JsonPropertyDescription("客户姓名")
    public String customerName;

    @JsonProperty(value = "items", required = true)
    @JsonPropertyDescription("订单商品列表")
    public List<OrderItem> items;

    @JsonProperty(value = "total_amount", required = true)
    @JsonPropertyDescription("总金额，保留两位小数")
    public double totalAmount;

    public static class OrderItem {
        @JsonProperty(value = "sku", required = true)
        public String sku;

        @JsonProperty(value = "quantity", required = true)
        public int quantity;

        @JsonProperty(value = "unit_price", required = true)
        public double unitPrice;
    }

    @Override
    public String toString() {
        return "Order{orderId=" + orderId +
               ", customer=" + customerName +
               ", items=" + items.size() +
               ", total=" + totalAmount + "}";
    }
}
```

## 实验 2：生成 JSON Schema

`/tmp/agent-lab/src/main/java/lab/SchemaGen.java`：

```java
package lab;

import io.agentscope.core.tool.ToolSchemaGenerator;
import com.fasterxml.jackson.databind.JsonNode;

public class SchemaGen {
    public static void main(String[] args) {
        JsonNode schema = ToolSchemaGenerator.generate(Order.class);
        System.out.println(schema.toPrettyString());
    }
}
```

```bash
mvn -q compile exec:java -Dexec.mainClass=lab.SchemaGen
```

**预期输出**：

```json
{
  "type": "object",
  "properties": {
    "order_id": {"type": "string", "description": "订单号，字符串"},
    "customer_name": {"type": "string", "description": "客户姓名"},
    "items": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "sku": {"type": "string"},
          "quantity": {"type": "integer"},
          "unit_price": {"type": "number"}
        },
        "required": ["sku", "quantity", "unit_price"]
      }
    },
    "total_amount": {"type": "number", "description": "总金额，保留两位小数"}
  },
  "required": ["order_id", "customer_name", "items", "total_amount"]
}
```

## 实验 3：让 ScriptedModel 输出结构化 JSON

`/tmp/agent-lab/src/main/java/lab/StructuredOutput.java`：

```java
package lab;

import io.agentscope.core.ReActAgent;
import io.agentscope.core.agent.RuntimeContext;
import io.agentscope.core.message.*;
import io.agentscope.core.model.*;
import io.agentscope.core.tool.ToolSchemaGenerator;
import com.fasterxml.jackson.databind.*;
import reactor.core.publisher.Flux;
import java.util.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Supplier;

public class StructuredOutput {

    public static class ScriptedModel extends ChatModelBase {
        private final List<Supplier<Flux<ChatResponse>>> scripts;
        private final AtomicInteger idx = new AtomicInteger(0);
        public ScriptedModel(List<Supplier<Flux<ChatResponse>>> s) { this.scripts = s; }
        @Override public String getModelName() { return "scripted"; }
        @Override
        protected Flux<ChatResponse> doStream(
                List<Msg> messages, List<ToolSchema> tools, GenerateOptions options) {
            int i = idx.getAndIncrement();
            System.out.println("[model] call #" + i + " options.responseFormat=" +
                (options.getResponseFormat() != null ? "yes" : "no"));
            if (i >= scripts.size()) return Flux.just(textResponse("[end]"));
            return scripts.get(i).get();
        }
        public static ChatResponse textResponse(String t) {
            return ChatResponse.builder()
                .content(List.<ContentBlock>of(TextBlock.builder().text(t).build())).build();
        }
    }

    public static void main(String[] args) throws Exception {
        ObjectMapper mapper = new ObjectMapper();

        // 1. 预生成 Order 的 JSON
        String orderJson = """
            {
              "order_id": "ORD-001",
              "customer_name": "张三",
              "items": [
                {"sku": "BOOK-101", "quantity": 2, "unit_price": 49.9},
                {"sku": "PEN-202",  "quantity": 5, "unit_price": 12.5}
              ],
              "total_amount": 162.3
            }
            """;

        ScriptedModel model = new ScriptedModel(List.of(
            () -> Flux.just(ScriptedModel.textResponse(orderJson))
        ));

        ReActAgent agent = ReActAgent.builder()
            .name("struct-demo")
            .sysPrompt("你输出 JSON 订单。")
            .model(model)
            .build();

        RuntimeContext ctx = RuntimeContext.builder()
            .sessionId("struct").userId("u1").build();

        Msg resp = agent.call(
            Msg.builder().role(MsgRole.USER).textContent("给我一个示例订单").build(),
            ctx
        ).block();

        // 2. 解析为 POJO
        Order order = mapper.readValue(resp.getTextContent(), Order.class);
        System.out.println("[parsed] " + order);
    }
}
```

**注意**：本实验用 `ScriptedModel` 模拟"完美输出"，实际生产中：

- 用 `ReActAgent.builder().structuredOutput(Order.class).build()` 替代
- 框架会**自动**注册虚拟工具 `generate_response` 或设置 `response_format`
- 解析失败会自动**重试**

## 实验 4：完整结构化输出（生产用法）

```java
ReActAgent agent = ReActAgent.builder()
    .name("order-extractor")
    .sysPrompt("从用户输入抽取订单信息。")
    .model(realChatModel)
    .structuredOutput(Order.class)   // 框架接管一切
    .build();

// 直接拿 POJO
Order order = agent.call(userMsg, ctx)
    .map(msg -> mapper.readValue(msg.getTextContent(), Order.class))
    .block();
```

## 实验 5：观察 `StructuredOutputReminder`

注册一个 Middleware 打印 system prompt：

```java
.middleware(new MiddlewareBase() {
    @Override
    public String onSystemPrompt(Agent a, RuntimeContext c, String prompt) {
        System.out.println("[system prompt] " + prompt);
        return prompt;
    }
})
```

如果用了结构化输出但**未**设置原生 `response_format`，会看到 prompt 末尾追加：

```
...你的 prompt...

[结构化输出提醒]
请严格按以下 JSON Schema 输出 JSON：
{...}
```

## 验收标准

- [ ] 你能在 30 秒内解释两种结构化输出路径
- [ ] 你能用 `ToolSchemaGenerator.generate(Class)` 生成 JSON Schema
- [ ] 你能让 Agent 输出符合 schema 的 JSON 并反序列化为 POJO
- [ ] 你能区分 `response_format` 和 `ToolChoice.SPECIFIC` 的使用场景

## 思考题答案提示

1. **缺字段**：Jackson 默认抛 `UnrecognizedPropertyException` 或把字段设为 null（看配置）；框架有自动重试
2. **strict 行为**：OpenAI 严格校验、Anthropic 软提示、Gemini 强约束
3. **SPECIFIC vs AUTO**：SPECIFIC 强制模型调指定工具（用于结构化输出）；AUTO 让模型自己决定

## 下一步

完成 Ch09 后你已经掌握让 Agent 输出**强类型**的能力。Ch10 进入**多 Agent 编排**与 `HarnessAgent`。
