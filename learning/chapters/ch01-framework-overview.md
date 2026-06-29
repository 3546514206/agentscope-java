# Ch01 · 框架全景与学习地图

> 状态：🔲 · 预计时长：1.5h · 前置：无

## 1. 本章目标

- 能在白板画出 `agentscope-java` 的模块拓扑
- 理解 `core` / `harness` / `extensions` / `examples` 各自的角色
- 知道 `ReActAgent` 与 `HarnessAgent` 的关系
- 能在本地把仓库构建起来

## 2. 核心概念

### 2.1 它是什么、不是什么

`agentscope-java` 是一个 **agent-oriented 编程框架**，目标用户是 Java 后端工程师。它**不是**：

- ❌ 一个 LLM 客户端（那是 `ChatModelBase` 子类的事）
- ❌ 一个工作流引擎（它是 ReAct 范式，而非 DAG）
- ❌ 一个向量数据库（它消费 RAG 能力，但本身不存 embedding）

它是：

- ✅ ReAct 推理循环的参考实现
- ✅ 模型无关的 Agent 运行时（5 家 ChatModel 实现 + 1 个 ModelRegistry）
- ✅ 工具调用、记忆、状态、中间件、Hook 的统一抽象
- ✅ 企业级工程增强（`HarnessAgent` = ReActAgent + Workspace + 长期记忆 + 沙箱 + 多 Agent + A2A/MCP）

### 2.2 一句话总结

> **`ReActAgent` 是引擎，`HarnessAgent` 是整车。** 你要造引擎读 `core`，你要开车用 `harness`。

## 3. 源码精读

### 3.1 模块拓扑

```
agentscope-java/
├── pom.xml                          # 父 POM，定义 revision / dependencyManagement
├── agentscope-core/                 # 引擎本体 (~50 个子包)
├── agentscope-harness/              # 整车：workspace + 长期记忆 + 子 Agent + 沙箱
├── agentscope-extensions/           # 可选扩展（16 个子模块）
│   ├── agentscope-extensions-channel/   # 消息渠道（IM、Webhook）
│   ├── agentscope-extensions-higress/   # Higress 网关
│   ├── agentscope-extensions-mem/       # 长期记忆（Redis、MySQL）
│   ├── agentscope-extensions-mysql/     # 状态存储
│   ├── agentscope-extensions-nacos/     # 服务发现 + A2A
│   ├── agentscope-extensions-oss/       # 对象存储
│   ├── agentscope-extensions-protocol/  # A2A 协议基础
│   ├── agentscope-extensions-rag/       # RAG（向量检索）
│   ├── agentscope-extensions-redis/     # Redis 状态存储
│   ├── agentscope-extensions-sandbox/   # 沙箱（Docker、Shell、GUI）
│   ├── agentscope-extensions-scheduler/ # 定时任务
│   ├── agentscope-extensions-skills/    # 技能包
│   ├── agentscope-extensions-studio/    # 可视化 Studio
│   ├── agentscope-extensions-training/  # 训练后端
│   └── agentscope-spring-boot-starters/ # Spring Boot 自动装配
├── agentscope-distribution/         # BOM 聚合包
└── agentscope-examples/             # 官方示例
```

**观察**：所有扩展模块名都以 `agentscope-extensions-` 开头，通过 `<dependencyManagement>` 统一版本号。

### 3.2 core 的子包地图

读 `agentscope-core/src/main/java/io/agentscope/core/`：

| 包 | 一句话职责 | 关键文件 |
|---|---|---|
| `ReActAgent.java` | ReAct 循环本体 | `ReActAgent.java:200, 769, 1835, 2167, 2937` |
| `agent/` | Agent 接口与骨架 | `Agent.java`, `AgentBase.java`, `RuntimeContext.java` |
| `message/` | 消息模型 | `Msg.java` 842 行 + 12 个 ContentBlock |
| `tool/` | 工具调用体系 | `Toolkit.java` 1031 行 + 30+ 工具相关类 |
| `model/` | 模型抽象 | `ChatModelBase.java` + 5 家实现 + transport/ |
| `formatter/` | 多家协议适配 | openai/dashscope/anthropic/gemini/ollama |
| `memory/` | 记忆（v1 旧接口，v2 改用 AgentState） | `Memory.java` 已 `@Deprecated` |
| `state/` | 状态（v2 新范式） | `AgentState.java`, `AgentStateStore.java` |
| `hook/` | 事件订阅 | `Hook.java` + 9 类事件 |
| `middleware/` | AOP 拦截 | `MiddlewareChain.java`, `MiddlewareBase.java` |
| `event/` | 事件定义 | 30+ 事件类型 |
| `tracing/` | OTel 集成 | `OtelTracingMiddleware.java` |
| `permission/` | 权限系统 | `PermissionEngine.java` |
| `shutdown/` | 优雅停机 | `GracefulShutdownManager.java` |
| `interruption/` | 中断控制 | `InterruptContext.java` |
| `rag/` | RAG 抽象 | `Knowledge.java`, `GenericRAGHook.java` |
| `skill/` | 技能 | `AgentSkill.java`, `SkillBox.java` |
| `exception/`, `util/`, `credential/`, `workspace/` | 杂项 | — |

