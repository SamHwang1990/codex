# 纯前端 Agent 技术、架构与产品交互方案

本文基于 `agent-learning-notes` 中对 Codex agent 的学习结果，设计一个可以嵌入 Web Word 类产品的纯前端 agent。这里的“纯前端”指 agent runtime、核心调度、工具执行、事件流、持久化和交互状态都运行在浏览器内；模型请求可以走企业模型网关或用户自带 key，但浏览器侧仍把它抽象成 `ModelClient`，不依赖后端 agent runtime。

## 1. 设计目标与边界

目标不是在文档右侧放一个聊天框，而是做一个浏览器内的文档协作 agent：

```text
用户目标
-> 读取文档、选区、光标、权限和协作上下文
-> 由前端 AgentCore 生成 turn context
-> 调模型并消费流式响应
-> 调用受控文档工具创建建议、批注、引用或预览
-> 通过事件流更新 UI
-> 用户审批后再落到正文或协作文档状态
-> 在 IndexedDB 保存可恢复的 thread/turn/item 过程
```

明确边界：

- 浏览器内不暴露底层数据库、任意 HTTP 或任意脚本执行能力。
- 默认优先生成 suggestion/comment，谨慎直接改正文。
- 模型只能请求工具；工具是否执行由前端权限策略、文档协作权限和用户审批共同决定。
- 历史完整保存，但每次模型输入由 `PromptBuilder` 临时筛选，不把 IndexedDB 日志等同于 prompt。
- 纯前端不保存企业密钥。企业场景建议使用模型网关；个人或内测场景可支持用户自带 key。

## 2. 从 Codex 学到的可迁移原则

参考依据：

- `codex-agent-learning-synthesis/02-agent-domain-model.md`：抽象 thread、session、turn、item、event、tool runtime 的领域边界。
- `core-agent-ddd-closed-loops.md`：说明核心 agent 层的 DDD 闭环和聚合关系。
- `agent-design-tradeoffs.md`：总结 app-server 边界、SQ/EQ、TurnContext、ToolRouter、持久化等取舍。
- `web-word-agent-implementation-blueprint.md`：把 Codex agent 能力迁移到 Web Word 文档、选区、建议、批注和权限场景。
- `turn-tool-discovery-call-model-flow.md`：支撑工具发现、调用、审批、结果回灌和后续采样设计。

| Codex 设计 | 纯前端迁移方式 |
| --- | --- |
| Thread / Session / Turn / Item 分层 | 保留分层，但收敛成前端内存对象和 IndexedDB 记录 |
| app-server 作为产品入口边界 | 简化成 `AgentFacade`，给 UI 暴露稳定方法 |
| SQ/EQ 异步事件模型 | 前端用 `AgentEventBus` 或 observable stream 承载 |
| TurnContext 快照 | 每次用户请求生成不可变 `TurnContext` |
| ToolRouter + ToolRuntime | 文档工具统一注册、校验、审批、执行和回灌 |
| 完整 rollout 持久化 | IndexedDB 保存 thread/turn/item/event 摘要，支持恢复与审计 |
| 抵制 core 膨胀 | Core 只调度，文档、权限、模型、存储、UI 都通过端口接入 |

对纯前端要主动简化：

- 不照搬 app-server JSON-RPC，UI 可以直接调用 typed facade。
- 不引入多层 bounded context 目录嵌套，用扁平模块表达 DDD 概念。
- MVP 不做多 agent、fork、复杂 compact、外部 MCP、shell、沙箱。
- 审批和权限先围绕文档副作用建模，不做通用系统权限模型。

## 3. 推荐方案与备选方案

### 推荐：浏览器内 AgentCore + 端口适配器

把 agent runtime 放在前端独立包里，核心只依赖接口，不依赖具体 UI、编辑器、模型 SDK 或存储实现。

收益：

- 最符合纯前端目标，离文档选区、编辑器事务和 UI 状态最近。
- Core 可以保持 DDD 的业务对象，又不会被具体产品 API 污染。
- 单元测试可以用 fake model、fake document tools、memory store 覆盖 turn 闭环。

代价：

- 大文档检索、模型请求和 IndexedDB 事务都要控制性能。
- 浏览器环境不适合重型 embedding、复杂权限策略和企业密钥管理。

### 备选 A：UI 直接调用模型和文档 API

收益是实现最快，适合原型验证。代价是 prompt、工具、权限、事件和持久化会散落在组件里，后期很难做恢复、审批、审计和测试。不推荐用于正式架构。

### 备选 B：后端 Agent Runtime + 前端 UI

