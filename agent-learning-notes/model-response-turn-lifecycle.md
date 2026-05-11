# 模型响应处理和 Turn 生命周期

这份笔记解释：模型收到复合 prompt 之后，agent 能观察到什么、如何处理返回流、如何形成你日常看到的“推理、工具、plan、确认、turn 完成”等交互。

先说边界：我们无法从客户端代码知道模型内部神经网络“具体怎么想”。这里讨论的是 agent 和模型 API 之间的可观察协议，以及 Codex 如何把模型返回转成产品可用事件、历史和下一轮 prompt。

## 1. 总体流程

```text
模型收到结构化请求
  -> 按 role / instructions / input / tools / sampling config 生成响应
  -> 通过 stream 返回事件
  -> agent 解析 stream event
  -> 对每个 output item 判断：
       是普通内容？推理？plan？工具调用？
  -> 普通内容：发给 UI，记录历史
  -> 工具调用：记录工具请求，执行工具，记录工具结果
  -> 如果有工具结果或模型声明未结束：继续下一次采样
  -> 如果没有 follow-up：运行 stop / after-agent hooks
  -> 发出 turn complete
```

通用抽象：

```ts
interface ModelRuntimeLoop {
  input: PromptInputItem[];
  tools: ModelVisibleTool[];
  stream: AsyncIterable<ModelStreamEvent>;
  sessionHistory: ResponseItem[];
  uiEvents: ClientEvent[];
  needsFollowUp: boolean;
  lastAgentMessage?: string;
}
```

关键结论：

> 模型返回的不是一个“最终字符串”，而是一串事件。agent 把这些事件翻译成 UI 事件、历史 item、工具执行和下一次模型采样。

## 2. 模型如何“处理”复合 prompt

从 agent 角度看，模型会同时看到几类信息：

```ts
interface ModelVisibleRequest {
  instructions: string;
  input: PromptInputItem[];
  tools: ModelVisibleTool[];
  reasoning?: ReasoningRequest;
  text?: OutputTextControls;
}
```

模型通常按这些语义使用它们：

| 输入部分 | 模型侧语义 | 对输出的影响 |
| --- | --- | --- |
| `instructions` | 最基础、最高层的 agent 行为规则 | 决定回答风格、工具使用规则、完成标准 |
| developer message | 应用/开发者运行规则 | 约束权限、协作模式、skill 使用、工具策略 |
| contextual user message | 项目/环境上下文 | 提供 cwd、日期、项目指令、运行环境 |
| prior history | 之前对话、工具调用和工具结果 | 让模型保持连续性，避免重复工作 |
| current user message | 当前用户真正任务 | 决定本轮目标 |
| tools | 可执行动作的 schema | 让模型选择是否发起工具调用 |
| reasoning config | 推理强度和摘要可见性 | 影响是否返回 reasoning summary |
| output schema | 最终答案结构约束 | 影响 assistant message 的格式 |

模型输出时通常会选择一种或多种行为：

```ts
type ModelDecision =
  | { kind: "think"; output: ReasoningItem }
  | { kind: "say"; output: AssistantMessageItem }
  | { kind: "call_tool"; output: ToolCallItem }
  | { kind: "finish"; responseId: string; usage?: TokenUsage; endTurn?: boolean };
```

注意：这些不是模型显式返回的单个字段，而是 agent 根据 stream 里的 item 类型推断出的业务行为。

## 3. 返回流的基础结构

Codex 把 OpenAI Responses API 的 SSE/WebSocket 事件解析成内部 `ResponseEvent`。

```ts
type ModelStreamEvent =
  | { type: "created" }
  | { type: "output_item_added"; item: ResponseItem }
  | { type: "output_item_done"; item: ResponseItem }
  | { type: "output_text_delta"; delta: string }
  | { type: "tool_call_input_delta"; itemId: string; callId?: string; delta: string }
  | { type: "reasoning_summary_delta"; itemId?: string; summaryIndex: number; delta: string }
  | { type: "reasoning_content_delta"; itemId?: string; contentIndex: number; delta: string }
  | { type: "reasoning_summary_part_added"; itemId?: string; summaryIndex: number }
  | { type: "completed"; responseId: string; tokenUsage?: TokenUsage; endTurn?: boolean }
  | { type: "server_model"; model: string }
  | { type: "rate_limits"; snapshot: unknown };
```

