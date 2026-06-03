# Web WPS Agent 产品设计

## 状态

本文是 2026-06-03 确认的设计基线。

## 产品定位

Web WPS Agent 是运行在浏览器内的文档协作运行主干。用户通过自然语言表达目标，Agent 驱动模型请求、文档上下文、JSAPI 能力发现、权限审批、事件展示和本地历史保存，帮助用户完成文档编辑、分析、预览和修改。

当前阶段边界：

- Agent runtime 运行在浏览器。
- Node.js 服务只代理模型请求。
- 运行历史保存在浏览器 IndexedDB。
- Thread 只保存元数据。
- AgentStore 按 `threadId` 分区保存运行数据。
- JSAPI 只定义 delayed search 原则。具体执行协议、metadata、沙箱、dry-run、写入识别等能力，等接入真实 codebase 时再设计。
- 默认权限为 `requestApproval`。

## 核心关系

```text
Thread = metadata only
Agent = runtime trunk

ThreadStore[threadId]
  -> thread metadata

AgentStore[threadId]
  -> flattened runtime state
```

Thread 和 Agent 是平级概念：

| 概念 | 定义 |
| --- | --- |
| Thread | 元数据容器，只服务列表、筛选、归档、置顶、重命名、Fork、默认权限等产品能力。 |
| Agent | 唯一运行主干，持有并处理运行时数据。 |
| Session | 领域语言，表示 Agent 打开某个 Thread 后形成的运行视图。 |
| Turn | 领域语言，表示一次用户目标的执行过程。 |
| Item | 领域语言，表示模型可见历史里的结构化事实。 |
| EventMessage | 领域语言，表示 UI 和审计事件。 |
| Context | 领域语言，表示 Turn 开始时生成的上下文快照。 |

Session、Turn、Item、EventMessage、Context 是产品语义和 service 视图，不要求一一映射为 IndexedDB store 或 Agent state 顶层数组。底层运行数据按使用路径打平组织。

## Thread ID

```text
threadId = f(userIdentity, documentId, threadLocalId)
```

- `userIdentity` 用于隔离不同用户。
- `documentId` 绑定 Web WPS 文档。
- `threadLocalId` 支持同一用户、同一文档下存在多条 Thread。
- “我的全部 Thread”表示当前浏览器内，当前用户在所有文档下的本地 Thread。

## Thread Metadata

ThreadStore 只保存轻量元数据：

```ts
type ThreadMetadata = {
  threadId: string;
  userIdentity: string;
  documentId: string;
  title: string;
  archived: boolean;
  pinned: boolean;
  deleted: boolean;
  defaultPermission: "requestApproval" | "fullAccess";
  createdAt: number;
  updatedAt: number;
  lastTurnStatus?: "completed" | "running" | "waitingApproval" | "failed" | "interrupted";
  forkedFromThreadId?: string;
};
```

Thread 操作：

- Rename 只更新 metadata。
- Pin / Unpin 只更新 metadata。
- Archive / Unarchive 只更新 metadata。
- Delete 软删除 metadata，AgentStore 可延迟清理。
- Fork 生成新的 `threadId`，复制 metadata，并按实现阶段选定策略复制或引用 AgentStore 历史。

## Agent Runtime State

AgentStore 按 `threadId` 分区：

```ts
type AgentRuntimeState = {
  metadataRef: {
    threadId: string;
    userIdentity: string;
    documentId: string;
  };

  runState: {
    status: "idle" | "running" | "waitingApproval" | "failed" | "interrupted";
    activeTurnId?: string;
    activePhase?: string;
    lastError?: unknown;
  };

  modelVisibleHistory: ModelHistoryItem[];

  eventMessages: AgentEventMessage[];

  contextSnapshots: ContextSnapshot[];

  pendingInteractions: {
    approval?: ApprovalRequest;
    blockedReason?: string;
  };

  artifacts: {
    changePlans: ChangePlan[];
    changeSummaries: ChangeSummary[];
    citations: Citation[];
    previews: PreviewArtifact[];
  };

  attachments: AttachmentState[];

  runtimeConfig: {
    model?: string;
    reasoning?: string;
    defaultPermission: "requestApproval" | "fullAccess";
    currentTurnPermissionOverride?: "requestApproval" | "fullAccess";
  };

  caches: {
    jsapiSearchResults?: unknown;
    documentFragments?: unknown;
    capabilityDescriptions?: unknown;
  };
};
```

