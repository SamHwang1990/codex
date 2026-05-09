# Agent “历史”问题归纳与答案

这份文件不是按对话时间顺序记录，而是把围绕“历史”的疑问归纳成几个核心问题，并给出当前结论。目标是帮助自研 Web Word agent 时建立清晰的数据模型。

## 核心问题 1：到底有几种“历史”？

### 问题

“历史”这个词同时出现在 thread、session、turn、prompt 里，容易混成一个东西：

- thread 本身有历史，是持久化的吗？
- session 本身有历史，是内存的吗？
- turn 里又有历史吗？
- prompt 里的历史是不是就是 JSONL 里的历史？

### 答案

至少要拆成四层：

| 名称 | 所在位置 | 本质 | 主要作用 | 是否等于模型 prompt |
| --- | --- | --- | --- | --- |
| thread history | thread JSONL / rollout log | append-only 持久化日志 | resume、fork、rollback、compact、审计 | 否 |
| session history | session 内存状态 | 当前运行中的候选上下文 | 快速追加、替换、rollback、估算 token | 否 |
| turn history | 本轮执行视图 | 某个 turn 启动时从 session history 派生出的历史基础 | 支撑本轮一次或多次模型采样 | 否 |
| prompt history | 模型请求里的历史片段 | 经过过滤、截断、归一化后的最终输入 | 真正发给模型 | 是，但只是 prompt 的一部分 |

简化关系：

```text
thread JSONL
  -> resume/replay
  -> session in-memory history
  -> turn context + current input + tools + instructions
  -> prompt-ready history
  -> model request
```

关键结论：

> JSONL 保存的是可恢复日志，不是最终 prompt。session 内存历史是恢复后的候选上下文，也不是最终 prompt。最终 prompt 是在每次采样前再次加工出来的。

## 核心问题 2：thread JSONL 里保存的到底是什么？

### 问题

如果要自己实现 agent，需要知道 thread JSONL 保存的历史数据结构是什么，每种结构来源是什么、含义是什么。

### 答案

thread JSONL 保存的不是单一 `messages[]`，而是一组按时间追加的 rollout line：

| JSONL 行类型 | 来源 | 含义 | 对模型历史的影响 |
| --- | --- | --- | --- |
| `session_meta` | thread 创建、fork、resume 初始化 recorder | thread 级元数据：thread id、cwd、来源、版本、基础指令、动态工具、git 信息等 | 不直接进入模型历史 |
| `response_item` | session 记录模型可见 item 时同步落盘 | 最接近“聊天历史”的结构：用户消息、上下文注入、助手消息、reasoning、工具调用、工具结果 | 会进入 session history，但之后还会被处理 |
| `turn_context` | 每个真实 user turn 计算完上下文后落盘；compact 后也可能落盘 | 本 turn 的运行配置快照：cwd、日期、时区、模型、权限、sandbox、指令、输出 schema、截断策略等 | 不作为聊天消息，但用于恢复上下文 baseline |
| `compacted` | 历史压缩完成时落盘 | compact 检查点；新版包含 `replacement_history` | 恢复时可直接替换旧历史 |
| `event_msg` | agent 发事件后，经持久化策略过滤落盘 | 可恢复/可审计事件：user message、agent message、turn start/complete、rollback、token usage、tool end 等 | 大多不进模型历史；rollback 会改变有效历史 |

更完整的 TypeScript schema 已整理在：

- `thread-jsonl-history-schema.md`

## 核心问题 3：为什么“历史不是原样发送”？

### 问题

直觉上会以为 JSONL 里保存了什么，下一次就把什么原样发给模型。但实际不是这样。

### 答案

因为 JSONL 是“恢复日志”，不是“模型请求体”。发给模型前会经历多步加工：

```text
读取 JSONL
  -> 根据 compact checkpoint 找历史替换点
  -> 根据 rollback 事件删除无效用户 turn
  -> 重建 session in-memory history
  -> 过滤不能作为 API message 的 item
  -> 截断过长工具输出
  -> 每次采样前归一化 call/output 配对
  -> 根据模型能力移除不支持的图片
  -> 合并当前 turn 输入、工具定义、指令、上下文更新
  -> 形成最终 model request
```

