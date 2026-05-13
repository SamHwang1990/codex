# Turn 与历史整合

这一页整合本次学习中最关键的概念问题：thread、session、turn、prompt 里都出现“历史”，但它们不是同一个东西。

## 1. Turn 的本质

Turn 不是一条消息。

Turn 是一次用户目标从启动到结束的运行闭环：

```text
用户输入
  -> 创建 turn
  -> 读取 session history
  -> 构造 turn context
  -> 首次采样
  -> 处理模型输出
  -> 如果有工具调用，执行工具
  -> 工具结果进入历史
  -> 后续采样
  -> 完成、等待、中断或失败
```

一个 turn 内可能发生：

- 多次模型请求。
- 多个工具调用。
- 多个工具结果。
- 多次 UI 事件更新。
- 用户审批或补充输入。
- 运行上下文变化。

所以 Web Word agent 里也应该把“用户点一次改写”建模成 turn，而不是只建模成一条 chat message。

## 2. 四层历史

| 名称 | 所在位置 | 本质 | 用途 | 是否原样发给模型 |
| --- | --- | --- | --- | --- |
| thread history | 持久化 JSONL / rollout | append-only 恢复日志 | resume、fork、rollback、compact、审计 | 否 |
| session history | 内存 | 当前运行候选上下文 | 追加 item、裁剪、替换、估算 token | 否 |
| turn history | 本轮视图 | turn 启动时从 session 派生 | 支撑本轮多次采样 | 否 |
| prompt history | 模型请求 | 过滤加工后的最终输入 | 让模型理解上下文 | 是 |

简化关系：

```text
thread JSONL
  -> replay / compact / rollback
  -> session in-memory history
  -> turn context + current user input
  -> prompt-ready input
  -> model request
```

## 3. 为什么历史不能原样发送

持久化日志的目标是“能恢复过程”，模型输入的目标是“能继续推理”。目标不同，所以必须处理。

典型处理：

| 来源数据 | 处理逻辑 |
| --- | --- |
| thread JSONL | 先 replay，不能整文件当 prompt |
| compacted checkpoint | 可能替换早期历史 |
| rollback event | 删除被撤销的 turn 影响 |
| session_meta | 恢复 thread 元信息，不作为聊天消息 |
| turn_context | 恢复上下文 baseline，不作为普通消息 |
| response_item | 进入候选历史，但仍需过滤和归一化 |
| event_msg | 多数只用于 UI/审计，不进入模型历史 |
| 工具输出 | 可能截断，避免 prompt 过大 |
| 图片/附件 | 根据模型能力决定是否保留 |
| 工具 call/output | 发送前要保证配对关系合理 |

核心结论：

> 保存历史是为了未来能重建状态；发送历史是为了当前模型能继续工作。

## 4. 首次采样与后续采样

### 首次采样

首次采样通常包含：

- 之前 session history。
- 当前用户消息。
- 本轮上下文注入。
- 当前可用工具定义。
- 推理和输出配置。

```text
prior history
  + initial/diff context
  + current user message
  + tools
  + inference/output config
  -> first model request
```

### 后续采样

后续采样发生在工具调用、用户补充、上下文变化之后。

它会在已有历史基础上追加：

- 模型刚才发出的 tool call。
- 工具执行结果。
- 权限拒绝或错误信息。
- 新的上下文 diff。

```text
first model request
  -> model emits tool call
  -> agent executes tool
  -> tool output appended
  -> next model request includes call + output
```

## 5. Turn 期间的状态变化

可以用这个通用状态机理解：

```text
idle
  -> running
  -> sampling
  -> handling_model_items
  -> executing_tools
  -> sampling
  -> completed

running
  -> waiting_for_user
  -> running

running
  -> interrupted

running
  -> failed
```

状态不是为了 UI 好看，而是为了决定：

- 是否允许新 turn。
- 是否允许 steer / 补充输入。
- 是否需要等待审批。
- 是否还有后续采样。
- 哪些 item 要保存。
- 哪些事件要广播。

## 6. Web Word 对应设计

Web Word agent 可以这样建模：

```ts
type DocumentTurn = {
  id: string;
  threadId: string;
  userGoal: string;
  documentSnapshotId: string;
  selection?: DocumentRange;
  status: TurnStatus;
  historyView: PromptItem[];
  pendingApprovals: ApprovalRequest[];
};

type DocumentRange = {
  blockId: string;
  start: number;
  end: number;
};

type TurnStatus =
  | "running"
  | "waiting_for_user"
  | "waiting_for_approval"
  | "completed"
  | "interrupted"
  | "failed";
```

建议：

- thread 代表一个长期文档协作会话。
- session 代表当前打开文档后的运行时。
- turn 代表一次写作、改写、审阅或格式调整任务。
- prompt history 只放和当前任务相关的文档片段、批注、版本摘要和历史工具结果。
- 持久化历史保留完整过程，支持恢复和审计。

