# Ch06 · 实验：工具注册的三种姿势

> 配套章节：[ch06-toolkit-and-function-calling.md](../chapters/ch06-toolkit-and-function-calling.md)
> 预计时长：60 分钟
> 前置：Ch05 完成

## 目标

1. 掌握 `@Tool` 注解驱动的最小写法
2. 理解 `ToolBase` 继承 + `checkPermissions` 自定义权限
3. 理解 `ToolGroup` 激活 / 禁用
4. 验证 `ToolSchemaGenerator` 自动生成的 JSON Schema

## 实验 1：`@Tool` 注解

`/tmp/agent-lab/src/main/java/lab/WeatherTools.java`：

```java
package lab;

import io.agentscope.core.tool.annotation.Tool;
import io.agentscope.core.tool.annotation.ToolParam;

public class WeatherTools {

    @Tool(name = "get_weather", description = "查询指定城市的当前天气")
    public String getWeather(
        @ToolParam(name = "city", description = "城市名，如 '北京'") String city) {
        return switch (city) {
            case "北京" -> "晴 25℃";
            case "上海" -> "多云 28℃";
            case "广州" -> "雷阵雨 31℃";
            default -> "未知";
        };
    }
}
```

`/tmp/agent-lab/src/main/java/lab/CalculatorTools.java`：

```java
package lab;

import io.agentscope.core.tool.annotation.Tool;
import io.agentscope.core.tool.annotation.ToolParam;

public class CalculatorTools {

    @Tool(name = "add", description = "两个数相加")
    public double add(
        @ToolParam(name = "a", description = "加数") double a,
        @ToolParam(name = "b", description = "被加数") double b) {
        return a + b;
    }

    @Tool(name = "multiply", description = "两个数相乘")
    public double multiply(
        @ToolParam(name = "a") double a,
        @ToolParam(name = "b") double b) {
        return a * b;
    }
}
```

## 实验 2：`ToolBase` 继承 + 自定义权限

`/tmp/agent-lab/src/main/java/lab/SensitiveTool.java`：

```java
package lab;

import io.agentscope.core.tool.*;
import io.agentscope.core.message.ToolResultBlock;
import io.agentscope.core.permission.*;
import reactor.core.publisher.Mono;
import java.util.Map;

public class SensitiveTool extends ToolBase {

    public SensitiveTool() {
        super("delete_file", "删除文件（危险）",
            Map.of("type", "object",
                   "properties", Map.of("path", Map.of("type", "string")),
                   "required", List.of("path")),
            false,   // readOnly
            false,   // concurrencySafe
            false,   // externalTool
            null,    // stateInjected：false
            false,   // mcp
            false);  // mcpName
    }

    @Override
    public Mono<PermissionDecision> checkPermissions(
            Map<String, Object> toolInput, PermissionContextState context) {
        // 自定义：只允许"开发模式"调用
        String mode = (String) context.getMode();
        if ("dev".equals(mode)) {
            return Mono.just(PermissionDecision.allow("dev mode"));
        }
        return Mono.just(PermissionDecision.ask("生产模式需人工确认"));
    }

    @Override
    public Mono<ToolResultBlock> callAsync(ToolCallParam param) {
        String path = (String) param.getInput().get("path");
        return Mono.just(ToolResultBlock.text("已删除（模拟）：" + path));
    }
}
```

## 实验 3：注册 + 启动 Agent

`/tmp/agent-lab/src/main/java/lab/AgentWithTools.java`：

