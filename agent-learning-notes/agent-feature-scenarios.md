# Codex Agent 功能场景矩阵

本文按“用户能感知的完整场景”整理 Codex agent 的功能面。每个场景都包含入口、核心流程、关键源码和学习要点，便于从需求反推实现。

## 1. 交互式代码协作

用户在 TUI 或 IDE 中输入自然语言请求，例如“修复这个测试失败”。入口通常是 TUI 或 app-server client，最终都会转成 app-server `turn/start`，再进入 core `Op::UserInputWithTurnContext` 或 `Op::UserTurn`。

核心链路：

```text
tui/ide input
-> app-server turn/start
-> CodexThread::submit
-> session submission_loop
-> new TurnContext
-> run_sampling_request
-> ToolRouter + model stream
-> tool calls / final message
-> EventMsg
-> app-server notifications
-> UI render
```

关键源码：

- `codex-rs/app-server-protocol/src/protocol/v2/turn.rs`
- `codex-rs/app-server/src/request_processors/turn_processor.rs`
- `codex-rs/core/src/session/handlers.rs`
- `codex-rs/core/src/session/turn.rs`
- `codex-rs/core/src/tools/router.rs`

学习要点：用户请求不会直接调用模型，而是被包装成有 id、可追踪、可中断、可持久化的 turn。

## 2. 非交互式 `codex exec`

`codex exec` 面向脚本化任务：输入一次请求，输出最终文本或 JSONL 事件。它和 TUI 共享 app-server/core 路径，但输出约束更严格。

核心差异：

- stdout 只放最终答案或机器可读 JSONL。
- stderr 放进度和日志。
- 可以从 stdin、参数、图片、output schema 构造一次性 turn。
- 仍然走权限、沙箱、工具和持久化逻辑。

关键源码：

- `codex-rs/exec/src/lib.rs`
- `codex-rs/app-server/src/in_process.rs`
- `codex-rs/app-server-protocol/src/protocol/v2/turn.rs`

学习要点：产品形态可以不同，但 agent 运行闭环应该复用同一协议和 core。

## 3. Thread 生命周期

Thread 是一条可恢复的工作线。它支持 start、resume、fork、archive、list、read、rollback、rename、goal 和 metadata 更新。

核心流程：

```text
thread/start or resume
-> ThreadManager loads config/history
-> CodexThread created
-> Session spawned
-> rollout/thread-store attached
-> app-server returns ThreadStartResponse
```

关键源码：

- `codex-rs/core/src/thread_manager.rs`
- `codex-rs/core/src/codex_thread.rs`
- `codex-rs/app-server/src/request_processors/thread_processor.rs`
- `codex-rs/app-server-protocol/src/protocol/v2/thread.rs`
- `codex-rs/thread-store/src/lib.rs`

学习要点：Thread 是持久业务身份；Session 是运行时实体；Turn 是一次任务边界。

## 4. Turn override 与上下文更新

客户端可以在 `turn/start` 上调整 cwd、model、effort、summary、service tier、approval、sandbox、permissions、personality、collaboration mode、environment。

设计规则：

- 有 override 时 app-server 先同步校验，避免接受用户输入后才发现配置无效。
- `permissions` 不能和旧 `sandboxPolicy` 同时使用。
- 真正生效的配置随 op 一起排队，保证顺序一致。
- 本轮运行以 `TurnContext` 为准。

关键源码：

- `codex-rs/app-server/src/request_processors/turn_processor.rs`
- `codex-rs/core/src/codex_thread.rs`
- `codex-rs/core/src/session/session.rs`
- `codex-rs/core/src/session/turn_context.rs`

学习要点：turn override 是 API 语义、配置约束和执行顺序共同构成的功能，不只是字段赋值。

## 5. 流式模型采样

模型返回不是单一响应，而是一串 `ResponseEvent`。core 边接收边转成用户可见事件，并在工具调用出现时启动工具闭环。

核心处理：

- output item started/completed。
- assistant text delta。
- reasoning summary/content delta。
- function/tool call arguments delta。
- completed/rate limit/model verification。
- stream error retry 和 WebSocket 到 HTTPS fallback。

关键源码：

- `codex-rs/core/src/session/turn.rs`
- `codex-rs/core/src/client.rs`
- `codex-rs/protocol/src/protocol.rs`
- `codex-rs/app-server-protocol/src/protocol/event_mapping.rs`

学习要点：agent UI 的“实时感”来自事件流；工具调用和最终回答也都是流中的 item。

## 6. 工具暴露与工具路由

每轮可见工具由配置、MCP、plugins、apps/connectors、dynamic tools、skills 和输入 mention 共同决定。

核心流程：

```text
built_tools
-> read MCP tools
-> load plugins/apps
-> merge connectors
-> filter explicit mentions and tool suggestions
-> build ToolRouter
-> model_visible_specs sent to model
```

关键源码：