每个事件的含义：

| 事件 | 含义 | agent 怎么处理 |
| --- | --- | --- |
| `created` | response 创建 | 通常只用于生命周期/指标 |
| `output_item_added` | 一个输出 item 开始 | 转成 `ItemStarted` UI 事件 |
| `output_text_delta` | assistant 文本增量 | 追加到当前 assistant item 展示 |
| `tool_call_input_delta` | 工具参数增量 | 展示工具参数逐步生成 |
| `reasoning_summary_delta` | 推理摘要增量 | 展示/记录 reasoning summary |
| `reasoning_content_delta` | 原始推理内容增量 | 仅在配置允许时展示 |
| `output_item_done` | 一个输出 item 完成 | 记录历史；若是工具调用则执行工具 |
| `completed` | 本次模型响应完成 | 更新 token usage，判断是否继续采样 |

## 4. 模型输出 item 结构

模型完成的输出 item 会落成 `ResponseItem`。这是“历史”和“后续采样”的核心材料。

```ts
type ResponseItem =
  | AssistantMessageResponseItem
  | ReasoningResponseItem
  | FunctionCallResponseItem
  | CustomToolCallResponseItem
  | LocalShellCallResponseItem
  | WebSearchCallResponseItem
  | ImageGenerationCallResponseItem
  | ToolSearchCallResponseItem
  | ToolOutputResponseItem
  | { type: "other" };

interface AssistantMessageResponseItem {
  type: "message";
  role: "assistant";
  content: Array<{ type: "output_text"; text: string }>;
  phase?: "commentary" | "final_answer";
}

interface ReasoningResponseItem {
  type: "reasoning";
  summary: Array<{ type?: string; text?: string }>;
  content?: Array<{ type: "reasoning_text" | "text"; text: string }>;
  encrypted_content?: string;
}

interface FunctionCallResponseItem {
  type: "function_call";
  name: string;
  namespace?: string;
  /**
   * 注意：这是 JSON 字符串，不是已经 parse 好的对象。
   */
  arguments: string;
  call_id: string;
}

interface CustomToolCallResponseItem {
  type: "custom_tool_call";
  call_id: string;
  name: string;
  input: string;
  status?: string;
}

interface LocalShellCallResponseItem {
  type: "local_shell_call";
  call_id?: string;
  status: string;
  action: unknown;
}

interface ToolSearchCallResponseItem {
  type: "tool_search_call";
  call_id?: string;
  status?: string;
  execution: string;
  arguments: unknown;
}

interface WebSearchCallResponseItem {
  type: "web_search_call";
  status?: string;
  action?: unknown;
}

interface ImageGenerationCallResponseItem {
  type: "image_generation_call";
  id: string;
  status: string;
  revised_prompt?: string;
  result: string;
}

type ToolOutputResponseItem =
  | {
      type: "function_call_output";
      call_id: string;
      output: string | FunctionCallOutputContentItem[];
    }
  | {
      type: "custom_tool_call_output";
      call_id: string;
      name?: string;
      output: string | FunctionCallOutputContentItem[];
    }
  | {
      type: "tool_search_output";
      call_id?: string;
      status: string;
      execution: string;
      tools: unknown[];
    };

type FunctionCallOutputContentItem =
  | { type: "input_text"; text: string }
  | { type: "input_image"; image_url: string };
```

## 5. UI 可见 TurnItem

UI 不一定直接展示 `ResponseItem`。Codex 会把其中一部分转成更适合 UI 的 `TurnItem`。