典型处理规则：

| 输入 | 处理 |
| --- | --- |
| `response_item.message(role="system")` | 不进入 session history |
| `response_item.other` | 不作为有效历史 |
| `function_call_output` / `custom_tool_call_output` | 进入历史前按截断策略裁剪 |
| 缺少工具输出的 call | 发送前补齐或归一化 |
| 没有对应 call 的工具输出 | 发送前移除 |
| 图片内容 | 如果模型不支持图片，发送前剥离 |
| `thread_rolled_back` 事件 | 不作为消息发送，但会删除最后 N 个用户 turn |

核心结论：

> 持久历史负责“可恢复”，内存历史负责“可操作”，prompt 历史负责“可发送”。三者目标不同，所以不能原样复用。

## 核心问题 4：每次 turn 启动时，agent 会给模型准备什么？

### 问题

一次 turn 启动时，模型输入不是只有用户当前输入。需要明确到底准备了哪些信息，以及这些信息来自哪里。

### 答案

turn 启动后，agent 会准备一组模型输入材料：

| 信息 | 来源 | 作用 |
| --- | --- | --- |
| base instructions | session meta / 模型管理器 | 定义 agent 基础行为 |
| developer/user instructions | turn context / 配置 | 注入持久规则、项目规则、用户偏好 |
| session history | session 内存状态 | 提供之前对话、工具调用、工具结果 |
| context updates | 当前 turn context 与上一基线的差异 | 告诉模型环境、权限、模型、日期等变化 |
| current user input | 本轮用户提交 | 当前任务目标 |
| tool definitions | 当前可用工具注册表 | 告诉模型可调用哪些工具 |
| permissions/sandbox/network | turn context | 约束工具调用行为 |
| output schema | turn context | 约束最终输出格式 |

这些信息最后组合成模型请求。`history` 只是其中一部分。

## 核心问题 5：首次采样和后续采样的“历史”有什么区别？

### 问题

同一个 turn 里可能有多次模型采样：第一次模型调用、工具执行后的第二次模型调用、再次工具执行后的第三次模型调用。每次的历史是否一样？

### 答案

不一样。

### 首次采样

首次采样通常包含：

- 之前 session history。
- 当前用户输入。
- 首轮或配置变化时的上下文注入。
- 当前可用工具定义。

如果这是 thread 的第一个真实 user turn，通常会注入完整上下文 baseline。

### 后续采样

后续采样通常在模型调用工具之后发生，会额外包含：

- 模型刚才发出的 tool call。
- 工具执行结果 tool output。
- 可能新增的警告、错误、权限结果。
- 经过截断后的工具输出。

流程可以理解为：

```text
用户输入
  -> 第一次采样
  -> 模型产生 tool call
  -> agent 执行工具
  -> 工具结果写入 session history
  -> 第二次采样
  -> 模型继续回答或继续调用工具
```

核心结论：

> 一个 turn 内的“历史”会随着工具调用和工具结果不断增长；后续采样不是重复第一次请求，而是在新历史基础上继续采样。

## 核心问题 6：session 和 thread 如何交互？

### 问题

thread 和 session 都像是在管理历史。它们之间的数据交互逻辑是什么？

### 答案

可以把职责拆开：

| 对象 | 职责 |
| --- | --- |
| thread | 持久身份、生命周期、JSONL 存储、resume/fork/archive/list |
| session | 运行时执行器、内存历史、turn 调度、模型采样、工具执行、事件发送 |

写入方向：

```text
session 运行 turn
  -> 产生 response_item / turn_context / compacted / event_msg
  -> append 到 thread JSONL
```

恢复方向：

```text
打开 thread
  -> 读取 JSONL
  -> reconstruct session history
  -> 恢复 turn context baseline
  -> 新 session 继续运行
```

rollback 方向：