- `codex-rs/core/src/session/turn.rs`
- `codex-rs/core/src/tools/router.rs`
- `codex-rs/core/src/tools/spec_plan.rs`
- `codex-rs/core/src/tools/registry.rs`
- `codex-rs/tools/src/`

学习要点：工具规格是模型合约；ToolRouter 是模型输出到真实工具执行的翻译层。

## 7. Shell 命令执行

模型请求 shell 后，系统要解析命令、判断权限、选择沙箱、执行、流式回传 stdout/stderr、截断大输出并把结果反馈给模型。

核心风险：

- 文件系统写入。
- 网络访问。
- 长运行进程。
- 输出过大。
- 平台差异。
- 用户审批和 guardian 自动审查。

关键源码：

- `codex-rs/core/src/tools/handlers/shell/`
- `codex-rs/core/src/tools/orchestrator.rs`
- `codex-rs/core/src/exec.rs`
- `codex-rs/core/src/sandboxing/`
- `codex-rs/core/src/tools/runtimes/shell/`

学习要点：shell 不是普通工具，它是权限、沙箱、审批、网络策略和事件流的集中交汇点。

## 8. Apply Patch 与文件修改

文件修改可以通过 patch 工具完成。系统会记录 patch begin/end、必要时审批，并把文件 diff 和结果作为 turn item 保存。

关键源码：

- `codex-rs/core/src/tools/handlers/apply_patch.rs`
- `codex-rs/core/src/tools/handlers/apply_patch_spec.rs`
- `codex-rs/core/src/tools/runtimes/apply_patch.rs`
- `codex-rs/core/src/apply_patch.rs`

学习要点：文件变更要进入工具和事件体系，不能绕过权限、审计和持久化。

## 9. 审批、权限与安全

审批回答“是否需要问用户”；权限 profile 回答“实际给了哪些能力”；sandbox 回答“进程在哪里、以什么限制运行”。

典型流程：

```text
tool request
-> ToolOrchestrator
-> exec_approval_requirement
-> permission hooks / guardian / user approval
-> SandboxManager selects attempt
-> run tool
-> retry with escalated strategy if allowed
```

关键源码：

- `codex-rs/core/src/tools/orchestrator.rs`
- `codex-rs/core/src/config/permissions.rs`
- `codex-rs/core/src/exec_policy.rs`
- `codex-rs/core/src/guardian/`
- `codex-rs/protocol/src/protocol.rs`

学习要点：安全是 core 的业务逻辑，不是 UI 的二次确认弹窗。

## 10. MCP 工具生态

MCP server 提供外部 tools、resources 和 resource templates。Codex 负责启动、刷新、聚合、暴露、调用和处理 elicitation。

关键源码：

- `codex-rs/codex-mcp/src/connection_manager.rs`
- `codex-rs/core/src/session/mcp.rs`
- `codex-rs/core/src/tools/handlers/mcp.rs`
- `codex-rs/core/src/tools/handlers/mcp_resource/`
- `codex-rs/app-server/src/request_processors/mcp_processor.rs`

学习要点：MCP 失败通常只降级相关工具，不应让整个 agent 无法工作。

## 11. Apps、Connectors 与 Tool Search

Apps/connectors 是面向用户和插件的能力发现层。工具可能来自 MCP，也可能来自 plugin connector，再根据 mention、可访问性和配置决定是否本轮暴露。

关键源码：

- `codex-rs/core/src/connectors.rs`
- `codex-rs/connectors/src/`
- `codex-rs/core/src/tools/handlers/tool_search.rs`
- `codex-rs/app-server-protocol/src/protocol/v2/apps.rs`
- `codex-rs/app-server/src/request_processors/apps_processor.rs`

学习要点：能力发现和模型工具调用解耦，避免把所有潜在工具一次性塞进上下文。

## 12. Skills 与 AGENTS.md

AGENTS.md 是项目本地规则；skills 是可复用流程知识；plugins 可以贡献 skills。它们共同影响模型“应该怎么做”，但不是同一种能力。

关键源码：

- `codex-rs/core/src/agents_md.rs`
- `codex-rs/core/src/skills.rs`
- `codex-rs/core-skills/`
- `codex-rs/core/src/context/skill_instructions.rs`
- `codex-rs/core/src/context/user_instructions.rs`

学习要点：instructions 是 agent 行为的一部分，需要保留来源和优先级。

## 13. Plugins 与 Marketplace

插件可以贡献 skills、apps/connectors、marketplace 元数据等。插件管理既影响上下文，也影响工具发现。

关键源码：

- `codex-rs/plugin/`
- `codex-rs/core-plugins/`
- `codex-rs/core/src/plugins/`
- `codex-rs/app-server/src/request_processors/plugins.rs`
- `codex-rs/app-server/src/request_processors/marketplace_processor.rs`

学习要点：插件能力应尽量落在 plugin/app-server 边界，不要直接膨胀 `codex-core`。

## 14. 多 Agent 协作

多 agent 不是一个模型里开多个线程，而是 root session tree 下创建多个 sub-agent thread，并通过 registry/mailbox/agent control 管理。

