# Ch01 · 实验：环境就绪与模块结构

> 配套章节：[ch01-framework-overview.md](../chapters/ch01-framework-overview.md)
> 预计时长：30 分钟
> 前置：JDK 17+、Maven 3.9+、Git

## 目标

1. 验证本地构建链路通畅
2. 熟悉 core 模块的物理结构
3. 跑通一个最简单的 `VersionTest`

## 步骤 1：环境自检

```bash
java -version    # 期望 17.x 或 21.x
mvn -version     # 期望 3.9.x+
```

如果 Java < 17：

```bash
# macOS
brew install openjdk@17
export JAVA_HOME=$(/usr/libexec/java_home -v 17)

# Linux
sudo apt install openjdk-17-jdk
```

## 步骤 2：编译 core 模块

仓库根目录执行：

```bash
mvn -pl agentscope-core -am -DskipTests clean install
```

预期：

```
[INFO] BUILD SUCCESS
[INFO] Total time: ~ 30-90s（首次会下载依赖）
```

**常见错误**：

| 错误 | 原因 | 解法 |
|---|---|---|
| `JAVA_HOME is not set` | 环境变量缺失 | `export JAVA_HOME=...` |
| `invalid target release: 17` | Maven 用了 Java 8 | `mvn -version` 看 Java home |
| `Cannot resolve io.projectreactor:reactor-core` | 仓库未配置 | 检查 `~/.m2/settings.xml` 是否有公司内网镜像 |

## 步骤 3：跑通 VersionTest

```bash
mvn -pl agentscope-core test -Dtest=VersionTest
```

预期输出末尾：

```
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

如果想看更多测试（**会真正调用 LLM API，可能失败**）：

```bash
# 只跑不需要 API Key 的
mvn -pl agentscope-core test -Dtest='ReActAgentHitlTest,ReActAgentBuilderLegacyShimTest,ReActAgentPerSessionStateTest'
```

## 步骤 4：观察 core 子包结构

```bash
# 方式 1：tree
tree -L 3 agentscope-core/src/main/java/io/agentscope/core/

# 方式 2：find
find agentscope-core/src/main/java -maxdepth 4 -type d | sort
```

对照 `ch01` §3.2 的表，验证：

- ✅ `ReActAgent.java` 在根包，不在子包
- ✅ `agent/` 含 `AgentBase.java`
- ✅ `state/` 含 `AgentState.java`
- ✅ `tool/` 含 `Toolkit.java`（最大文件之一）

## 步骤 5：观察一个测试用 `ScriptedModel` 模式

打开 `agentscope-core/src/test/java/io/agentscope/core/agent/ReActAgentHitlTest.java:69`：

```java
private static final class ScriptedModel extends ChatModelBase {
    private final List<Supplier<Flux<ChatResponse>>> scripts;
    private final AtomicInteger idx = new AtomicInteger(0);

    ScriptedModel(List<Supplier<Flux<ChatResponse>>> scripts) {
        this.scripts = scripts;
    }

    @Override public String getModelName() { return "scripted"; }

    @Override
    protected Flux<ChatResponse> doStream(
            List<Msg> messages, List<ToolSchema> tools, GenerateOptions options) {
        int i = idx.getAndIncrement();
        if (i >= scripts.size()) {
            return Flux.just(textResponse(""));
        }
        return scripts.get(i).get();
    }
}
```

**关键观察**：

1. `ScriptedModel extends ChatModelBase` —— 任何 LLM 接入只需继承这一个抽象类
2. 注入 `List<Supplier<Flux<ChatResponse>>>` —— 用脚本驱动，**完全不需要 API Key**
3. `doStream` 返回 `Flux<ChatResponse>` —— 这就是后续所有章节会碰到的"反应式"返回值

**保存这个模板** —— Ch04 起会反复用到。

## 步骤 6（可选）：跑一遍全模块编译

```bash
mvn clean install -DskipTests
```

如果模块互相依赖有问题，错误会暴露在这一步。预期耗时 2-5 分钟。

## 验收标准

- [ ] `mvn -pl agentscope-core -am clean install` BUILD SUCCESS
- [ ] `mvn -pl agentscope-core test -Dtest=VersionTest` 1 passed
- [ ] 你能在 30 秒内口头说出 `ReActAgent` 与 `HarnessAgent` 的关系
- [ ] 你能在 30 秒内口头说出 `ScriptedModel` 的设计目的

## 常见问题

**Q: `mvn install` 报 `BOM` 错误怎么办？**
A: 检查 `agentscope-dependencies-bom/pom.xml` 是否能成功 install。可以单独：`mvn -pl agentscope-dependencies-bom -am install`

**Q: 测试运行很慢怎么办？**
A: 加上 `-DfailIfNoTests=false` 让没有匹配的测试时也不报错
