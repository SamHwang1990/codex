# 学习过程地图

本次学习的起点不是“怎么维护 Codex 仓库”，而是借 Codex 的实现理解一个成熟 agent 系统应该具备哪些领域对象、执行流程、上下文策略、工具边界和持久化模型，然后迁移到自研 Web Word 产品。

## 1. 学习目标如何逐步收敛

最初的问题是：

> 通过这个仓库学习完整 agent 功能场景、方案设计与实现思路。

随后目标收敛成：

> 不是维护 Codex，而是实现一个和公司 Web Word 产品结合的 agent。

这导致学习重点从“源码怎么改”转成“领域模型怎么抽象”：

| 早期关注 | 后来收敛成 |
| --- | --- |
| Codex 有哪些功能 | 一个 agent 产品需要哪些能力面 |
| 代码入口在哪里 | 运行闭环和领域边界在哪里 |
| Turn 具体怎么跑 | 自己实现时 turn 应该承担什么职责 |
| 历史怎么保存 | 持久历史、内存历史、prompt 历史如何分层 |
| 模型输入输出是什么 | prompt 构造器和模型事件处理器怎么设计 |
| 工具怎么调用 | Web Word 产品能力如何安全暴露给模型 |

## 2. 学习问题主线

整个学习过程可以归纳为六条主线。

### 主线一：agent 的业务闭环

要先把 agent 看成“任务执行系统”，不是聊天接口。

核心闭环：

```text
接收用户目标
  -> 建立运行上下文
  -> 选择模型和工具
  -> 让模型决定下一步
  -> 执行工具或继续对话
  -> 保存过程和结果
  -> 完成、等待、中断或恢复
```

这一阶段形成了：

- `core-agent-business-logic-guide.md`
- `core-agent-ddd-closed-loops.md`
- `agent-feature-scenarios.md`

### 主线二：Turn 是一次任务执行，不是一条消息

用户输入一条消息之后，系统可能发生很多事情：

- 首次采样。
- 模型推理。
- 模型请求工具。
- 工具执行。
- 工具结果回灌。
- 后续采样。
- 需要用户确认。
- 用户追加要求。
- 最终回答。

所以 turn 的本质是“为了完成一个用户目标而启动的运行闭环”。

这一阶段形成了：

- `turn-domain-deep-dive.md`
- `first-sampling-prompt-structure.md`
- `model-response-turn-lifecycle.md`

### 主线三：“历史”不是一个东西

学习中最大的概念阻塞来自“历史”这个词。

最终结论是至少要拆成四层：

| 历史层 | 本质 | 主要作用 |
| --- | --- | --- |
| thread history | 持久化 JSONL 日志 | resume、fork、rollback、compact、审计 |
| session history | 内存中的候选上下文 | 当前运行时快速追加、替换和估算 token |
| turn history | 本轮执行视图 | 支撑本 turn 的一次或多次采样 |
| prompt history | 最终发给模型的输入片段 | 模型真正可见的历史 |

这一阶段形成了：

- `history-questions-dialogue.md`
- `thread-jsonl-history-schema.md`
- `turn-items-persistence-and-resampling.md`

### 主线四：模型交互不是一次请求一次回答

模型收到的是复合 prompt，请求体包含历史、当前用户输入、上下文注入、工具定义、推理配置、输出配置等。

模型返回的是流式事件和结构化 item，不只是文本。

常见返回包括：

- reasoning。
- assistant message。
- tool call。
- plan/update plan。
- request user input。
- hosted tool call。
- completed/failed/incomplete。

这一阶段形成了：

- `first-sampling-prompt-structure.md`
- `model-response-turn-lifecycle.md`

### 主线五：工具系统是产品能力边界

工具不是简单函数列表。Codex 里工具每次采样前重新发现和构造，并区分：

- 直接暴露给模型的工具。
- 延迟发现的工具。
- 本地执行的工具。
- 模型服务侧执行的 hosted tools。
- 不可用但可解释的 dummy tools。
- 动态工具。

这一阶段形成了：

- `turn-tool-discovery-call-model-flow.md`
- `agent-design-tradeoffs.md`

### 主线六：迁移到 Web Word

最后把 Codex 的设计抽象成 Web Word agent 可用的架构：

- 文档上下文而不是代码仓库上下文。
- 建议和批注优先，而不是默认直接修改正文。
- 工具必须可定位到文档 range、comment、suggestion、version。
- 历史保存要支持审计、恢复、回滚和协作。
- prompt 构造要按需注入文档片段，而不是塞入整篇文档。

这一阶段形成了：

- `web-word-agent-implementation-blueprint.md`

## 3. 最终学习结论

如果只保留最重要的结论，是这几条：

1. agent 的核心不是模型，而是 turn runner。
2. turn 的核心不是聊天，而是“采样、工具、回灌、再采样”的执行循环。
3. 历史不是原样发送，持久历史和模型历史必须分层。
4. 工具是模型影响外部世界的唯一受控出口。
5. event_msg 更接近日志和 UI 状态流，不应和模型历史混淆。
6. Web Word agent 应优先设计文档语义工具和可审计变更，而不是让模型直接编辑全文。