收益是密钥、长任务、检索、审计更容易集中治理。代价是偏离“纯前端 agent”，而且文档编辑器的实时选区、协作事务和 inline suggestion 仍要在前端做复杂映射。可作为企业增强版演进方向，不作为本文主方案。

## 4. 总体技术架构

```text
Web Word UI
  -> AgentFacade
  -> AgentCore
      -> ThreadStorePort
      -> ModelClientPort
      -> PromptBuilder
      -> ToolRegistry
      -> ToolRuntime
      -> ApprovalPolicy
      -> EventBus
  -> Adapters
      -> EditorAdapter
      -> DocumentSearchAdapter
      -> SuggestionAdapter
      -> CommentAdapter
      -> IndexedDbThreadStore
      -> ModelGatewayClient
```

职责说明：

| 模块 | 职责 | 不应该做的事 |
| --- | --- | --- |
| `AgentFacade` | 给 UI 暴露 `startThread`、`startTurn`、`interruptTurn`、`approve`、`subscribe` | 不拼 prompt、不直接改文档 |
| `AgentCore` | 串联 thread/session/turn/tool/model/event/store | 不依赖 React、编辑器 SDK 或具体模型 SDK |
| `PromptBuilder` | 根据 turn context 和历史构造模型输入 | 不读取 UI 组件状态 |
| `ToolRegistry` | 声明本轮可见工具和 schema | 不执行副作用 |
| `ToolRuntime` | 校验、审批、执行文档工具并生成工具结果 | 不决定最终回答文本 |
| `EventBus` | 向 UI 分发 turn、item、tool、approval、error 事件 | 不作为模型历史来源 |
| `ThreadStorePort` | 保存 thread/turn/item 和恢复需要的元数据 | 不保存不可控大对象 |
| `EditorAdapter` | 把产品编辑器能力转成文档工具 | 不暴露内部编辑器事务给模型 |

## 5. 扁平 DDD Core 设计

核心仍保留 DDD 的聚合、实体、值对象和领域服务，但不用深层目录和多级 domain 嵌套。建议包结构：

```text
agent/
  core/
    agent-core.ts
    agent-facade.ts
    thread.ts
    session.ts
    turn.ts
    item.ts
    event.ts
    context.ts
    prompt-builder.ts
    tool-registry.ts
    tool-runtime.ts
    approval-policy.ts
    errors.ts
    ports.ts
  adapters/
    editor-adapter.ts
    document-search-adapter.ts
    indexeddb-thread-store.ts
    model-gateway-client.ts
  ui/
    agent-panel.tsx
    suggestion-card.tsx
    approval-dialog.tsx
```

核心对象保持扁平关系：

```text
AgentCore
  owns SessionRegistry
  starts Thread
  starts Turn
  builds TurnContext
  calls ModelClient
  routes ToolCall
  emits AgentEvent
  persists Item
```

### 5.1 聚合与实体

`Thread` 是持久工作线：

```ts
type Thread = {
  id: string;
  documentId: string;
  title?: string;
  status: "active" | "archived";
  createdAt: number;
  updatedAt: number;
  lastDocumentVersion: string;
};
```

`Session` 是运行时容器：

```ts
type Session = {
  threadId: string;
  activeTurnId?: string;
  historyCursor?: string;
  runtimeStatus: "idle" | "running" | "waitingForApproval" | "interrupted";
};
```

`Turn` 是一次用户目标的执行闭环：

```ts
type Turn = {
  id: string;
  threadId: string;
  userGoal: string;
  context: TurnContext;
  status: "running" | "waitingForApproval" | "completed" | "interrupted" | "failed";
  createdAt: number;
  completedAt?: number;
};
```

`Item` 是过程事实：

```ts
type AgentItem =
  | { type: "userMessage"; id: string; turnId: string; text: string }
  | { type: "assistantMessage"; id: string; turnId: string; text: string }
  | { type: "toolCall"; id: string; turnId: string; call: ToolCall }
  | { type: "toolResult"; id: string; turnId: string; callId: string; result: unknown }
  | { type: "approvalRequest"; id: string; turnId: string; request: ApprovalRequest }
  | { type: "documentSuggestion"; id: string; turnId: string; suggestionId: string };
```

### 5.2 值对象

`TurnContext` 是本轮快照：

```ts
type TurnContext = {
  documentId: string;
  documentVersion: string;
  selection?: DocumentRange;
  viewport?: DocumentRange;
  userRole: "owner" | "editor" | "commenter" | "viewer";
  locale: string;
  mode: "ask" | "rewrite" | "review" | "format" | "template";
  outputPolicy: "chatOnly" | "suggestionFirst" | "commentOnly" | "directEditWithApproval";
  availableTools: string[];
};
```

`ToolCall` 是模型请求副作用的边界：

