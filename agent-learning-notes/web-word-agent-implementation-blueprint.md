# 自研 Web Word Agent 实现蓝图

你的目标不是维护 Codex 仓库，而是从这个仓库学习 agent 的完整设计，再实现一个能和公司 Web Word 产品结合的 agent。本文把 Codex 的经验抽象成 Web Word 场景下可落地的架构。

这里的 Web Word 假设是一个浏览器里的在线文档编辑器，类似 Word/Google Docs/WPS 文档：有文档结构、选区、光标、评论、修订、协作权限、版本历史和导出能力。

## 1. 一句话目标

要实现的不是“接一个聊天模型”，而是一个文档工作流 agent：

```text
用户目标
-> 理解当前文档、选区、权限和协作上下文
-> 规划要做的文档操作
-> 调用受控工具读取/修改/评论/审阅文档
-> 把每一步变化以事件流展示给用户
-> 需要风险确认时请求用户批准
-> 保存过程、结果和可回滚记录
```

Codex 面向代码仓库，核心工具是 shell、patch、MCP 和文件系统；Web Word 面向文档，核心工具应该是 document API、selection API、comment/revision API、knowledge/search API 和 export API。

## 2. 产品场景矩阵

### 2.1 写作与改写

用户说：

- “把这一段改得更正式。”
- “把选中的内容压缩到 200 字。”
- “根据这些要点写一段会议纪要。”

Agent 需要：

- 读取当前选区和附近上下文。
- 生成候选改写。
- 以建议、替换或批注形式呈现。
- 允许用户接受、拒绝、局部编辑。

关键工具：

- `document.getSelection`
- `document.getTextRange`
- `document.replaceRange`
- `document.createSuggestion`
- `document.addComment`

### 2.2 长文档问答

用户说：

- “这份合同里付款条款是什么？”
- “帮我找出所有和违约责任相关的段落。”
- “总结第 3 章和第 5 章的差异。”

Agent 需要：

- 从文档结构、标题、段落、表格、批注中检索。
- 对长文档做分块和引用定位。
- 回答时带可点击引用。

关键工具：

- `document.search`
- `document.getOutline`
- `document.getBlocks`
- `document.getComments`
- `document.resolveCitation`

### 2.3 审阅与批注

用户说：

- “帮我审阅这份方案，指出逻辑漏洞。”
- “给这份合同加风险批注。”
- “检查语病和格式问题，但不要直接改正文。”

Agent 需要：

- 只读分析文档。
- 输出结构化 review findings。
- 在文档中创建 comments 或 suggestions。
- 区分事实问题、风格建议和风险提示。

关键工具：

- `document.createReviewFinding`
- `document.addComment`
- `document.createSuggestion`
- `document.getStyleInfo`

### 2.4 格式与结构调整

用户说：

- “把这份文档整理成标准项目方案格式。”
- “统一标题层级和编号。”
- “把表格里的内容改成条目列表。”

Agent 需要：

- 理解文档 block tree。
- 生成结构化修改计划。
- 分批应用格式和结构变更。
- 支持预览和撤销。

关键工具：

- `document.getStructure`
- `document.updateBlockStyle`
- `document.moveBlocks`
- `document.convertTableToList`
- `document.applyBatch`

### 2.5 生成与填充模板

用户说：

- “基于这个模板生成一份周报。”
- “把 CRM 数据填进这份报价单。”
- “根据会议转写生成行动项表格。”

Agent 需要：

- 读取模板占位符。
- 调用业务系统或知识库。
- 生成结构化内容。
- 保留来源和可审计记录。

关键工具：

- `document.getTemplateFields`
- `document.fillTemplate`
- `business.query`
- `knowledge.search`

## 3. 推荐总体架构

建议采用五层架构：

```text
Web Word UI
  -> Agent Gateway / BFF
  -> Agent Runtime
  -> Tool Layer
  -> Product Services
```

### Web Word UI

职责：

- 展示 agent 面板。
- 发送用户输入、当前文档 id、选区、协作上下文。
- 订阅事件流。
- 展示工具进度、建议、批注、审批请求。
- 让用户接受/拒绝文档修改。

UI 不应该直接让模型改文档。它只负责交互和事件展示。

### Agent Gateway / BFF

职责：

- 提供前端 API，例如 `thread/start`、`turn/start`、`turn/interrupt`、`approval/respond`。
- 做用户身份、文档权限、租户隔离和输入大小限制。
- 把 Web Word 的文档上下文转成 agent runtime 能理解的 turn context。
- 把 runtime event 映射成前端 notification。

这一层相当于 Codex 的 app-server，但可以按你们产品简化。

### Agent Runtime

职责：

- 管理 thread/session/turn。
- 构造模型上下文。
- 处理模型流式输出。
- 调度工具调用。
- 执行审批、权限、回滚、持久化。

这是 agent 的业务核心，建议不要和 Web Word UI 混在一起。

### Tool Layer

职责：

- 把模型工具调用转成受控产品 API。
- 统一校验参数、权限和幂等性。
- 返回结构化工具结果。

Web Word 工具应围绕文档能力设计，而不是暴露底层数据库或任意 HTTP。