```ts
type TurnItem =
  | UserMessageItem
  | HookPromptItem
  | AgentMessageItem
  | PlanItem
  | ReasoningItem
  | WebSearchItem
  | ImageViewItem
  | ImageGenerationItem
  | FileChangeItem
  | McpToolCallItem
  | ContextCompactionItem;

interface AgentMessageItem {
  type: "AgentMessage";
  id: string;
  content: Array<{ type: "Text"; text: string }>;
  phase?: "commentary" | "final_answer";
  memoryCitation?: unknown;
}

interface ReasoningItem {
  type: "Reasoning";
  id: string;
  summaryText: string[];
  rawContent: string[];
}

interface PlanItem {
  type: "Plan";
  id: string;
  text: string;
}

interface McpToolCallItem {
  type: "McpToolCall";
  id: string;
  server: string;
  tool: string;
  arguments: unknown;
  status: "inProgress" | "completed" | "failed";
  result?: unknown;
  error?: { message: string };
  duration?: string;
}
```

UI 事件分两类：

```ts
type TurnItemLifecycleEvent =
  | { type: "item_started"; threadId: string; turnId: string; item: TurnItem; startedAtMs: number }
  | { type: "item_completed"; threadId: string; turnId: string; item: TurnItem; completedAtMs: number };

type TurnItemDeltaEvent =
  | { type: "agent_message_content_delta"; threadId: string; turnId: string; itemId: string; delta: string }
  | { type: "plan_delta"; threadId: string; turnId: string; itemId: string; delta: string }
  | { type: "reasoning_content_delta"; threadId: string; turnId: string; itemId: string; summaryIndex: number; delta: string }
  | { type: "reasoning_raw_content_delta"; threadId: string; turnId: string; itemId: string; contentIndex: number; delta: string };
```

## 6. 场景一：推理

推理有两种可见形态：

1. reasoning summary：模型返回的推理摘要，适合展示给用户。
2. raw reasoning content：更原始的推理内容，通常受配置控制，不一定展示。

结构：

```ts
interface ReasoningStream {
  started: {
    type: "item_started";
    item: ReasoningItem;
  };
  deltas: Array<
    | { type: "reasoning_content_delta"; summaryIndex: number; delta: string }
    | { type: "reasoning_raw_content_delta"; contentIndex: number; delta: string }
    | { type: "reasoning_section_break"; summaryIndex: number }
  >;
  completed: {
    type: "item_completed";
    item: ReasoningItem;
  };
  persistedHistoryItem: ReasoningResponseItem;
}
```

处理流程：

```text
模型开始 reasoning item
  -> output_item_added(reasoning)
  -> UI 收到 item_started
  -> reasoning_summary_delta / reasoning_content_delta 持续到达
  -> UI 增量展示
  -> output_item_done(reasoning)
  -> 记录 ResponseItem::Reasoning 到 session history
  -> UI 收到 item_completed
```

含义：

- reasoning 不是最终答复。
- reasoning item 进入历史后，后续采样可能会带上它，特别是 encrypted reasoning content。
- 是否展示 raw reasoning 取决于配置和 provider 返回。

## 7. 场景二：请求工具

工具调用是模型输出的一种 item，不是 agent 自己凭空决定的。模型通过工具 schema 选择工具名和参数。

```ts
type ToolCallItem =
  | FunctionCallResponseItem
  | CustomToolCallResponseItem
  | LocalShellCallResponseItem
  | ToolSearchCallResponseItem;

interface ToolExecutionCycle {
  modelToolCall: ToolCallItem;
  callId: string;
  parsedPayload: unknown;
  uiToolEvents: ClientEvent[];
  toolOutput: ToolOutputResponseItem;
  needsFollowUp: true;
}
```

处理流程：

```text
模型输出 tool call
  -> output_item_added(tool call)
  -> 可选：tool_call_input_delta 展示参数逐步生成
  -> output_item_done(tool call)
  -> agent 记录 tool call 到 history
  -> ToolRouter 根据 name/namespace/input 找 handler
  -> handler 执行真实动作
  -> 生成 function_call_output/custom_tool_call_output/tool_search_output
  -> agent 记录 tool output 到 history
  -> needs_follow_up = true
  -> 下一次采样把 tool call + tool output 发回模型
```

