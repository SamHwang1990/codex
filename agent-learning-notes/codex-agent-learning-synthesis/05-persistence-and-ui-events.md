# 持久化、Item 与 UI 事件整合

这一页整合 `response_item`、`event_msg`、thread JSONL、UI 状态和模型历史之间的关系。

## 1. 三类东西要分开

| 名称 | 本质 | 主要消费者 | 是否等于模型历史 |
| --- | --- | --- | --- |
| response_item | 模型可见或接近模型可见的结构化 item | session history、后续 prompt、持久化恢复 | 接近，但仍需加工 |
| event_msg | 运行时事件、UI 状态、审计信号 | UI、app-server、日志、恢复辅助 | 多数不是 |
| rollout line | JSONL 持久化行 | resume、fork、rollback、compact、审计 | 不是 |

关键结论：

> response_item 更像 agent、模型、用户交互的状态快照；event_msg 更像运行事件和 UI/日志信号。

## 2. Thread JSONL 保存什么

JSONL 不是 `messages[]`。它可能包含：

```ts
type RolloutLine =
  | { type: "session_meta"; value: SessionMeta }
  | { type: "response_item"; value: ResponseItem }
  | { type: "turn_context"; value: TurnContext }
  | { type: "compacted"; value: CompactedHistory }
  | { type: "event_msg"; value: AgentEvent };
```

每类的含义：

| 类型 | 来源 | 作用 |
| --- | --- | --- |
| session_meta | thread 创建/恢复初始化 | 保存 thread 元信息和恢复基线 |
| response_item | 模型输入输出、工具 call/output、用户消息 | 形成后续可加工历史 |
| turn_context | 每个真实用户 turn 的上下文快照 | resume 后恢复运行环境 |
| compacted | 历史压缩 | 替换旧历史，降低上下文长度 |
| event_msg | 运行时事件按策略落盘 | UI 重放、审计、rollback、usage 等 |

## 3. Item 是否“完结才落盘”

直觉上会觉得 item 要完整后才能保存。实际要分情况：

- 流式 delta 通常先服务 UI，不一定每个 delta 都落盘。
- 结构化 item 完成后，才更适合以 `response_item` 形式保存。
- 工具 call 一般在模型输出完成后保存，再执行工具。
- 工具 output 在工具执行完成后保存。
- 有些 event 会在开始时发出，用于 UI 状态，但未必进入模型历史。

可以这样理解：

```text
stream delta
  -> UI 实时显示
  -> item done
  -> response_item 保存
  -> 如果是 tool call，执行工具
  -> tool output done
  -> response_item/tool event 保存
```

## 4. event_msg 的生产和消费链路

```text
runtime action
  -> create event_msg
  -> send to event bus
  -> UI/app-server 实时消费
  -> persistence filter 判断是否写 JSONL
  -> resume/replay 时部分事件影响状态
```

event_msg 常见用途：

- UI 展示 turn started/completed。
- UI 展示 item started/delta/completed。
- 展示工具执行进度。
- 展示审批请求。
- 展示 token usage。
- 记录 rollback。
- 记录错误。

但它不是“给模型看的对话”。

## 5. 哪些东西会进入后续采样

会进入或可能进入：

- 用户消息。
- assistant 消息。
- reasoning summary。
- tool call。
- tool output。
- compacted replacement history。
- 某些上下文注入 item。

通常不会直接进入：

- session_meta。
- 大多数 turn_started/turn_completed UI event。
- token usage event。
- 纯 UI delta。
- telemetry。
- app-server notification wrapper。

## 6. 恢复时的处理逻辑

恢复不是“读 JSONL 后直接发模型”。

```text
读取 JSONL
  -> 找 session_meta
  -> 处理 compacted checkpoint
  -> 应用 rollback
  -> 收集 response_item
  -> 恢复 turn_context baseline
  -> 重建 session history
  -> 下一次采样前再次过滤/截断/归一化
```

这也是为什么持久化模型要比 prompt 模型更丰富。

## 7. Web Word 持久化建议

Web Word agent 至少保留这些层：

```ts
type WebWordRolloutLine =
  | { type: "thread_meta"; value: WebWordThreadMeta }
  | { type: "turn_context"; value: WebWordTurnContext }
  | { type: "agent_item"; value: WebWordAgentItem }
  | { type: "document_change"; value: DocumentChangeRecord }
  | { type: "event"; value: WebWordAgentEvent }
  | { type: "compacted"; value: CompactedHistory };
```

建议保存：

- 用户原始请求。
- agent 的最终回答。
- 工具调用和工具结果。
- 文档 range、suggestion id、comment id。
- 用户是否接受/拒绝建议。
- 审批记录。
- compact checkpoint。
- rollback 事件。

不建议把所有 UI delta 都作为核心历史永久保存。可以把实时流和审计日志分开存。