工具面：

- `spawn_agent`
- `send_input`
- `wait_agent`
- `close_agent`
- `resume_agent`
- v2 的 list/message/followup 等工具。

核心规则：

- sub-agent 是独立 thread。
- 继承有效配置、审批、沙箱、cwd、shell snapshot 和执行策略中的必要部分。
- 受最大线程数和深度限制。
- 可以 fork parent context。
- registry 只在 root thread tree 内共享。

关键源码：

- `codex-rs/core/src/agent/control.rs`
- `codex-rs/core/src/agent/registry.rs`
- `codex-rs/core/src/agent/mailbox.rs`
- `codex-rs/core/src/tools/handlers/multi_agents.rs`
- `codex-rs/core/src/tools/handlers/multi_agents_v2.rs`

学习要点：多 agent 的核心价值是隔离、并行和可追踪，而不是无约束地扩张模型调用。

## 15. Plan、Goal 与长任务推进

Plan 是当前 turn 内的步骤状态；Goal 是 thread 级长期目标。它们共同支持长任务的可见进度和持续推进。

关键源码：

- `codex-rs/core/src/tools/handlers/plan.rs`
- `codex-rs/core/src/tools/handlers/goal/`
- `codex-rs/core/src/goals.rs`
- `codex-rs/app-server/src/request_processors/thread_goal_processor.rs`

学习要点：计划是过程协作工具；goal 是跨 turn 的目标状态。

## 16. 用户输入请求与 Elicitation

Agent 或 MCP server 可以请求用户输入。core 通过事件向上层发出请求，上层收集答案后再以 op 回传。

关键源码：

- `codex-rs/core/src/tools/handlers/request_user_input.rs`
- `codex-rs/protocol/src/request_user_input.rs`
- `codex-rs/codex-mcp/src/elicitation.rs`
- `codex-rs/core/src/session/handlers.rs`

学习要点：需要用户决策的流程都应显式建模为 request/response，而不是阻塞在某个工具里。

## 17. Compact、Rollback、Resume 与 Fork

长会话需要控制上下文长度，同时保持持久化历史的可审计性。

能力区分：

- compact：把历史压缩为更短上下文。
- rollback：丢弃最近 N 个用户 turn 的模型可见影响，并写入 rollback marker。
- resume：从持久化 thread 继续。
- fork：从旧 thread 的某个历史切点创建新 thread。

关键源码：

- `codex-rs/core/src/compact*.rs`
- `codex-rs/core/src/session/rollout_reconstruction.rs`
- `codex-rs/core/src/thread_rollout_truncation.rs`
- `codex-rs/app-server/src/request_processors/thread_processor.rs`
- `codex-rs/thread-store/src/lib.rs`

学习要点：持久化历史追求完整，模型上下文追求可用；二者不能混为一谈。

## 18. Realtime Conversation

Realtime 支持音频和文本流式交互。它仍然进入 session handlers，但有独立的 start/audio/text/close/list voices 操作。

关键源码：

- `codex-rs/core/src/realtime_conversation.rs`
- `codex-rs/core/src/realtime_context.rs`
- `codex-rs/core/src/session/handlers.rs`
- `codex-rs/app-server-protocol/src/protocol/v2/realtime.rs`

学习要点：实时会话是同一个 session 内的特殊交互模式，需要与普通 turn 的上下文和状态保持一致。

## 19. 事件通知与 UI 映射

core 输出 `EventMsg`，app-server 映射成 v2 notifications，TUI/IDE 再渲染成界面状态。

关键事件：

- turn started/completed/interrupted/failed。
- item started/completed。
- agent message delta。
- reasoning delta。
- tool call begin/end/output delta。
- approval/user input/elicitation request。
- MCP startup。
- token usage。
- warning/error/deprecation。

关键源码：

- `codex-rs/protocol/src/protocol.rs`
- `codex-rs/app-server-protocol/src/protocol/event_mapping.rs`
- `codex-rs/app-server/src/bespoke_event_handling.rs`
- `codex-rs/tui/src/chatwidget.rs`

学习要点：UI 不应猜测 core 状态，而应消费明确事件。

## 20. 测试与快照场景

agent 功能的测试覆盖通常分布在对应模块附近，而不是一个大端到端测试包里。

测试类型：

- protocol/schema 测试。
- app-server processor 测试。
- session/core 集成测试。
- tool spec 和 handler 测试。
- MCP 连接测试。
- multi-agent control/handler 测试。
- TUI snapshot 测试。

关键源码：

- `codex-rs/core/src/session/tests.rs`
- `codex-rs/core/src/tools/handlers/*_tests.rs`
- `codex-rs/core/src/agent/*_tests.rs`
- `codex-rs/app-server/src/request_processors/*_tests.rs`
- `codex-rs/app-server-protocol/src/protocol/v2/tests.rs`

学习要点：新增 agent 能力时，测试应覆盖协议边界、core 行为、工具结果和 UI 可见变化。