```ts
type ToolCall = {
  id: string;
  name: string;
  input: unknown;
  approval: "notRequired" | "required" | "denied";
};
```

### 5.3 领域服务

`AgentCore` 是唯一调度中心：

```ts
interface AgentCore {
  startThread(input: StartThreadInput): Promise<Thread>;
  startTurn(input: StartTurnInput): Promise<Turn>;
  interruptTurn(turnId: string): Promise<void>;
  respondApproval(input: ApprovalResponse): Promise<void>;
  subscribe(listener: (event: AgentEvent) => void): () => void;
}
```

`AgentCore` 的内部流程：

```text
startTurn
-> load/create session
-> snapshot document context
-> create Turn + userMessage item
-> build prompt input
-> call model stream
-> emit assistant deltas or tool call events
-> ToolRuntime execute or request approval
-> append tool result
-> resample until final answer
-> complete turn and persist summary
```

## 6. 工具体系

MVP 只暴露文档业务语义工具：

```ts
type DocumentToolName =
  | "document.getSelection"
  | "document.getTextRange"
  | "document.search"
  | "document.createSuggestion"
  | "document.addComment"
  | "document.previewBatch"
  | "document.applySuggestion"
  | "document.applyBatch";
```

工具审批策略：

| 工具 | 默认策略 | 说明 |
| --- | --- | --- |
| `document.getSelection` | 自动允许 | 只读当前选区 |
| `document.getTextRange` | 自动允许 | 限制在当前文档 |
| `document.search` | 自动允许 | 返回带 range 的片段 |
| `document.createSuggestion` | 自动允许或轻审批 | 不直接改正文 |
| `document.addComment` | 可按租户策略审批 | 会产生协作可见副作用 |
| `document.previewBatch` | 自动允许 | 只生成预览 |
| `document.applySuggestion` | 必须用户确认 | 直接影响正文 |
| `document.applyBatch` | 必须用户确认 | 直接影响多处正文或批注 |

工具结果必须带稳定定位信息：

```ts
type DocumentToolResult = {
  ok: boolean;
  message?: string;
  range?: DocumentRange;
  citationId?: string;
  suggestionId?: string;
  commentId?: string;
  previewId?: string;
};
```

## 7. Prompt 与历史策略

每次采样只使用必要上下文：

| 场景 | 注入内容 |
| --- | --- |
| 改写选区 | 选区文本、上下文段落、文档语气、用户要求 |
| 长文档问答 | 检索片段、目录、引用 range、历史追问 |
| 审阅批注 | 审阅标准、目标范围、已有批注、风险等级 |
| 格式调整 | 结构树、样式信息、预览范围、可用格式工具 |
| 模板填充 | 模板字段、字段来源、缺失字段、输出格式 |

历史分三层：

- `AgentItem`：完整过程事实，用于审计和恢复。
- `PromptHistoryItem`：筛选后的模型可见历史，用于继续对话。
- `UiEvent`：实时体验事件，用于渲染，不反向污染模型历史。

## 8. 产品功能交互方案

### 8.1 入口

入口应该贴近文档工作流，而不是只放全局聊天：

- 选中文本后的浮动按钮：改写、缩短、扩写、润色、翻译。
- 右侧 agent 面板：长文档问答、审阅、结构调整、模板填充。
- 批注区入口：解释批注、生成回复、合并建议。
- 文档顶部命令入口：总结全文、生成目录、统一格式、导出前检查。

### 8.2 Agent 面板

面板由四块组成：

1. 输入区：自然语言目标、模式切换、当前选区提示。
2. 过程区：显示正在读取文档、搜索片段、创建建议、等待确认等事件。
3. 结果区：展示最终回答、引用、建议卡片、批注卡片。
4. 操作区：接受、拒绝、应用到正文、插入为批注、复制、继续追问。

面板不展示底层 prompt、token 或技术状态，除非进入调试模式。

### 8.3 Inline suggestion

改写类任务默认走 inline suggestion：

```text
用户选中段落
-> 点击“润色”
-> agent 读取选区和上下文
-> 创建 suggestion
-> 文档内显示修订建议
-> 用户接受、拒绝或继续要求“再正式一点”
```

关键交互规则：

- suggestion 必须绑定 range 和原始文档版本。
- 如果文档已被协作者修改，应用前提示冲突并要求重新生成。
- suggestion 卡片展示理由，但正文只展示修改差异。

### 8.4 长文档问答

问答类任务默认不改文档：

```text
用户提问
-> agent 搜索文档
-> 回答带引用
-> 用户点击引用跳转到原文
-> 可选择“把这个回答插入为摘要”或“给相关段落加批注”
```

关键交互规则：

