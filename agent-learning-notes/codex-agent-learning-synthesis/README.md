# Codex Agent 学习过程整合专题

这个子目录是对本次 Codex agent 学习过程的二次整理。它不按提问时间排序，而是按“自己实现一个 Web Word agent”需要建立的心智模型组织。

原始专题笔记还保留在上一层目录；这里的目标是把它们整合成一套更连续的读物：

1. 先理解 agent 的业务本质。
2. 再理解 thread、session、turn、item、event、history 的边界。
3. 然后理解 prompt、模型返回、工具调用、事件和持久化如何形成闭环。
4. 最后把这些抽象迁移到 Web Word 产品。

## 阅读顺序

| 顺序 | 文件 | 解决的问题 |
| --- | --- | --- |
| 1 | `01-learning-map.md` | 本次学习从哪些问题开始，最后收敛成哪些核心结论。 |
| 2 | `02-agent-domain-model.md` | agent 的核心领域对象是什么，它们分别负责什么。 |
| 3 | `03-turn-and-history.md` | turn 运行期间，thread/session/turn/prompt/history/event 的关系是什么。 |
| 4 | `04-prompt-model-tools-loop.md` | 每次采样前发给模型什么，模型如何返回，工具如何执行和回灌。 |
| 5 | `05-persistence-and-ui-events.md` | item/event_msg/thread JSONL 的保存、恢复、UI 展示和模型历史有什么区别。 |
| 6 | `06-web-word-agent-design.md` | 如果自己实现 Web Word agent，应该抽取哪些架构和数据模型。 |
| 7 | `07-source-cross-reference.md` | 原始笔记和源码入口对照，方便以后回查。 |
| 8 | `08-flat-state-sequence-flow.md` | 按扁平化方式整理 thread、session、turn、prompt、tool、history、event、approval、compact/rollback/fork 和 Web Word 映射的状态机、时序图、业务流程图。 |

## 一句话总结

Codex agent 不是“把聊天记录发给模型”的简单程序，而是一个围绕 turn 运转的任务执行系统：

```text
用户目标
  -> thread/session 管理运行状态
  -> turn 构造上下文和工具
  -> 模型规划下一步
  -> agent 执行工具或请求用户
  -> 工具结果回灌模型
  -> item/event 持久化和展示
  -> turn 完成或继续采样
```

真正值得迁移到 Web Word agent 的不是 Rust 源码结构，而是这些边界：

- thread 负责可恢复的任务历史。
- session 负责运行时调度和内存状态。
- turn 负责一次用户目标的执行闭环。
- prompt 是每次采样前临时组装的模型输入，不等于持久历史。
- tool 是产品能力的安全出口，不是模型直接访问产品数据库。
- event 是 UI 和观测的实时状态流，不等于模型历史。