为什么工具后还要再次采样：

```text
第一次采样：模型说“我要调用工具”
agent 执行工具：得到结果
第二次采样：模型读取工具结果，再决定继续调用工具或给最终答复
```

工具输出结构：

```ts
interface FunctionToolOutput {
  type: "function_call_output";
  call_id: string;
  output: string | FunctionCallOutputContentItem[];
  success?: boolean;
}
```

含义：

- `call_id` 用来和模型之前的工具调用配对。
- `output` 是给模型看的工具结果，不一定等于 UI 展示内容。
- 工具失败也可能写成 output 发回模型，让模型修正参数或解释失败。

## 8. 场景三：生成 plan

这里要区分两种“plan”：

1. **Plan mode 里的 proposed plan**：模型在文本中生成计划，agent 从 assistant stream 里解析出计划块。
2. **Default mode 里的 update_plan 工具**：模型调用 `update_plan` 工具更新 checklist。

Plan mode 的 proposed plan 不是普通工具调用。它来自 assistant message 的文本流。

```ts
interface ProposedPlanLifecycle {
  planItemId: string;
  started: { type: "item_started"; item: PlanItem };
  deltas: Array<{ type: "plan_delta"; itemId: string; delta: string }>;
  completed: { type: "item_completed"; item: PlanItem };
}
```

处理流程：

```text
模型输出 assistant message
  -> output_text_delta 持续到达
  -> agent 的 stream parser 判断哪些文本是普通说明，哪些是 proposed plan
  -> 普通说明变成 agent_message_content_delta
  -> plan 内容变成 plan_delta
  -> response item 完成后，从完整 assistant message 再抽取最终 plan text
  -> 发出 item_completed(PlanItem)
```

含义：

- plan delta 是 UI 展示用增量。
- completed PlanItem 是最终 plan 文本。
- 这个 plan 通常不是发回模型的工具结果，而是当前 turn 的 UI/协作产物。

## 9. 场景四：更新 plan

`update_plan` 是一个工具。模型调用它时，agent 更新 UI checklist，并给模型回一个简单工具结果。

参数结构：

```ts
interface UpdatePlanArgs {
  explanation?: string;
  plan: Array<{
    step: string;
    status: "pending" | "in_progress" | "completed";
  }>;
}
```

处理流程：

```text
模型输出 function_call(name = "update_plan", arguments)
  -> agent parse arguments
  -> 发送 EventMsg::PlanUpdate(args) 给 UI
  -> 工具返回 function_call_output("Plan updated", success=true)
  -> 记录工具输出到 history
  -> needs_follow_up = true
  -> 下一次采样让模型继续工作或说明进展
```

事件结构：

```ts
interface PlanUpdateEvent {
  type: "plan_update";
  explanation?: string;
  plan: Array<{
    step: string;
    status: "pending" | "in_progress" | "completed";
  }>;
}
```

含义：

- `PlanUpdate` 是工具副作用事件，主要给 UI 更新 checklist。
- 工具结果 `"Plan updated"` 是给模型看的确认，不是用户最终答复。
- Codex 在 Plan mode 下禁止使用这个工具，避免和 proposed plan 机制混淆。

## 10. 场景五：请求用户确认/响应

这里也有两类：

1. **权限确认**：执行命令、写文件、申请网络等需要用户批准。
2. **业务问题输入**：模型调用 `request_user_input`，请用户回答问题。

### 10.1 权限确认

权限确认通常不是模型直接问用户一句话，而是工具运行时发现需要审批。

```ts
interface ApprovalRequestEvent {
  type: "exec_approval_request" | "apply_patch_approval_request" | "request_permissions";
  id: string;
  turnId: string;
  reason?: string;
  command?: string[];
  patch?: string;
  availableDecisions: string[];
}

interface ApprovalResponse {
  id: string;
  decision: "approved" | "approved_for_session" | "denied" | string;
}
```