### Product Services

职责：

- 文档服务。
- 评论/修订服务。
- 搜索/向量检索服务。
- 权限服务。
- 版本历史和审计服务。
- 业务系统 connector。

Agent 只通过工具访问这些服务。

## 4. 核心领域模型

### Thread

表示一个持续的 agent 工作线，例如“帮我审阅合同 A”。它保存：

- thread id。
- document id。
- user id / tenant id。
- 当前目标。
- 历史 turns。
- 权限快照。
- 文档版本快照。

### Turn

表示一次用户请求，例如“把选中段落改正式一点”。它保存：

- turn id。
- 用户输入。
- 当前选区。
- 文档版本号。
- agent 输出。
- 工具调用。
- 用户审批。
- 最终状态。

### Item

表示 turn 里的过程事实：

- 用户消息。
- agent 消息。
- reasoning 摘要。
- 工具调用。
- 文档修改建议。
- comment 创建。
- approval request。
- error。

### DocumentContext

Web Word 特有的 turn context：

- document id。
- document version。
- selection range。
- visible viewport。
- surrounding paragraphs。
- outline。
- user role。
- collaboration state。
- track changes 状态。
- 是否允许直接写入。

### ToolCall

表示一次受控文档操作。它必须可审计、可重放或至少可解释。

## 5. 工具设计原则

### 优先暴露业务语义工具

不要给模型暴露太底层的工具，例如：

```text
db.query("select * from doc_blocks")
http.post("/internal/update")
```

更好的工具是：

```text
document.getSelection
document.createSuggestion
document.addComment
document.applyBatch
```

模型应该操作产品语义，而不是内部实现细节。

### 修改类工具默认生成 suggestion

对 Web Word 来说，直接改用户正文风险很高。推荐默认策略：

- 小范围低风险：允许直接替换，但要可撤销。
- 大范围改写：先生成 suggestion。
- 合同、法律、财务、审批文档：只生成 comments/suggestions，不直接改正文。
- 格式调整：允许预览后批量应用。

### 工具结果必须可定位

每次工具返回都应包含：

- affected range。
- block id。
- before/after 或 suggestion id。
- document version。
- 是否可撤销。
- 用户可见链接或定位信息。

## 6. 权限与审批设计

Web Word agent 至少需要三层权限：

### 用户权限

用户本人对文档是否有读、写、评论、分享、导出权限。Agent 不能超过用户权限。

### Agent 能力权限

即使用户有写权限，也可以限制 agent：

- 只能读。
- 可评论。
- 可创建建议。
- 可直接编辑。
- 可导出。
- 可调用外部业务系统。

### 本轮审批

某些动作需要 turn 内审批：

- 大范围替换。
- 删除内容。
- 修改合同关键条款。
- 调用外部系统获取敏感数据。
- 导出或发送文档。

推荐审批事件：

```json
{
  "type": "approvalRequest",
  "turnId": "...",
  "action": "document.applyBatch",
  "risk": "large_edit",
  "summary": "将替换 12 个段落并新增 3 条批注",
  "preview": {
    "changedBlocks": 12,
    "comments": 3
  }
}
```

## 7. 上下文构造策略

Web Word agent 的上下文不要一次塞完整文档。推荐分层：

1. 系统指令：agent 角色、禁止行为、安全边界。
2. 产品指令：如何使用文档工具、如何引用段落、何时用 suggestion。
3. 用户输入。
4. 当前选区和附近上下文。
5. 文档 outline。
6. 检索得到的相关 blocks。
7. 当前权限和审批规则。
8. 可用工具规格。

长文档策略：

- 用 outline 先建立结构。
- 按 block id 检索相关片段。
- 回答必须带引用。
- 修改前重新读取目标 range，避免基于旧上下文修改。
- 工具调用带 document version，防止并发协作冲突。

## 8. 事件流设计

前端不要等 agent 完整结束后才展示。建议事件：

- `turn.started`
- `agent.message.delta`
- `agent.reasoning.summary`
- `tool.call.started`
- `tool.call.progress`
- `document.suggestion.created`
- `document.comment.created`
- `approval.requested`
- `approval.resolved`
- `turn.completed`
- `turn.failed`
- `turn.interrupted`

这样用户能看到 agent 正在读文档、生成建议、等待确认还是已经完成。

## 9. 持久化与回滚

至少保存：

- thread。
- turn。
- user input。
- agent final message。
- tool calls。
- document operation ids。
- approvals。
- error。
- token usage 或成本信息。

修改文档时必须依赖产品自己的版本历史和 undo/redo 能力。Agent 层保存“我发起了什么操作”，文档服务保存“文档实际怎么变了”。

## 10. MVP 实现范围

第一版不要做全功能 agent。建议 MVP：

1. Agent 面板支持用户输入。
2. 支持读取当前文档选区和附近上下文。
3. 支持改写选区，默认生成 suggestion。
4. 支持对整篇文档做问答，回答带引用。
5. 支持添加批注。
6. 支持 turn 事件流。
7. 支持中断。
8. 支持保存 thread/turn/tool call。

暂缓：

