# Agent 领域模型整合

这一页把 Codex agent 的源码概念翻译成通用领域对象。自研 Web Word agent 时，不需要照搬命名，但建议保留这些边界。

## 1. 总体领域地图

```text
Client / UI
  -> Agent API
  -> Thread Manager
  -> Session
  -> Turn Runner
  -> Prompt Builder
  -> Model Client
  -> Tool Runtime
  -> Event Stream
  -> Persistence
```

每一层的职责应该清晰：

| 层 | 业务职责 | 不应该做的事 |
| --- | --- | --- |
| Client / UI | 展示对话、事件、审批、工具状态 | 不直接拼模型 prompt |
| Agent API | 提供 thread/turn/approval 等稳定入口 | 不暴露内部运行状态细节 |
| Thread Manager | 创建、恢复、归档、fork thread | 不直接执行模型采样 |
| Session | 管理运行时状态、内存历史、turn 调度 | 不承担 UI 展示逻辑 |
| Turn Runner | 执行一次用户目标 | 不持久拥有整个 thread 生命周期 |
| Prompt Builder | 组装每次采样请求 | 不把 JSONL 原样当 prompt |
| Model Client | 调模型、接收流式事件 | 不执行产品工具 |
| Tool Runtime | 执行工具、处理权限和结果 | 不决定最终回答文本 |
| Event Stream | 通知 UI 和记录过程 | 不等同于模型上下文 |
| Persistence | 保存可恢复日志和元数据 | 不要求每行都能直接发给模型 |

## 2. 核心对象

### Thread

Thread 是“可恢复的任务容器”。

它包含：

- thread id。
- 元数据。
- 持久化 rollout / JSONL。
- 可恢复的历史事件。
- fork、rollback、compact 的边界。

Thread 不应该等同于 `messages[]`。它保存的是运行过程，而不是只保存用户和助手文本。

```ts
type Thread = {
  id: string;
  metadata: ThreadMetadata;
  rollout: RolloutLine[];
  status: "active" | "archived";
};

type ThreadMetadata = {
  createdAt: number;
  cwd?: string;
  productContext?: unknown;
  source: "new" | "resume" | "fork";
};
```

### Session

Session 是“当前运行中的执行器”。

它通常是内存态，负责：

- 接收用户操作。
- 维护当前可用历史。
- 调度 active turn。
- 与模型、工具、事件、持久化协作。
- 从 thread JSONL 恢复运行态。

```ts
type Session = {
  threadId: string;
  history: ModelVisibleCandidateItem[];
  contextBaseline: TurnContext | null;
  activeTurn?: ActiveTurn;
  eventSubscribers: EventSink[];
};
```

### Turn

Turn 是“一次用户目标的执行闭环”。

一个 turn 可以包含多次模型采样和多次工具调用。

```ts
type Turn = {
  id: string;
  threadId: string;
  userGoal: UserMessage;
  context: TurnContext;
  status:
    | "running"
    | "waiting_for_tool"
    | "waiting_for_user"
    | "completed"
    | "interrupted"
    | "failed";
  items: TurnItem[];
};
```

### TurnContext

TurnContext 是本轮运行所需的上下文快照。

它不是聊天消息，而是告诉 prompt builder 和 tool runtime：

- 当前工作环境是什么。
- 模型和推理配置是什么。
- 权限和沙箱策略是什么。
- 可用工具和上下文源是什么。
- 当前日期、时区、产品上下文是什么。

```ts
type TurnContext = {
  environment: RuntimeEnvironment;
  model: ModelSelection;
  instructions: InstructionBundle;
  permissions: PermissionContext;
  tools: ToolContext;
  output?: OutputSchema;
  productContext?: ProductContext;
};
```

### Item

Item 是“运行过程中的结构化片段”。

它可能是用户消息、助手消息、模型 reasoning、工具调用、工具结果、上下文注入、压缩后的历史等。

关键是：

> item 有些会保存，有些会展示，有些会进入后续 prompt，这三者不是同一个集合。

```ts
type Item =
  | UserMessageItem
  | AssistantMessageItem
  | ReasoningItem
  | ToolCallItem
  | ToolOutputItem
  | ContextItem
  | CompactedHistoryItem;
```

### Event

Event 是“运行状态变化通知”。

它服务 UI、日志、恢复、审计，但多数 event 不直接进入模型历史。

```ts
type AgentEvent =
  | { type: "turn_started"; turnId: string }
  | { type: "item_started"; item: TurnItem }
  | { type: "item_delta"; itemId: string; delta: unknown }
  | { type: "item_completed"; item: TurnItem }
  | { type: "approval_requested"; request: ApprovalRequest }
  | { type: "turn_completed"; turnId: string; usage?: TokenUsage }
  | { type: "turn_failed"; turnId: string; error: AgentError };
```

## 3. 三条主链路

### 请求链路

```text
UI action
  -> Agent API
  -> Session operation queue
  -> Active turn
  -> Prompt builder
  -> Model client
```

### 工具链路

```text
Model tool call
  -> Tool router
  -> Tool runtime
  -> Permission / approval / sandbox
  -> Product service
  -> Tool output
  -> Follow-up sampling
```

### 事件链路

```text
Runtime state change
  -> Agent event
  -> UI notification
  -> selected persistence
  -> resume/replay if needed
```

## 4. 设计判断

自研 agent 时，最容易犯的错误是把所有东西做成一个 `Conversation` 对象。Codex 的启发是应该拆成：

| 不推荐 | 推荐 |
| --- | --- |
| 一个 `messages[]` 管全部 | thread/session/turn/item/event 分层 |
| UI 自己拼 prompt | 后端统一 prompt builder |
| 工具散落在业务代码里 | tool registry + tool runtime |
| 每次把全部历史发给模型 | 先恢复，再过滤、截断、归一化、注入 |
| 模型直接改产品数据 | 工具层做权限、审批、建议和审计 |

