# 学习路线图

> 写在正式开课前：明确『为什么学、学到什么程度、怎么学』。

## 1. 目标

把 `agentscope-java` 框架读懂到能**自己改造**的程度。具体可验证的产出：

- [ ] 能用白板画出一次 `agent.call(msg, ctx)` 内部完整事件流
- [ ] 能为框架新增一个 `Middleware` 或 `Hook`
- [ ] 能基于 `HarnessAgent` 构建一个能落盘可恢复的多 Agent 应用
- [ ] 能读懂 200 行以上的框架源码不迷路
- [ ] 能用 `ScriptedModel` 写出可重复的 Agent 集成测试

## 2. 前置知识

| 知识 | 必备程度 | 自检 |
|---|---|---|
| Java 17 语法（record, sealed, switch expression, text block） | 必备 | 能流畅阅读现代 Java |
| Maven 多模块项目 | 必备 | 知道 `parent` / `dependencyManagement` / `BOM` 是什么 |
| Stream API / Optional / Lambda | 必备 | 日常使用无压力 |
| Project Reactor（`Mono` / `Flux`） | **强烈建议** | 至少跑过官方 intro；不熟也无所谓，Ch02 会从零讲 |
| 设计模式（Builder, Strategy, Chain of Responsibility, Observer） | 建议 | 框架大量使用 |
| LLM 基础概念（token, system prompt, tool calling） | 建议 | 至少用过 ChatGPT / Claude |
| JSON Schema | 选 | Ch09 会讲最小必要集 |
| OpenTelemetry 概念 | 选 | Ch12 会讲 |

**判断标准**：你能用 5 分钟向同事讲清楚『ReAct 和普通 function calling 的本质差异』，就具备入门条件。

## 3. 学习方法论

### 3.1 三个层次递进

```
┌─────────────────────────────────────────┐
│  L3  改造层：能改框架源码、加扩展点       │
├─────────────────────────────────────────┤
│  L2  实践层：能基于框架写出生产级 Agent   │
├─────────────────────────────────────────┤
│  L1  阅读层：能读懂 ReAct 主循环每行代码 │
└─────────────────────────────────────────┘
```

每章学习**默认目标**是 L1，**挑战目标**是 L2/L3。

### 3.2 源码阅读法（"5 行 5 问"）

每进入一个新文件，按以下流程：

1. 先看 `package-info` 和类级 Javadoc（3 分钟）
2. 用 IDE 折叠全部方法，只看字段 + 构造器 + 公共方法签名（5 分钟）
3. 找到**入口方法**（如 `call`, `reasoning`），断点跑一遍
4. 读核心方法**前 5 行**，问自己 5 个问题：
   - 这一行在做什么？
   - 它依赖谁？
   - 它的输出是什么类型（`Mono` 还是 `Flux`）？
   - 它是否阻塞？
   - 如果出错会抛什么？
5. 用 "5 行 5 问" 一路推进到方法末尾

### 3.3 实验驱动

**禁止**只读不写。每章至少完成一个 lab 文档里的任务。判定标准：

- 实验跑通：日志里看到预期的事件序列
- 实验可重复：跑两次结果一致
- 实验可断言：能用 assert 验证关键状态

### 3.4 笔记法

每章留 5 条以内**金句**笔记（不是抄书），格式建议：

```markdown
## ChXX · 我的金句
1. **反应式不是『异步』的同义词** —— Reactor 解决的是背压 + 组合性，单纯 `CompletableFuture` 没有这两个
2. **ReAct 循环本质是把『工具调用』也建模成一次『模型推理』** —— 所以可以套娃
3. ...
```

放在 `notes/chXX-my-takeaways.md`。

## 4. 时间投入

- 每日 2-3 小时 × 14 天 ≈ 35 小时
- 周末可以集中 4-5 小时冲一章
- 不必每天推一章，关键是**不要断超过 3 天**（手会生）

## 5. 衡量结业

学完 12 章后，尝试独立完成：

> 用 `HarnessAgent` + `SubAgent` + `LongTermMemory` + MCP filesystem 工具，构建一个『个人研究助手』：
> 给定主题 → 子 Agent 检索本地知识库 → 主 Agent 整合 → 写入 MEMORY.md → 下次启动能回忆

能跑通即算结业。

## 6. 配套资料

- 官方文档：<https://java.agentscope.io/>
- DeepWiki：<https://deepwiki.com/agentscope-ai/agentscope-java>
- GitHub：<https://github.com/agentscope-ai/agentscope-java>
- 源码本仓库：`/agentscope-core/`, `/agentscope-harness/`, `/agentscope-extensions/`
- 参考实现：`/agentscope-examples/`

## 7. 反馈

发现内容错误、源码过期、想新增章节，直接在 `00-index.md` 末尾的 changelog 加一条。

---

## Changelog

- 2026-06-29 · 创建 · 12 章主线确定（v0.1，opencode 生成）
- 2026-06-29 · v0.2 · Claude 第 2 轮源码核验：修复 8 处 A 类严重错误 + 9 处 lab 编译失败 + 若干 B 类行号错
  - 关键修正：Ch01 call 入口行号（190 → 627）、Ch02 callInternal 行数（80 → 5）、Ch03 sealed class、Ch07 LongTermMemory deprecated、Ch09 Formatter 泛型 + structuredOutput API、Ch11 registerMcpClient API、lab 系列
  - 详细核验报告见 `~/.claude/plans/learning-opencode-cosmic-sloth.md`