- 多 agent。
- 复杂插件市场。
- 任意外部系统连接。
- 自动大范围改文。
- 跨文档批量处理。
- 完整实时语音。

## 11. 推荐 API 草案

### `thread/start`

```json
{
  "documentId": "doc_123",
  "mode": "edit_with_suggestions",
  "initialContext": {
    "selection": true,
    "outline": true
  }
}
```

### `turn/start`

```json
{
  "threadId": "thread_123",
  "input": [{ "type": "text", "text": "把选中段落改得更正式" }],
  "documentContext": {
    "documentId": "doc_123",
    "version": 42,
    "selection": { "start": "...", "end": "..." }
  }
}
```

### `approval/respond`

```json
{
  "threadId": "thread_123",
  "turnId": "turn_456",
  "approvalId": "approval_789",
  "decision": "approved"
}
```

## 12. 推荐工具草案

### `document.getSelection`

读取当前选区、block ids、文本和结构信息。

### `document.search`

按关键词或语义检索文档 blocks，返回 block id、标题路径、摘要、引用位置。

### `document.createSuggestion`

对指定 range 创建建议，不直接修改正文。

### `document.replaceRange`

直接替换指定 range。默认需要更高权限或审批。

### `document.addComment`

对指定 range 添加批注。

### `document.applyBatch`

批量应用结构化操作。必须支持 dry run 或 preview，通常需要审批。

## 13. 与 Codex 设计的对应关系

| Codex 概念 | Web Word 对应 |
| --- | --- |
| workspace cwd | document id + document version |
| AGENTS.md | 产品/租户/文档类型指令 |
| shell tool | document/business tools |
| apply_patch | createSuggestion / replaceRange / applyBatch |
| sandbox | agent capability permission + document version guard |
| approval policy | edit/comment/export approval policy |
| MCP tools | business connectors / knowledge connectors |
| thread-store | agent thread/turn storage |
| rollout | agent operation log |
| EventMsg | frontend agent notifications |
| sub-agent | 后续可用于长文档审阅、分章节分析 |

## 14. 关键设计权衡

### 直接改正文还是生成建议

推荐默认生成建议。直接改正文体验更快，但风险更高，尤其是多人协作文档和严肃文档。

### 完整文档进 prompt 还是按需检索

推荐按需检索。完整文档简单但成本高、上下文噪声大、容易过期。长文档必须用 outline + search + block reread。

### Agent runtime 放前端还是后端

推荐后端。前端适合展示和局部上下文采集，但模型密钥、权限、审计、工具调用和持久化都应在后端。

### 工具调用同步返回还是事件流

推荐事件流。文档生成、长文档检索、批量建议都可能耗时，事件流能让用户理解进度并中断。

### 一开始是否做多 agent

不建议 MVP 做。长文档分章节审阅可以先用内部任务队列或批处理实现，等单 agent 闭环稳定后再引入多 agent。

## 15. 实施路线

### 阶段 1：单 turn 文档改写

- Agent 面板。
- `turn/start`。
- 读取 selection。
- 模型生成改写。
- `document.createSuggestion`。
- 流式输出和完成事件。

### 阶段 2：长文档问答

- outline + search。
- block citation。
- 回答带引用。
- thread/turn 持久化。

### 阶段 3：审阅与批注

- review finding schema。
- comment/suggestion 工具。
- 风险等级。
- 用户接受/拒绝闭环。

### 阶段 4：审批和批量修改

- `approval.requested`。
- `document.applyBatch` preview。
- version guard。
- undo/rollback 对接。

### 阶段 5：业务系统连接

- connector registry。
- 权限和租户隔离。
- 工具发现。
- 审计和成本控制。

## 16. 最小后端模块划分

```text
agent-gateway
  api: thread/start, turn/start, interrupt, approval/respond
  auth: user/document permission check
  events: SSE/WebSocket notifications

agent-runtime
  thread manager
  session/turn runner
  prompt builder
  model client
  tool router
  approval manager

agent-tools
  document tools
  comment/revision tools
  search tools
  business connector tools

agent-store
  threads
  turns
  items
  tool calls
  approvals
```

## 17. 最容易踩的坑

- 把 agent 做成普通聊天框，无法真正操作文档。
- 直接把整篇文档塞给模型，长文档成本和质量都会失控。
- 给模型暴露太底层 API，导致安全和稳定性差。
- 没有 document version guard，协作编辑时覆盖别人修改。
- 没有 suggestion/approval，agent 误改正文后难以挽回。
- 没有事件流，用户不知道 agent 卡在哪里。
- 没有保存 tool call 和 approval，后续无法审计。
- 一开始就做多 agent 和插件市场，主闭环还没稳定就复杂化。

## 18. 判断 MVP 是否成型

满足这些条件才算真正有了 Web Word agent：

- 用户能在文档里选中内容并让 agent 改写。
- Agent 能读到正确选区和附近上下文。
- 修改以 suggestion 或可撤销操作进入文档。
- 用户能看到 agent 正在做什么。
- 用户能中断。
- Agent 的每次工具调用可审计。
- 长文档回答能定位到文档引用。
- 权限不足时 agent 明确失败或请求授权，而不是静默越权。