处理流程：

```text
模型请求工具
  -> agent 准备执行工具
  -> sandbox/permission policy 判断需要用户批准
  -> agent 发 approval request UI event
  -> UI 等用户选择
  -> 用户批准：继续执行工具
  -> 用户拒绝：把拒绝/错误作为工具结果写回模型
```

含义：

- 模型只是请求工具；是否需要审批由 agent policy 决定。
- 审批结果会影响工具是否执行。
- 对模型来说，最后看到的是工具 output。

### 10.2 request_user_input

`request_user_input` 是模型可调用的工具，用来向用户询问结构化问题。

请求结构：

```ts
interface RequestUserInputArgs {
  questions: RequestUserInputQuestion[];
}

interface RequestUserInputQuestion {
  id: string;
  header: string;
  question: string;
  isOther?: boolean;
  isSecret?: boolean;
  options?: Array<{
    label: string;
    description: string;
  }>;
}
```

UI 事件：

```ts
interface RequestUserInputEvent {
  type: "request_user_input";
  call_id: string;
  turn_id: string;
  questions: RequestUserInputQuestion[];
}
```

用户响应：

```ts
interface RequestUserInputResponse {
  answers: Record<
    string,
    {
      answers: string[];
    }
  >;
}
```

处理流程：

```text
模型调用 request_user_input
  -> agent 校验当前 mode / root thread 限制
  -> agent 发 RequestUserInputEvent 给 UI
  -> active turn 内保存 pending response channel
  -> UI 收集用户答案
  -> client 发 UserInputAnswer(op)
  -> agent 唤醒等待中的 tool handler
  -> tool handler 把答案 JSON 序列化成 function_call_output
  -> needs_follow_up = true
  -> 下一次采样模型读取用户答案并继续
```

含义：

- 这不是 turn complete；turn 仍在等待用户输入。
- 用户回答不会作为普通 user message 直接插入当前 prompt，而是作为这次工具调用的 output 回给模型。
- 子 agent 通常不能直接调用 root UI 的 `request_user_input`。

## 11. 场景六：assistant 文本和最终回答

assistant 文本也是一个 output item。

```ts
interface AssistantMessageLifecycle {
  item: AssistantMessageItem;
  deltas: Array<{ type: "agent_message_content_delta"; delta: string }>;
  completed: boolean;
}
```

处理流程：

```text
output_item_added(message)
  -> UI 收到 item_started(AgentMessage)
output_text_delta
  -> UI 收到 agent_message_content_delta
output_item_done(message)
  -> agent 清理隐藏 markup / memory citation
  -> 记录 ResponseItem::Message 到 history
  -> UI 收到 item_completed(AgentMessage)
response.completed
  -> 如果没有工具和 pending input，turn 可以结束
```

`phase` 的含义：

```ts
type MessagePhase = "commentary" | "final_answer";
```

- `commentary`：中途说明，后面可能还有工具调用或最终答复。
- `final_answer`：本轮最终回答。
- `undefined`：provider 没有给 phase；agent 需要按兼容逻辑处理。

## 12. Turn 完整

turn complete 不是模型直接返回的 item，而是 agent 在采样循环收敛后发出的生命周期事件。

```ts
interface TurnCompleteEvent {
  type: "turn_complete";
  turn_id: string;
  last_agent_message?: string;
  completed_at?: number;
  duration_ms?: number;
  time_to_first_token_ms?: number;
}

interface TurnAbortedEvent {
  type: "turn_aborted";
  turn_id?: string;
  reason: "interrupted" | "replaced" | "review_ended" | "budget_limited" | string;
  completed_at?: number;
  duration_ms?: number;
}
```

判断逻辑：