这个结构是说明性的，不是强制数据库 schema。实现时可以为了 IndexedDB 查询、恢复、性能或 compact 继续打平或归一化。

## Thread 列表 User Stories

- 作为用户，我打开 Web WPS 文档并点击 Sidebar 后，可以异步加载 Agent Thread 列表。
- 默认展示当前用户、当前文档、未归档 Thread。
- 我可以筛选当前文档 Thread、我的全部 Thread、已归档 Thread。
- 每个列表项展示 Thread 名称、上次对话时间、置顶状态、最近 Turn 状态。
- 每个列表项支持 Rename、Pin / Unpin、Archive / Unarchive、Fork、Delete。
- Fork 创建的新 Thread 仍绑定同一个 `documentId`，继承可见历史和默认权限，但后续 Turn 独立。
- Delete 在当前阶段是软删除。

Thread 列表只读取 ThreadStore，不加载完整 AgentStore 分区。

## Thread 详情 User Stories

初始化流程：

```text
用户打开 Thread
-> Agent 读取 Thread metadata
-> Agent 按 threadId 加载 AgentStore 分区
-> Agent 恢复 runState、history、events、pendingInteractions
-> UI 订阅 Agent event stream
-> 展示 Thread 详情
```

用户可见行为：

- 用户可以看到历史对话、执行过程、变更预览、审批卡片、错误和最终结果。
- 如果 `runState.status` 是 `waitingApproval`，恢复审批卡片。
- 如果 `runState.status` 是 `running`，展示执行中状态。
- 如果 `runState.status` 是 `failed`，展示失败原因和重试入口。
- 当前存在阻塞交互时，用户需要先处理阻塞交互；否则可以继续发起新 Turn。

## 输入栏

输入栏包含：

- 输入框。
- 发送按钮。
- Reasoning 设置。
- 权限设置。
- 文件管理：添加、上传状态、预览、删除。
- 后续 `/` 快捷指令入口。

权限设置默认使用 Thread 默认值，并允许当前 Turn 临时覆盖。

## 权限模型

```ts
type AgentPermission = "requestApproval" | "fullAccess";
```

| 权限 | 行为 |
| --- | --- |
| `requestApproval` | 默认值。任何写入真正应用前，Agent 必须生成 Turn 级 ChangePlan，并获得用户一次性确认。 |
| `fullAccess` | Agent 可以直接执行当前文档内的读写操作，结束后生成变更摘要。 |

规则：

- 默认权限来自 `ThreadMetadata.defaultPermission`。
- 输入栏允许当前 Turn 临时覆盖权限。
- Agent 需要在运行数据中记录本轮最终生效权限，用于恢复和审计。
- `requestApproval` 使用 Turn 汇总审批，不做逐个 JSAPI 调用审批。

## EventMessage UX

| EventMessage | UX |
| --- | --- |
| `agent_loaded` | Thread 详情数据加载完成。 |
| `turn_started` | 新增执行中状态。 |
| `user_message_added` | 展示用户输入。 |
| `assistant_message_delta` | 流式展示 Agent 回复。 |
| `reasoning_summary_delta` | 可折叠展示分析或规划摘要。 |
| `jsapi_search_started` | 展示 Agent 正在查找文档能力。 |
| `jsapi_search_completed` | 展示能力搜索摘要。 |
| `tool_call_started` | 展示 Agent 正在读取、分析或规划。 |
| `tool_call_completed` | 展示简短工具结果。 |
| `change_plan_created` | 展示 Turn 级变更预览。 |
| `approval_requested` | Turn 阻塞，等待用户确认或拒绝。 |
| `approval_resolved` | 展示用户批准或拒绝结果。 |
| `document_change_applied` | 展示应用结果。 |
| `turn_completed` | 输入栏恢复可用。 |
| `turn_failed` | 展示失败和重试入口。 |
| `turn_interrupted` | 展示执行已停止。 |

