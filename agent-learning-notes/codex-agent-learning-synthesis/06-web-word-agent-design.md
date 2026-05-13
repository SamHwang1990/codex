# Web Word Agent 设计整合

这一页把 Codex agent 的学习结果迁移成 Web Word agent 的设计建议。

## 1. 产品目标

Web Word agent 的目标不是“聊天机器人嵌入文档”，而是：

> 一个能理解文档上下文、通过受控工具生成建议/批注/改写/结构调整，并能被用户审计、接受、拒绝和恢复的文档协作 agent。

## 2. 推荐总体架构

```text
Web Word UI
  -> Agent Gateway
  -> Agent Runtime
  -> Prompt Builder
  -> Model Client
  -> Tool Runtime
  -> Document Service / Comment Service / Suggestion Service
  -> Event Stream
  -> Thread Store
```

职责：

| 模块 | 责任 |
| --- | --- |
| Web Word UI | 展示对话、建议、批注、审批、工具进度 |
| Agent Gateway | 提供 thread/start、turn/start、approval/respond 等 API |
| Agent Runtime | 管理 session、turn、history、事件和工具回灌 |
| Prompt Builder | 构造模型输入，控制文档上下文注入 |
| Model Client | 调模型和处理流式输出 |
| Tool Runtime | 执行文档工具、权限判断、审批、错误包装 |
| Product Services | 真正读写文档、批注、建议、权限 |
| Thread Store | 保存可恢复、可审计的运行日志 |

## 3. 核心领域对象

```ts
type WebWordThread = {
  id: string;
  documentId: string;
  userId: string;
  createdAt: number;
  status: "active" | "archived";
};

type WebWordTurn = {
  id: string;
  threadId: string;
  userGoal: string;
  documentSnapshotId: string;
  selection?: DocumentRange;
  status: TurnStatus;
};

type WebWordTurnContext = {
  documentId: string;
  snapshotId: string;
  selection?: DocumentRange;
  visibleOutline?: DocumentOutlineNode[];
  permissions: DocumentPermission[];
  userLocale?: string;
  outputMode: "chat" | "suggestion" | "comment" | "structured";
};
```

## 4. 工具设计

优先提供业务语义工具，而不是底层数据库工具。

### 推荐 direct tools

```ts
type DocumentTool =
  | {
      name: "document.getSelection";
      input: {};
      output: { range: DocumentRange; text: string };
    }
  | {
      name: "document.search";
      input: { query: string; limit?: number };
      output: { matches: Array<{ range: DocumentRange; text: string }> };
    }
  | {
      name: "document.createSuggestion";
      input: { range: DocumentRange; replacement: string; rationale?: string };
      output: { suggestionId: string; range: DocumentRange };
    }
  | {
      name: "document.addComment";
      input: { range: DocumentRange; text: string };
      output: { commentId: string; range: DocumentRange };
    };
```

### 谨慎 direct 暴露的工具

- `document.replaceRange`
- `document.applyBatch`
- `document.deleteRange`
- `document.share`
- `document.export`

这些工具更适合要求审批，或者先生成 suggestion。

## 5. Prompt 构造策略

不要默认把整篇文档塞进 prompt。

推荐按任务注入：

| 场景 | 注入内容 |
| --- | --- |
| 改写选区 | 当前选区、上下段、文档风格摘要 |
| 长文档问答 | 相关片段检索结果、目录、引用范围 |
| 审阅 | 目标段落、审阅规则、已有批注 |
| 格式调整 | 结构树、样式信息、目标范围 |
| 模板填充 | 模板 schema、用户资料、可用字段 |

通用 prompt input：

```ts
type WebWordPromptInput =
  | { type: "history"; items: PromptInputItem[] }
  | { type: "document_context"; snapshotId: string; ranges: DocumentRangeWithText[] }
  | { type: "current_user_message"; text: string; selection?: DocumentRange }
  | { type: "tool_result"; callId: string; result: unknown };
```

## 6. 事件设计

UI 需要看到 agent 的过程，但不是每个过程都进入模型历史。

```ts
type WebWordAgentEvent =
  | { type: "turn_started"; turnId: string }
  | { type: "model_reasoning"; turnId: string; summary?: string }
  | { type: "tool_call_started"; callId: string; name: string }
  | { type: "tool_call_completed"; callId: string; outputPreview?: string }
  | { type: "suggestion_created"; suggestionId: string; range: DocumentRange }
  | { type: "approval_requested"; requestId: string; action: string }
  | { type: "turn_completed"; turnId: string }
  | { type: "turn_failed"; turnId: string; message: string };
```

建议：

- UI event 用于实时体验。
- agent item 用于模型历史。
- rollout log 用于恢复和审计。
- telemetry 用于产品观测。

## 7. 权限和审批

Web Word 的权限比代码 agent 更偏业务协作。

至少区分：

| 行为 | 建议策略 |
| --- | --- |
| 读取当前选区 | 默认允许 |
| 搜索当前文档 | 默认允许 |
| 创建建议 | 默认允许或轻审批 |
| 添加批注 | 默认允许或按企业策略 |
| 直接替换正文 | 需要审批 |
| 批量修改 | 需要审批 |
| 访问外部业务系统 | 需要权限和审计 |
| 分享、导出、发送邮件 | 强审批 |

核心原则：

> 模型可以建议，工具负责执行，用户或权限系统决定是否生效。

## 8. MVP 路线

### 阶段 1：单 turn 改写

- thread/start。
- turn/start。
- selection context。
- `document.getSelection`。
- `document.createSuggestion`。
- 基础事件流。
- 简单持久化。

### 阶段 2：长文档问答

- 文档片段检索。
- range 引用。
- 工具结果回灌。
- 历史截断。

### 阶段 3：审阅和批注

- `document.addComment`。
- 审阅规则注入。
- 批注历史。
- 用户接受/拒绝事件。

### 阶段 4：审批和批量修改

- approval/respond。
- batch suggestion。
- rollback。
- 审计记录。

### 阶段 5：外部系统工具

- deferred tools。
- tool_search。
- connectors。
- 企业权限。

## 9. 最重要的落地原则

1. 先做可靠的单 turn，再做多 turn 长记忆。
2. 先做 suggestion，不要默认直接改正文。
3. 工具输出必须带 range/id，方便 UI 定位和后续模型引用。
4. 历史分层设计要一开始就做好，否则后面 resume、rollback、compact 都会痛苦。
5. prompt builder 应该是后端核心模块，不要散落在 UI。
6. event stream 要服务体验，但不要污染模型历史。