```text
一次模型 response.completed
  -> 如果 end_turn === false：needs_follow_up = true
  -> 如果模型输出过工具调用：needs_follow_up = true
  -> 如果有 pending user input：needs_follow_up = true
  -> 如果 token limit 到达且还需要 follow-up：先 compact，再继续
  -> 如果 needs_follow_up = false：
       run stop hooks
       如果 hook 要求 continuation：写入 hook prompt，继续采样
       如果 hook 不阻塞：run after-agent hooks
       发 TurnComplete
```

注意：

- `response.completed` 只是“本次模型响应完成”。
- `turn_complete` 是“整个用户 turn 完成”。
- 一个 turn 可能包含多次模型采样和多次工具调用。

## 13. 六种日常场景的关系图

```text
用户发起 turn
  -> 首次采样
      -> 推理 item?
      -> assistant commentary?
      -> tool call?
          -> 权限确认?
          -> 工具执行
          -> tool output
          -> 后续采样
      -> request_user_input?
          -> UI 等用户回答
          -> user answer as tool output
          -> 后续采样
      -> plan?
          -> plan_delta / PlanItem
      -> update_plan?
          -> PlanUpdate event
          -> tool output
          -> 后续采样
      -> final assistant message?
  -> response.completed
  -> 如果无 follow-up
  -> turn_complete
```

## 14. 对 Web Word Agent 的抽象建议

建议把返回处理分成四层，避免 UI、工具和模型协议耦合：

```ts
interface ProviderStreamAdapter {
  parse(event: unknown): ModelStreamEvent | null;
}

interface ResponseInterpreter {
  onEvent(event: ModelStreamEvent): AgentRuntimeEffect[];
}

type AgentRuntimeEffect =
  | { type: "emit_ui_event"; event: ClientEvent }
  | { type: "record_history"; item: ResponseItem }
  | { type: "execute_tool"; call: ToolCallItem }
  | { type: "continue_sampling" }
  | { type: "complete_turn"; event: TurnCompleteEvent };

interface WebWordToolOutput {
  callId: string;
  success: boolean;
  visibleSummary?: string;
  modelOutput: string | object;
  documentRevision?: string;
}
```

Web Word 里可以映射成：

| Codex 场景 | Web Word 场景 |
| --- | --- |
| reasoning | 展示“正在分析文档/评论/选区”的过程摘要 |
| tool call | 读取文档、定位段落、生成修改、插入评论、应用建议 |
| plan | 生成编辑计划、审阅步骤、重写大纲 |
| update_plan | 更新左侧任务 checklist |
| request_user_input | 问用户选择语气、目标读者、是否保留原格式 |
| approval request | 应用文档修改前请求确认 |
| turn complete | 本次文档任务完成，释放输入框/更新状态 |

关键设计原则：

1. 模型输出只是“意图和内容”，不要让模型直接改文档状态。
2. 所有文档变更必须通过工具，并校验 document revision。
3. 用户确认和业务问题要分开：权限确认是安全策略，`request_user_input` 是任务信息补全。
4. turn complete 由 agent runtime 判定，不由模型一句“完成了”决定。
5. response completed 和 turn complete 要分开建模，否则工具链路会很难处理。

## 源码对照

| 主题 | 源码位置 |
| --- | --- |
| 模型 stream 事件定义 | `codex-rs/codex-api/src/common.rs` |
| SSE 事件解析 | `codex-rs/codex-api/src/sse/responses.rs` |
| 采样循环和 follow-up 判断 | `codex-rs/core/src/session/turn.rs` |
| output item 处理、工具调用判断、历史记录 | `codex-rs/core/src/stream_events_utils.rs` |
| TurnItem / UI item 数据结构 | `codex-rs/protocol/src/items.rs` |
| EventMsg / item delta / turn complete 结构 | `codex-rs/protocol/src/protocol.rs` |
| `update_plan` 工具 | `codex-rs/core/src/tools/handlers/plan.rs`, `codex-rs/protocol/src/plan_tool.rs` |
| `request_user_input` 工具 | `codex-rs/core/src/tools/handlers/request_user_input.rs`, `codex-rs/protocol/src/request_user_input.rs` |