- 每个结论尽量带 citation。
- 没有证据时回答“不确定”，并给出需要用户补充的信息。
- 回答和引用分开存储，避免后续文档改动导致旧引用误导。

### 8.5 审阅与批注

审阅类任务输出 findings：

```text
用户选择“审阅合同风险”
-> agent 读取范围、规则和已有批注
-> 输出风险列表
-> 用户勾选要落文档的 findings
-> agent 创建 comments 或 suggestions
```

finding 建议字段：

```ts
type ReviewFinding = {
  id: string;
  severity: "high" | "medium" | "low";
  title: string;
  evidenceRange: DocumentRange;
  explanation: string;
  recommendedAction: "comment" | "suggestion" | "none";
};
```

### 8.6 批量修改与审批

批量修改必须先预览：

```text
agent 生成 batch preview
-> UI 展示影响范围和变更摘要
-> 用户逐项勾选
-> applySuggestion / applyBatch 需要确认
-> 完成后写入 item 和审计摘要
```

审批弹窗应回答三个问题：

- 将修改哪里。
- 会产生什么副作用。
- 如何撤销或回滚。

## 9. 状态、持久化与恢复

浏览器内持久化建议用 IndexedDB：

| Store | 内容 |
| --- | --- |
| `threads` | thread 元数据、文档 id、状态、更新时间 |
| `turns` | turn 状态、用户目标、context 摘要、完成时间 |
| `items` | 用户消息、工具调用、工具结果、建议、批注、错误 |
| `events` | 可选保存关键事件，用于恢复过程视图 |
| `approvals` | 未决审批和处理结果 |

恢复策略：

- 页面刷新后恢复 thread list 和最近 turn 摘要。
- 未完成 turn 标记为 `interrupted`，不自动继续执行。
- 未决 approval 恢复为可重新确认或取消。
- 文档版本不一致时，旧 suggestion 进入“需要重新校验”状态。

## 10. 错误处理与降级

| 错误 | 用户体验 | Core 行为 |
| --- | --- | --- |
| 模型流失败 | 显示可重试 | 保存失败 item，允许 retry turn |
| 文档版本冲突 | 提示重新生成 | 拒绝直接 apply，保留 suggestion |
| 工具参数非法 | 显示 agent 修正中 | 把结构化错误回灌模型 |
| 权限不足 | 告知需要编辑权限 | 不请求模型继续执行副作用 |
| IndexedDB 失败 | 继续当前 turn 但提示无法恢复 | 降级到内存 store |
| 用户中断 | 停止生成并保留已完成结果 | abort stream，turn 标记 interrupted |

## 11. MVP 路线

### 阶段 1：单 turn 选区改写

- `AgentFacade.startTurn`
- `TurnContext` 快照
- `document.getSelection`
- `document.createSuggestion`
- 基础事件流
- IndexedDB 保存 thread/turn/item

### 阶段 2：长文档问答

- `document.search`
- citation/range 展示
- prompt history 筛选
- 回答插入为摘要

### 阶段 3：审阅与批注

- review finding 结构化输出
- `document.addComment`
- finding 勾选落文档
- 批注审计记录

### 阶段 4：批量修改与审批

- `document.previewBatch`
- approval flow
- apply 前版本校验
- 回滚或撤销入口

### 阶段 5：企业增强

- 模型网关策略
- 组织级权限配置
- 共享 thread
- 文档外知识库检索
- 轻量多 agent 或后台 worker

## 12. 测试与验收

核心测试优先覆盖闭环，而不是组件快照：

- `startTurn` 会创建 thread/turn/user item，并发出 `turnStarted`。
- 模型返回 tool call 时，`ToolRuntime` 执行文档工具并写入 tool result。
- 需要审批的工具不会绕过 approval。
- 文档版本冲突时不能直接 apply。
- 页面刷新后能恢复 completed turn 和 interrupted turn。
- prompt builder 不把完整 IndexedDB history 原样塞进模型输入。

产品验收标准：

- 用户能从选区触发改写，并在文档内看到可接受/拒绝的 suggestion。
- 用户能对全文提问，并点击引用跳转到原文。
- 用户能把审阅 finding 转成批注。
- 批量修改有预览和确认，不会默认直接改正文。
- 刷新页面后能看到最近 thread 和 turn 结果。

## 13. 结论

纯前端 agent 的核心不是“前端直接调模型”，而是在浏览器内保留完整 agent 闭环：thread 管历史，session 管运行态，turn 管一次目标，item 管过程事实，tool 管产品副作用，event 管实时交互。设计上应保留 Codex 的 DDD 边界，但用扁平模块减少层级，用 `AgentCore` 串联调度，把具体文档能力、模型调用、存储和 UI 都放到端口适配器后面。
