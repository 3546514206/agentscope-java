# AgentScope Java 学习笔记

> 个人 AI Agent 开发学习沉淀。课程主线基于 [`agentscope-java`](https://github.com/agentscope-ai/agentscope-java) v2 源码深度解读。
>
> **当前版本**：v0.2（2026-06-29 第二次源码核验后修订）—— 已修复 8 处 A 类严重错误 + 9 处 lab 编译失败。

## 目录

- **[00-index.md](./00-index.md)** — 课程总目录 + 进度看板
- **[01-roadmap.md](./01-roadmap.md)** — 路线图 + 前置知识 + 学习方法论
- **[chapters/](./chapters/)** — 12 章正文
- **[lab/](./lab/)** — 每章配套实验（基于 `ScriptedModel` 的 Markdown 文档）
- **[notes/](./notes/)** — 你的个人学习笔记
- **[assets/](./assets/)** — 流程图、序列图

## 快速开始

1. 先读 [`01-roadmap.md`](./01-roadmap.md) 建立全局认知
2. 按 `00-index.md` 顺序学习 `chapters/`
3. 每章正文读完，进入对应 `lab/` 跑一遍实验
4. 关键心得直接写在 `notes/`，自由发挥

## 实验前置

所有实验均基于 `ScriptedModel extends ChatModelBase`（`ChatModelBase` 是 `agentscope-core` 里所有 ChatModel 的抽象基类，**业务侧可直接继承**），无需任何 LLM API Key。

```bash
cd /path/to/agentscope-java
mvn -pl agentscope-core test -DfailIfNoTests=false
```

通过单元测试验证环境就绪后即可进入 Ch01。

## ⚠️ 阅读须知

- Ch07 涉及 `LongTermMemory` 概念，但该接口在 v2 已 `@Deprecated forRemoval since 2.0.0`。**新代码不应再实现 `LongTermMemory`**——跨会话持久化由应用层用 `AgentStateStore` 自己实现
- 章节中标注 `⚠️` 的位置都是 v1 → v2 演进中被废弃的 API
- 实验代码中标注 `注意：xxx 不存在` 的位置都是 opencode v0.1 报告里的错误，已在 v0.2 修正

## 多端阅读

仓库是纯 Markdown + Mermaid，推荐用以下任一工具阅读：

- VS Code / Cursor（`Ctrl+Shift+V` 预览，支持 Mermaid）
- Obsidian（双链友好，路径以 `learning/` 根）
- GitHub 网页直读
- Typora / MarkText