```text
用户请求 rollback
  -> 不删除旧 JSONL 行
  -> 追加 thread_rolled_back event
  -> session 内存历史立即删除最后 N 个用户 turn
  -> 下次 resume 时根据 rollback event 重新计算有效历史
```

compact 方向：

```text
历史过长
  -> 生成 compact summary / replacement_history
  -> session 内存历史替换为压缩后历史
  -> thread JSONL 追加 compacted checkpoint
  -> 后续 resume 从 checkpoint 开始重建
```

核心结论：

> session 是“正在运行的脑内状态”，thread 是“可恢复的账本”。session 改变历史时，会把足够的信息追加到 thread；thread 恢复时，会重新算出 session history。

## 核心问题 7：UI 历史、事件历史、模型历史是不是同一个？

### 问题

JSONL 里有 `event_msg.user_message`，也有 `response_item.message(role="user")`。它们看起来都像用户消息，是否重复？

### 答案

不是同一个用途。

| 类型 | 面向对象 | 作用 |
| --- | --- | --- |
| `event_msg.user_message` | UI / replay / turn 分段 | 表示用户看见或提交了什么 |
| `response_item.message(role="user")` | 模型历史 | 表示模型应该看到的用户输入 |
| `event_msg.agent_message` | UI 展示 | 表示助手输出给用户看的文本 |
| `response_item.message(role="assistant")` | 模型历史 | 表示下一次采样时模型要承接的助手输出 |

它们可能内容相近，但不能互相替代。

核心结论：

> UI 展示历史和模型 prompt 历史要分层。UI 需要可读、可回放；模型需要结构化、可归一化、可压缩。

## 对 Web Word Agent 的设计答案

如果要把这个思想迁移到 Web Word 产品，建议不要只存 `messages[]`。更稳妥的数据模型是 append-only log + 可重建 session state：

```ts
type WebWordAgentLogLine =
  | { type: "thread_meta"; payload: ThreadMeta }
  | { type: "model_history_item"; payload: ModelHistoryItem }
  | { type: "turn_context_snapshot"; payload: TurnContextSnapshot }
  | { type: "history_checkpoint"; payload: HistoryCheckpoint }
  | { type: "recoverable_event"; payload: RecoverableEvent };
```

Web Word 特有的 `turn_context_snapshot` 建议包含：

| 字段 | 用途 |
| --- | --- |
| `documentId` | 当前文档身份 |
| `documentRevision` | 防止恢复后上下文和文档版本错位 |
| `selectionRange` | 用户当时选中的文本范围 |
| `visibleOutline` | 当前文档结构视图 |
| `activeComments` | 评论/批注上下文 |
| `permissionProfile` | 当前用户可编辑、可评论、可导出的权限 |
| `availableTools` | 可调用的文档工具，如 rewrite、insert、comment、format、search |
| `model` / `effort` | 本轮模型设置 |
| `locale` / `timezone` / `currentDate` | 写作和时间上下文 |

核心设计原则：

1. 持久层保存可恢复日志，不保存“最终 prompt 快照”作为唯一事实来源。
2. session 内存层保存当前有效历史，允许 compact、rollback、工具结果追加。
3. turn context 必须保存产品上下文快照，尤其是文档版本、选区、权限。
4. prompt 构造层每次采样前重新计算，不能直接复用 JSONL。
5. UI 历史和模型历史分开存或至少分开解释。
6. compact 应保存可替换历史，而不只是摘要文本。
7. rollback 应追加事件并重算有效历史，而不是物理删除历史日志。

## 已落盘的详细笔记

- `thread-jsonl-history-schema.md`：完整列出 thread JSONL 里保存的结构，并用 TypeScript 表达。
- `turn-domain-deep-dive.md`：解释 turn 启动、采样、工具调用、事件、完成/中断。
- `core-agent-ddd-closed-loops.md`：解释 thread、session、turn、tool call 等领域对象边界。
- `web-word-agent-implementation-blueprint.md`：把这些设计迁移到 Web Word agent 的实现蓝图。