### 3.3 入口与依赖

读 `pom.xml`（仓库根）和 `agentscope-core/pom.xml`：

```xml
<!-- agentscope-core/pom.xml 第 47-110 行 -->
<dependencies>
    <!-- 反应式底座 -->
    <dependency>
        <groupId>io.projectreactor</groupId>
        <artifactId>reactor-core</artifactId>
    </dependency>

    <!-- JSON -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>

    <!-- JSON Schema 生成（用于结构化输出 / 工具 schema） -->
    <dependency>
        <groupId>com.github.victools</groupId>
        <artifactId>jsonschema-generator</artifactId>
    </dependency>
    <dependency>
        <groupId>com.github.victools</groupId>
        <artifactId>jsonschema-module-jackson</artifactId>
    </dependency>
</dependencies>
```

**设计判断**：

- 只引 Reactor **核心**，没引 `reactor-netty`（HTTP 走 `java.net.http.HttpClient` 或 OkHttp 适配）
- 选 victools 而非 json-schema-generator 的官方实现，因为它对 Jackson 注解友好
- 没有 Spring 依赖（`agentscope-extensions-spring-boot-starters` 才引 Spring）

### 3.4 ReActAgent 与 HarnessAgent 的关系

读 `agentscope-harness/src/main/java/io/agentscope/harness/agent/HarnessAgent.java`（约 200 行）：

```text
HarnessAgent
  ├── 内部持有一个 ReActAgent（核心推理能力）
  ├── 包装 Workspace（AGENTS.md / MEMORY.md / skills/）
  ├── 包装 LongTermMemory + Compaction（自动压缩）
  ├── 包装 SubAgentFactory（多 Agent 协作）
  ├── 包装 Sandbox 注入（FileSystemTool 安全执行）
  └── 包装 ToolRegistry 聚合（MCP / 内置 / 业务工具）
```

**关键洞察**：`HarnessAgent` 不是 ReActAgent 的子类，是**装饰者**（Decorator 模式）。它把 ReActAgent 包一层，加工程能力。

## 4. 设计权衡

| 选择 | 原因 | 替代方案 |
|---|---|---|
| 反应式底座 | 工具调用天然异步、利于流式输出、避免线程池堆积 | `CompletableFuture`、回调、`kotlinx.coroutines` |
| Builder 模式 | 30+ 字段的复杂对象，构造器无法管理可选项 | Lombok `@Builder`（部分类用了）、Kotlin DSL |
| 装饰者组装（HarnessAgent 包裹 ReActAgent） | 用户可单独用 core，也能用 harness | 单一巨型类、抽象基类继承 |
| `Toolkit` 反射注册 | 用户写业务代码最少（@Tool 注解 + registerObject） | 手写 `Tool` 实现类 |
| 多家 ChatModel 抽象 | 同一份业务代码可换模型 | 每家 SDK 写一遍 |

## 5. 实验任务

详见 [`lab/ch01-environment-setup.md`](../lab/ch01-environment-setup.md)。核心目标：

1. 克隆仓库（你已有）
2. `mvn -pl agentscope-core -am compile` 通过
3. `mvn -pl agentscope-core test -Dtest=VersionTest` 通过
4. 用 `tree -L 3 agentscope-core/src/main/java/io/agentscope/core/` 打印子包结构，验证 §3.2 表与实际一致

## 6. 思考题

1. 为什么框架选 Reactor 而不是 `CompletableFuture`？提示：看 `ReActAgent.java` 中 `flatMap` / `concatMap` 的使用密度。
2. 如果让你加一个 `agentscope-extensions-kafka`（让 Agent 监听 Kafka 消息），pom 怎么写？
3. `HarnessAgent` 为什么不继承 `ReActAgent`？

## 7. 参考资料

- 官方文档：<https://java.agentscope.io/>
- DeepWiki：<https://deepwiki.com/agentscope-ai/agentscope-java>
- `docs/v2/en/intro.md` — 介绍页（重点看 harness 部分）
- `docs/v2/en/docs/quickstart.md` — 5 分钟上手

## 8. 学习笔记

在 `notes/ch01-my-takeaways.md` 写 3-5 条金句。

---

> 下一章：[Ch02 · 反应式编程基石](./ch02-reactive-foundation.md)
