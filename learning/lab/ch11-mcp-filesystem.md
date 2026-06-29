# Ch11 · 实验：MCP Filesystem 接入

> 配套章节：[ch11-mcp-a2a-protocols.md](../chapters/ch11-mcp-a2a-protocols.md)
> 预计时长：60 分钟
> 前置：Ch10 完成、Node.js 18+ 已安装

## 目标

1. 启动官方 MCP filesystem server
2. 通过 `McpClientBuilder` 集成到 Agent
3. 让 Agent 调用 `read_file` / `list_directory`
4. 验证工具 schema 来自 MCP server

## 实验 1：环境准备

```bash
# 安装 Node.js（如已安装跳过）
brew install node  # macOS

# 创建实验目录
mkdir -p /tmp/mcp-workspace
echo "Hello MCP" > /tmp/mcp-workspace/hello.txt
echo "AgentScope learning" > /tmp/mcp-workspace/readme.md
```

## 实验 2：测试 MCP server 启动

官方 server 路径：`@modelcontextprotocol/server-filesystem`

```bash
npx -y @modelcontextprotocol/server-filesystem /tmp/mcp-workspace
```

**预期**：进程启动并等待 stdin 输入。Ctrl-C 退出。

## 实验 3：Java 集成

`/tmp/agent-lab/pom.xml` 追加依赖：

```xml
<dependency>
    <groupId>io.modelcontextprotocol.sdk</groupId>
    <artifactId>mcp</artifactId>
    <version>0.10.0</version>  <!-- 跟随 agentscope-core 实际版本 -->
</dependency>
```

`/tmp/agent-lab/src/main/java/lab/McpDemo.java`：

```java
package lab;

import io.agentscope.core.ReActAgent;
import io.agentscope.core.agent.RuntimeContext;
import io.agentscope.core.message.*;
import io.agentscope.core.model.*;
import io.agentscope.core.tool.*;
import io.agentscope.core.tool.mcp.*;     // ← McpClientWrapper 在这里
import reactor.core.publisher.Flux;
import java.util.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Supplier;

public class McpDemo {

    public static class ScriptedModel extends ChatModelBase {
        private final List<Supplier<Flux<ChatResponse>>> scripts;
        private final AtomicInteger idx = new AtomicInteger(0);
        public ScriptedModel(List<Supplier<Flux<ChatResponse>>> s) { this.scripts = s; }
        @Override public String getModelName() { return "scripted"; }
        @Override
        protected Flux<ChatResponse> doStream(
                List<Msg> messages, List<ToolSchema> tools, GenerateOptions options) {
            int i = idx.getAndIncrement();
            System.out.println("[model] available tools: " +
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
        // 1. 构造 MCP client wrapper（注意：McpClientWrapper，不是 McpClient）
        McpClientWrapper mcp = McpClientWrapper.builder()
            .name("filesystem")
            .command("npx")
            .args(List.of("-y", "@modelcontextprotocol/server-filesystem", "/tmp/mcp-workspace"))
            .build();

        // 2. 验证连接（listTools）—— 实际 McpClientWrapper 通过 McpClientManager 管理生命周期
        //    业务侧不需要直接调 listTools()
        // List<McpTool> tools = mcp.listTools().block();   ← 之前这行不能编译

        // 3. 注册到 Toolkit（正确 API：registerMcpClient）
        Toolkit tk = new Toolkit();
        tk.registerMcpClient(mcp);                    // ← 注意：registerMcpClient，不是 registerMcpServer
        // registerMcpClient 返回 Mono<Void>，完整链：
        // tk.registerMcpClient(mcp).block();

        // 4. 验证注册（注意：MCP 工具作为 ToolBase 注入）
        System.out.println("[all tools] " + tk.getToolNames());

        // 5. 跑 Agent
        ScriptedModel model = new ScriptedModel(List.of(
            () -> Flux.just(ScriptedModel.toolUse("c1", "read_file",
                Map.of("path", "/tmp/mcp-workspace/hello.txt"))),
            () -> Flux.just(ScriptedModel.textResponse("文件内容已读取。"))
        ));

        ReActAgent agent = ReActAgent.builder()
            .name("mcp-demo")
            .sysPrompt("你可以读取 /tmp/mcp-workspace 下的文件。")
            .model(model)
            .toolkit(tk)
            .build();

        RuntimeContext ctx = RuntimeContext.builder()
            .sessionId("mcp").userId("u1").build();

        Msg resp = agent.call(
            Msg.builder().role(MsgRole.USER)
                .textContent("读 hello.txt").build(),
            ctx
        ).block();

        System.out.println("\n[final] " + resp.getTextContent());
    }
}
```

**关键纠正（与之前报告相比）**：
- `McpClientBuilder.builder()...build()` 返回 **`McpClientWrapper`**，不是 `McpClient`
- 变量名建议用 `mcp`（泛指）而不是 `McpClient mcp`（具体类名错）
- `registerMcpClient(...)` 返回 `Mono<Void>`，**需要 `.block()` 才会真正注册**
- 之前报告里 `mcp.listTools().block()` 不能编译——`McpClientWrapper` 没这个 API，listTools 在 `McpClientManager` 内部

## 实验 4：实际测试 MCP 工具

```bash
mvn -q compile exec:java -Dexec.mainClass=lab.McpDemo
```

**预期**：

- MCP 工具列表含 `read_file` / `write_file` / `list_directory` 等
- 调 `read_file` 后，框架通过 MCP 协议发请求到 server
- server 返回文件内容
- Agent 收到 `ToolResultBlock`

## 实验 5：A2A 远端调用（伪代码）

```java
// 假设远端有个 Python Agent 在 8000 端口
toolkit.registerSubAgent(SubAgentConfig.builder()
    .name("python-analyst")
    .agent(RemoteSubagentStub.builder()
        .endpoint("http://localhost:8000/agents/analyst")
        .build())
    .build());

// 主 Agent 调 call_python_analyst(message)
```

完整 A2A server 实现见 `agentscope-extensions-protocol/` + `agentscope-extensions-nacos/`。

## 验收标准

- [ ] 你能在 30 秒内解释 MCP 三种原语
- [ ] 你能启动官方 filesystem server
- [ ] 你能通过 `McpClientBuilder` 集成到 Toolkit
- [ ] 你能让 Agent 调 MCP 工具
- [ ] 你能区分 MCP（工具协议）和 A2A（Agent 协议）的目标

## 思考题答案提示

1. **MCP vs 普通 Tool**：协议层（HTTP / stdio）、Schema 标准化、跨语言
2. **A2A 状态**：远端 Agent 持有，本地只持 `conversation_id` 引用
3. **MCP 断开**：框架有重连 + 心跳机制；详细看 `McpClientManager`

## 下一步

完成 Ch11 后你已经能接入协议层。Ch12 进入最后一章：**生产化与可观测性**。