```java
package lab;

import io.agentscope.core.ReActAgent;
import io.agentscope.core.agent.RuntimeContext;
import io.agentscope.core.message.*;
import io.agentscope.core.model.*;
import io.agentscope.core.tool.*;
import reactor.core.publisher.Flux;
import java.util.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Supplier;

public class AgentWithTools {

    public static class ScriptedModel extends ChatModelBase {
        private final List<Supplier<Flux<ChatResponse>>> scripts;
        private final AtomicInteger idx = new AtomicInteger(0);
        public ScriptedModel(List<Supplier<Flux<ChatResponse>>> s) { this.scripts = s; }
        @Override public String getModelName() { return "scripted"; }
        @Override
        protected Flux<ChatResponse> doStream(
                List<Msg> messages, List<ToolSchema> tools, GenerateOptions options) {
            int i = idx.getAndIncrement();
            System.out.println("[model] call #" + i + " available tools: " +
                tools.stream().map(ToolSchema::getName).toList());
            if (i >= scripts.size()) return Flux.just(textResponse("[end]"));
            return scripts.get(i).get();
        }
        public static ChatResponse textResponse(String t) {
            return ChatResponse.builder()
                .content(List.<ContentBlock>of(TextBlock.builder().text(t).build())).build();
        }
        public static ChatResponse toolUse(String id, String name, Map<String,Object> input) {
            return ChatResponse.builder()
                .content(List.<ContentBlock>of(
                    ToolUseBlock.builder().id(id).name(name).input(input).build())).build();
        }
    }

    public static void main(String[] args) {
        // 1. 注册多个工具来源
        Toolkit tk = new Toolkit();
        tk.registerTool(new WeatherTools());
        tk.registerTool(new CalculatorTools());
        tk.registerAgentTool(new SensitiveTool());

        // 2. 打印工具名
        System.out.println("[tools] " + tk.getToolNames());

        // 3. 跑一个只发 get_weather 的脚本
        ScriptedModel model = new ScriptedModel(List.of(
            () -> Flux.just(ScriptedModel.toolUse("c1", "get_weather",
                Map.of("city", "北京"))),
            () -> Flux.just(ScriptedModel.textResponse("查好了。"))
        ));

        ReActAgent agent = ReActAgent.builder()
            .name("tools-demo")
            .sysPrompt("...")
            .model(model)
            .toolkit(tk)
            .build();

        RuntimeContext ctx = RuntimeContext.builder()
            .sessionId("tools-demo").userId("u1").build();

        Msg resp = agent.call(
            Msg.builder().role(MsgRole.USER).textContent("北京天气？").build(),
            ctx
        ).block();
        System.out.println("\n[final] " + resp.getTextContent());
    }
}
```

```bash
mvn -q compile exec:java -Dexec.mainClass=lab.AgentWithTools
```

**预期**：

```
[tools] [get_weather, add, multiply, delete_file]
[model] call #0 available tools: [get_weather, add, multiply, delete_file]
...
```

## 实验 4：`ToolGroup` 激活 / 禁用

`/tmp/agent-lab/src/main/java/lab/ToolGroupDemo.java`：

```java
package lab;

import io.agentscope.core.tool.*;
// 假设 Agent + Toolkit 已就绪
// tk.registerToolGroup(ToolGroup.builder()
//     .name("safe")
//     .active(true)
//     .tools(Set.of("get_weather", "add"))
//     .build());
// tk.registerToolGroup(ToolGroup.builder()
//     .name("dangerous")
//     .active(false)   // 默认不激活
//     .tools(Set.of("delete_file"))
//     .build());
//
// runtime：state.getToolContext().activateGroup("dangerous") // 手动激活
```

观察：

- 推理时 `tools` 列表**只含激活组的工具**
- 不同 session 可激活不同组

## 实验 5：Schema 自动生成 vs 手写

```java
// 自动：ToolSchemaGenerator.generate(method)
Map<String,Object> auto = ToolSchemaGenerator.generate(
    WeatherTools.class.getDeclaredMethod("getWeather", String.class)
);
System.out.println("[auto schema] " + auto);

// 手写：等价 schema
Map<String,Object> manual = Map.of(
    "type", "object",
    "properties", Map.of("city", Map.of("type", "string")),
    "required", List.of("city")
);
```

观察：两者结构应一致。

## 验收标准

- [ ] 你能在 30 秒内描述 `@Tool` 注解驱动的注册流程
- [ ] 你能用 `ToolBase` 继承 + `checkPermissions` 实现自定义权限
- [ ] 你能用 `ToolGroup` 启用 / 禁用工具集
- [ ] 你能解释 `stateInjected` 的作用
- [ ] 你能验证 `ToolSchemaGenerator` 生成的 JSON Schema

## 思考题答案提示

1. **同名工具**：后注册的覆盖先注册的，但会打 warn 日志
2. **stateInjected 缺参数**：构造 `ReflectiveFunctionTool` 时校验失败 → 抛 `IllegalStateException`
3. **externalTool**：MCP server / A2A 远端 agent 负责实际执行

## 下一步

完成 Ch06 后你已经掌握工具调用全链路。Ch07 进入**记忆与持久化**。
