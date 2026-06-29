# Ch06 · 工具调用：`Toolkit` + `@Tool` + 反射

> 状态：🔲 · 预计时长：3h · 前置：Ch05

## 1. 本章目标

- 理解 `Toolkit` 的注册机制：`registerAgentTool` vs `registerTool(@Tool)`
- 掌握 `ToolBase` 抽象 + 7 个安全标志（readOnly / concurrencySafe / externalTool / stateInjected / mcp / dangerousFiles / dangerousDirectories）
- 理解 `ToolSchemaGenerator` 如何用 victools + Jackson 注解生成 JSON Schema
- 掌握 `ToolMethodInvoker` 反射调用机制
- 理解 `ToolGroup` 权限分组与 `ToolGroupManager`

## 2. 核心概念

### 2.1 三种工具注册路径

`Toolkit.java:154, 203, 619`：

| 方法 | 用途 | 适合 |
|---|---|---|
| `registerTool(Object toolObject)` | 扫描 `@Tool` 注解方法 | 业务代码、注解驱动 |
| `registerAgentTool(AgentTool tool)` | 注册已实现 `AgentTool` 的对象 | 框架内置工具、子类化 |
| `registerToolGroup(ToolGroup group)` | 注册一组工具（带激活条件） | 动态启用 / 禁用工具集 |

### 2.2 `@Tool` 注解驱动 vs `ToolBase` 继承

```java
// 方式 1：注解驱动（推荐 90% 场景）
public class WeatherTools {
    @Tool(name = "get_weather", description = "查天气")
    public String getWeather(
        @ToolParam(name = "city", description = "城市名") String city) {
        return "晴 25℃";
    }
}

// 注册
Toolkit tk = new Toolkit();
tk.registerTool(new WeatherTools());
```

```java
// 方式 2：继承 ToolBase（需要自定义权限 / 状态注入时）
public class MyTool extends ToolBase {
    public MyTool() {
        super("my_tool", "desc", schema, false, true, false, null, false, false);
    }
    @Override
    public Mono<ToolResultBlock> callAsync(ToolCallParam param) {
        return Mono.just(ToolResultBlock.text("result"));
    }
}

// 注册
tk.registerAgentTool(new MyTool());
```

### 2.3 `ToolBase` 的 9 个安全标志

读 `ToolBase.java` 构造器（`Builder` 模式）：

| 标志 | 作用 |
|---|---|
| `name` | 工具名（模型看到的名字） |
| `description` | 工具描述（模型决定何时调用） |
| `inputSchema` | JSON Schema（参数校验 + 模型 prompt） |
| `readOnly` | 是否只读（影响权限） |
| `concurrencySafe` | 同 session 是否可并发 |
| `externalTool` | 是否外部执行（返回 `ToolSuspendException`） |
| `stateInjected` | 是否注入 `AgentState` 参数 |
| `mcp` | 是否来自 MCP server |
| `mcpName` | MCP server 名 |
| `dangerousFiles` | 敏感文件路径列表 |
| `dangerousDirectories` | 敏感目录列表 |

### 2.4 `ToolSchemaGenerator` —— JSON Schema 自动生成

`ToolSchemaGenerator.java` + `ToolSchemaModule.java`：

- 基于 `victools/jsonschema-generator`（看 `pom.xml` 第 60+ 行）
- 模块 `jsonschema-module-jackson` 支持 Jackson 注解
- 从方法签名 + `@ToolParam` 注解生成完整 JSON Schema

**自动推断规则**：

| Java 类型 | JSON Schema 类型 |
|---|---|
| `String` | `{"type": "string"}` |
| `int` / `Integer` | `{"type": "integer"}` |
| `long` / `Long` | `{"type": "integer"}` |
| `double` / `Double` | `{"type": "number"}` |
| `boolean` / `Boolean` | `{"type": "boolean"}` |
| `List<String>` | `{"type": "array", "items": {"type": "string"}}` |
| `MyPojo` | 递归生成 |

### 2.5 `ToolMethodInvoker` 反射调用

`ToolMethodInvoker.java`（约 200 行）：

1. 拿到 `Method` 对象
2. 检查 `stateInjected` → 如需，注入 `AgentState`
3. 把 `Map<String,Object> input` 按参数名绑定
4. 处理 `Mono` 返回值 / 同步返回值转换
5. 异常 → `ToolResultBlock` 错误块

### 2.6 `ToolGroup` 权限分组

`ToolGroup.java:178` + `ToolGroupManager.java:103`：

```java
ToolGroup weatherGroup = ToolGroup.builder()
    .name("weather")
    .description("天气相关工具")
    .active(true)            // 默认激活
    .tools(Set.of("get_weather", "get_forecast"))
    .build();

tk.registerToolGroup(weatherGroup);
```

**激活机制**：

- Agent state 维护 `ToolContextState.activatedGroups`（一个 Set）
- 不同 session 可激活不同工具集
- ReAct 推理时，**只列出**激活组的工具给模型

## 3. 源码精读

### 3.1 `Toolkit.registerTool` 流程

读 `Toolkit.java:154-201`（约 50 行）：