EventMessage 主要服务 UI 和审计，不等同于模型历史。

## Model Visible History

`modelVisibleHistory` 保存会进入后续 Prompt 的结构化事实：

- User message。
- Assistant message。
- Reasoning summary。
- Tool call。
- Tool output。
- Change plan。
- Approval response。
- Compact summary。

原则：

- UI event 不直接进入模型历史。
- 工具进度不直接进入模型历史。
- Prompt 构建时会筛选、截断、归一化 `modelVisibleHistory`。
- 长历史后续可以 compact 成 summary。

## Prompt 构建

Agent 每次模型采样都临时构建 Prompt，不直接发送 IndexedDB 原始历史。

Prompt 输入按以下顺序组装：

1. System：Agent 身份、文档协作目标、安全边界。
2. Development：Web WPS 场景规则、权限语义、审批规则、JSAPI delayed search 原则。
3. Runtime config：模型、reasoning、权限。
4. Context snapshot：文档 id、文档版本、选区、视口、附件引用。
5. Model visible history：筛选后的历史事实。
6. JSAPI capability search：只暴露 delayed search 入口，不暴露完整 JSAPI 目录。
7. Current user message：当前 Turn 目标。
8. Output requirement：最终回答、ChangePlan 或变更摘要。

## JSAPI 原则

当前阶段只定义抽象原则：

- JSAPI 面很大，不一次性塞进 Prompt。
- Agent 使用 delayed search，从宽到精细发现能力。
- 模型基于接口描述判断如何组合 API。
- 具体接口 metadata、沙箱、dry-run、写入识别和执行协议，后续接入真实 codebase 时再设计。
- `requestApproval` 下，写入前必须形成 ChangePlan。
- `fullAccess` 下，可以直接写入，但必须生成事件和变更摘要。

## Turn 流程

`requestApproval`：

```text
用户发送消息
-> Agent 设置 runState.running
-> Agent 创建 context snapshot
-> Agent 将 user message 写入 modelVisibleHistory
-> Agent 构建 Prompt
-> 模型搜索 JSAPI 能力
-> Agent / 模型进行只读分析
-> Agent / 模型生成 ChangePlan
-> Agent 写入 pendingInteractions.approval
-> Agent 发出 approval_requested
-> 用户确认
-> Agent 应用变更
-> Agent 写入 change summary
-> Agent 完成 Turn
```

`fullAccess`：

```text
用户发送消息
-> Agent 设置 runState.running
-> Agent 创建 context snapshot
-> Agent 构建 Prompt
-> 模型搜索 JSAPI 能力
-> Agent 执行读取和写入
-> Agent 写入 change summary
-> Agent 完成 Turn
```

用户拒绝审批：

```text
approval rejected
-> Agent 写入 approval_response
-> Agent 清理 pendingInteractions
-> 模型继续给出替代方案或结束 Turn
```

## 示例：替换 foo 为 bar

用户说：“帮我把文档所有 foo 替换为 bar。”

```text
Turn started
-> delayed search: replace / search / range / insert
-> Agent 选择合适 API 组合
-> requestApproval: 生成 ChangePlan，展示 N 处替换
-> 用户确认后应用
-> fullAccess: 直接应用并展示摘要
```

## 示例：试卷题目自动编号

用户说：“将试卷文档里所有题目做自动编号。”

```text
Turn started
-> delayed search: 文档结构 / 段落 / 样式 / 编号
-> Agent 识别题目段和已有编号
-> Agent 生成编号策略
-> requestApproval: 展示 ChangePlan
-> 用户确认后应用
-> Agent 展示编号结果摘要
```

## 设计决策摘要

- Thread 只保存元数据。
- Agent 是运行主干。
- AgentStore 按 `threadId` 分区。
- 运行数据按使用路径打平，不强制拆成 Session、Turn、Item、Event、Context 表。
- Session、Turn、Item、EventMessage、Context 保留为领域语言和 service 视图。
- 权限模型是 Thread 默认值加当前 Turn 临时覆盖。
- 审批模型是 Turn 级汇总审批。
- JSAPI 细节等接入真实 codebase 时再设计。