```java
public void registerTool(Object toolObject) {
    // 1. 反射扫描所有声明方法
    for (Method m : toolObject.getClass().getDeclaredMethods()) {
        Tool annotation = m.getAnnotation(Tool.class);
        if (annotation == null) continue;

        // 2. 用 ToolSchemaGenerator 生成 JSON Schema
        Map<String,Object> schema = ToolSchemaGenerator.generate(m);

        // 3. 构造 ReflectiveFunctionTool（继承 ToolBase）
        ReflectiveFunctionTool rft = ReflectiveFunctionTool.create(
            toolObject, m, annotation, schema, ...

        // 4. 注册到内部 Map
        registerAgentTool(rft);
    }
}
```

**观察 1**：注册时**立即**生成 schema，避免每次推理时重新反射。

**观察 2**：`@Tool` 注解的方法签名是**业务**的，框架通过 `ReflectiveFunctionTool` 把它包装成 `ToolBase` 体系。

### 3.2 `ToolSchemaGenerator` 的注解支持

`ToolSchemaModule.java:51-62`（实际 123 行）：

```java
public void applyToConfigBuilder(SchemaGeneratorConfigBuilder builder) {
    // 注意：用 forFields()，不是 forTypesInGeneral()
    this.applyToConfigBuilder(builder.forFields());
}

private void applyToConfigBuilder(FieldScopeBuilder builder) {
    // private overload：处理字段级别的 Jackson 注解
    builder.withDescriptionResolver(...);
    // 注意：没有直接 new JacksonModule() 调用 —— Jackson 模块通常由外部初始化
}
```

支持：

- `@JsonProperty(name=..., description=...)`
- `@JsonPropertyDescription("...")`
- `@ToolParam(name=..., description=...)`

### 3.3 `ToolMethodInvoker` 的状态注入

读 `ToolMethodInvoker.java`（实际 399 行）关键段：

```java
// 注意：方法名是 convertParameters，不是 bindArguments
public Object[] convertParameters(Method m, Map<String,Object> input, ToolExecutionContext ctx) {
    List<Object> args = new ArrayList<>();
    for (Parameter p : m.getParameters()) {
        if (AgentState.class.isAssignableFrom(p.getType())) {
            // stateInjected=true：注入 state（来源是 ctx.runtimeContext() 或 agent.getAgentState() 兜底）
            args.add(extractState(ctx));
        } else {
            // 注意：用 @ToolParam.name()，不是直接 input.get(p.getName())
            String name = p.getAnnotation(ToolParam.class).name();
            args.add(input.get(name));   // 按名绑定（L291）
        }
    }
    return args.toArray();
}
```

**关键纠正**（与之前报告相比）：
- 方法名是 `convertParameters`（L149），不是 `bindArguments`
- 多了 `ToolExecutionContext ctx` 参数，`AgentState` 从 ctx 取，**不是方法直接拿**

**为什么需要 `stateInjected`**：某些工具需要**读写** state（如 `PlanNotebook`、`TodoTools`）。

### 3.4 `ToolExecutor` 的权限与并发控制

`ToolExecutor.java` 关键逻辑：

```java
public Mono<ToolResultBlock> execute(ToolCallParam param) {
    ToolBase tool = lookup(param.getToolUseBlock().getName());
    return tool.checkPermissions(param.getInput(), permCtx)
        .flatMap(decision -> {
            switch (decision.getBehavior()) {
                case ALLOW: return invoke(tool, param);
                case DENY:  return Mono.just(denied(decision.getMessage()));
                case ASK:   return Mono.error(new ToolSuspendException(...));
            }
        });
}
```

**关键**：`ASK` 模式返回 `ToolSuspendException`，Agent 会把这个抛到上层，由 `PostActingEvent` 处理为 `RequireUserConfirmEvent`。

## 4. 设计权衡

| 选择 | 原因 |
|---|---|
| 注解驱动（`@Tool`） | 业务代码最少，IDE 友好 |
| 同时支持 `ToolBase` 继承 | 框架内部工具（如 PlanNotebook）需要更细控制 |
| Schema 生成 + 缓存 | 性能（避免每轮重新反射） |
| `ToolGroup` 激活机制 | 动态工具集（场景化启用） |
| `stateInjected` 而非隐式 | 显式优于隐式，避免意外依赖 |

## 5. 实验任务

详见 [`lab/ch06-tool-registration.md`](../lab/ch06-tool-registration.md)。核心：

1. 用 `@Tool` 注解写一个 `WeatherTools` 和 `CalculatorTools`
2. 用 `ToolBase` 继承写一个 `LoggingTool`（带 `checkPermissions` 拦截）
3. 用 `ToolGroup` 创建"开发模式"和"生产模式"两套工具集
4. 用 `ToolSchemaGenerator` 单独生成 schema，对比手写 schema

## 6. 思考题

1. 如果同一个 `Tool` 类里两个方法 `@Tool` 注解 `name` 相同，会发生什么？
2. `stateInjected=true` 但方法签名**没有** `AgentState` 参数，注册时会报错吗？
3. `externalTool=true` 的工具，谁来"外部执行"它？（提示：MCP / A2A）

## 7. 参考资料

- `docs/v2/en/docs/building-blocks/tool.md`（约 590 行，**必读**）
- victools 官方：<https://victools.github.io/jsonschema-generator/>
- JSON Schema 规范：<https://json-schema.org/>

## 8. 学习笔记

在 `notes/ch06-my-takeaways.md` 写 3-5 条金句。

---

> 上一章：[Ch05](./ch05-react-loop-deep-dive.md) · 下一章：[Ch07](./ch07-memory-and-persistence.md)
