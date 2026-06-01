# Codex Agent 技术实现学习路线

本文给出一条按源码入口学习 agent 实现的路线，并附上常见扩展任务的落点。它适合边读源码边做笔记或准备开发。

## 1. 先抓住协议骨架

先读：

- `codex-rs/protocol/src/protocol.rs`
- `codex-rs/app-server-protocol/src/protocol/v2/thread.rs`
- `codex-rs/app-server-protocol/src/protocol/v2/turn.rs`
- `codex-rs/app-server-protocol/src/protocol/v2/item.rs`
- `codex-rs/app-server-protocol/src/protocol/v2/notification.rs`

重点看：

- `Op`：core 能接收哪些操作。
- `EventMsg`：core 会产出哪些事件。
- `ThreadStartParams` / `ThreadStartResponse`。
- `TurnStartParams` / `TurnStartResponse`。
- `UserInput` 的 text/image/local image/skill/mention。
- v2 类型的 camelCase、experimental、TS export 规则。

判断标准：读完后能解释“一次 turn/start 如何变成 core op，以及结果如何以 notification 返回”。

## 2. 再看 app-server 边界

先读：

- `codex-rs/app-server/src/message_processor.rs`
- `codex-rs/app-server/src/request_processors/initialize_processor.rs`
- `codex-rs/app-server/src/request_processors/thread_processor.rs`
- `codex-rs/app-server/src/request_processors/turn_processor.rs`
- `codex-rs/app-server/src/thread_state.rs`
- `codex-rs/app-server/src/bespoke_event_handling.rs`

重点看：

- initialize/initialized 连接生命周期。
- request processor 如何按方法分发。
- thread/start 如何加载配置、动态工具和环境。
- turn/start 如何做输入限制、permission/sandbox 冲突校验、override 校验。
- core event 如何映射成 v2 notification。

判断标准：能说清 app-server 做边界校验和协议转换，core 做 agent 执行。

## 3. 进入 core 会话生命周期

先读：

- `codex-rs/core/src/thread_manager.rs`
- `codex-rs/core/src/codex_thread.rs`
- `codex-rs/core/src/session/mod.rs`
- `codex-rs/core/src/session/session.rs`
- `codex-rs/core/src/session/handlers.rs`

重点看：

- `ThreadManagerState` 持有哪些共享服务。
- `StartThreadOptions` 如何启动 session。
- `CodexThread::submit` 是 app-server 到 session 的门面。
- `submission_loop` 如何分发 `Op`。
- `user_input_or_turn_inner` 如何创建 turn、尝试 steer、启动 task。
- `SessionConfiguration::apply` 如何处理 turn override。

判断标准：能把 thread、session、turn、submission、event 的关系画出来。

## 4. 深入 turn 执行

先读：

- `codex-rs/core/src/session/turn_context.rs`
- `codex-rs/core/src/session/turn.rs`
- `codex-rs/core/src/client.rs`
- `codex-rs/core/src/stream_events_utils.rs`

重点看：

- `TurnContext` 从哪些配置和状态生成。
- `run_sampling_request` 如何构建工具、prompt、client session。
- `try_run_sampling_request` 如何消费模型 stream。
- 工具调用完成后为什么需要 follow-up sampling。
- stream retry、fallback、rate limit、token usage 如何处理。

判断标准：能解释“一个用户 turn 内部可能有多次模型请求和多次工具调用”。

## 5. 学工具系统

先读：

- `codex-rs/core/src/tools/spec_plan.rs`
- `codex-rs/core/src/tools/router.rs`
- `codex-rs/core/src/tools/registry.rs`
- `codex-rs/core/src/tools/orchestrator.rs`
- `codex-rs/core/src/tools/context.rs`
- `codex-rs/core/src/tools/events.rs`

再按工具类型读：

- shell: `codex-rs/core/src/tools/handlers/shell/`
- unified exec: `codex-rs/core/src/tools/handlers/unified_exec/`
- apply patch: `codex-rs/core/src/tools/handlers/apply_patch.rs`
- MCP: `codex-rs/core/src/tools/handlers/mcp.rs`
- dynamic: `codex-rs/core/src/tools/handlers/dynamic.rs`
- plan: `codex-rs/core/src/tools/handlers/plan.rs`
- goal: `codex-rs/core/src/tools/handlers/goal/`
- request user input: `codex-rs/core/src/tools/handlers/request_user_input.rs`
- multi agent: `codex-rs/core/src/tools/handlers/multi_agents.rs`

重点看：

- spec 如何生成。
- handler 如何解析参数。
- ToolRuntime 如何声明审批和沙箱需求。
- 输出如何转成模型可消费结果。

判断标准：能设计一个新工具需要改哪些文件、加哪些测试、接入哪些权限。

## 6. 学安全、审批和沙箱

先读：

- `codex-rs/core/src/config/permissions.rs`
- `codex-rs/core/src/exec_policy.rs`
- `codex-rs/core/src/tools/sandboxing.rs`
- `codex-rs/core/src/tools/orchestrator.rs`
- `codex-rs/core/src/exec.rs`
- `codex-rs/core/src/guardian/`
- `codex-rs/core/src/network_policy_decision.rs`

重点看：

- approval policy 和 permission profile 的区别。
- legacy sandbox policy 如何从 permission profile 投影。
- ToolOrchestrator 如何做 approval -> sandbox -> run -> retry。
- network approval 和 managed network proxy 如何参与。
- guardian 和 permission hooks 如何接入。

判断标准：能判断一个新副作用能力是否需要审批、沙箱、网络规则和事件。

## 7. 学 MCP、Apps、Plugins、Skills

MCP：

- `codex-rs/codex-mcp/src/connection_manager.rs`
- `codex-rs/codex-mcp/src/rmcp_client.rs`
- `codex-rs/core/src/session/mcp.rs`
- `codex-rs/core/src/mcp_tool_call.rs`
- `codex-rs/core/src/mcp_tool_exposure.rs`

Apps/connectors：

- `codex-rs/core/src/connectors.rs`
- `codex-rs/connectors/src/`
- `codex-rs/app-server-protocol/src/protocol/v2/apps.rs`

Skills/AGENTS.md：

- `codex-rs/core/src/agents_md.rs`
- `codex-rs/core/src/skills.rs`
- `codex-rs/core/src/context/available_skills_instructions.rs`
- `codex-rs/core/src/context/skill_instructions.rs`

Plugins：

- `codex-rs/core/src/plugins/`
- `codex-rs/core-plugins/`
- `codex-rs/app-server/src/request_processors/plugins.rs`

重点看：

- 外部能力如何发现。
- 哪些能力进入 prompt，哪些进入 tool specs。
- 插件和 MCP 如何被限制在对应边界。

判断标准：能区分 instruction injection、tool exposure、resource read、connector discovery。

## 8. 学多 agent

先读：

- `codex-rs/core/src/agent/control.rs`
- `codex-rs/core/src/agent/registry.rs`
- `codex-rs/core/src/agent/mailbox.rs`
- `codex-rs/core/src/agent/role.rs`
- `codex-rs/core/src/tools/handlers/multi_agents.rs`
- `codex-rs/core/src/tools/handlers/multi_agents_v2.rs`
- `codex-rs/core/src/session/multi_agents.rs`

重点看：

- `AgentControl` 为什么持有 `Weak<ThreadManagerState>`。
- root session tree 如何共享 registry。
- spawn 如何继承 shell snapshot 和 exec policy。
- fork context 如何清洗 history。
- send/wait/close/resume 如何通过 mailbox 和 thread status 协作。
- roles 如何影响子 agent config。

判断标准：能解释为什么 sub-agent 是 thread，而不是普通函数调用。

## 9. 学持久化与恢复

先读：

- `codex-rs/thread-store/src/lib.rs`
- `codex-rs/core/src/rollout.rs`
- `codex-rs/core/src/session/rollout_reconstruction.rs`
- `codex-rs/core/src/thread_rollout_truncation.rs`
- `codex-rs/core/src/context_manager/`
- `codex-rs/core/src/compact.rs`

重点看：

- rollout item 和 thread item 如何保存过程事实。
- resume/fork 如何重建历史。
- rollback 如何写入 marker。
- compact 如何改变模型可见上下文。
- token usage replay 如何服务 app-server。

判断标准：能区分“保存完整历史”和“构造模型输入”。

## 10. 学 UI 和快照

TUI 入口：

- `codex-rs/tui/src/lib.rs`
- `codex-rs/tui/src/app.rs`
- `codex-rs/tui/src/chatwidget.rs`
- `codex-rs/tui/src/bottom_pane/`

重点看：

- app-server events 如何变成 UI 状态。
- approval、plan、tool output、reasoning、goal 如何渲染。
- snapshot 测试如何覆盖用户可见变化。

判断标准：能判断一个 core event 变更是否需要 app-server mapping 和 TUI snapshot。

## 11. 常见扩展任务落点

### 新增 app-server API

1. 改 `codex-rs/app-server-protocol/src/protocol/v2/` 类型。
2. 在 `common.rs` 注册方法。
3. 加 processor 或扩展现有 processor。
4. 映射到 core op 或 thread manager 能力。
5. 更新 README/schema fixtures。
6. 跑 `cargo test -p codex-app-server-protocol`。

注意：v2 payload 字段遵守 camelCase、`*Params`/`*Response` 命名、TS export、experimental gating。

### 新增模型工具

1. 在 `core/src/tools/handlers/` 下加 spec 和 handler。
2. 接入 `spec_plan` 和 `ToolRegistry`。
3. 如果有副作用，实现 ToolRuntime 或走 orchestrator。
4. 定义 approval requirement、sandbox preference、network approval spec。
5. 加 spec tests、handler tests、必要的 session tests。
6. 如果用户可见，更新 app-server/TUI 映射或 snapshots。

注意：不要让 handler 私自越过权限系统做副作用。

### 新增 MCP 相关能力

1. 优先看 `codex-rs/codex-mcp/src/connection_manager.rs` 是否已有抽象。
2. 如果只是工具调用或资源读取，接入现有 manager。
3. 如果改变 app-server API，同步 v2 schema。
4. 加 MCP processor/core tests。

注意：仓库说明要求 MCP tool calls 优先利用 MCP connection manager 或现有连接管理抽象，减少层层透传；当前源码入口是 `codex-rs/codex-mcp/src/connection_manager.rs`。

### 新增多 agent 行为

1. 判断是 tool surface、agent registry、mailbox、role config 还是 app-server 展示。
2. 保持 sub-agent 作为 thread 的模型。
3. 明确继承哪些父配置，哪些必须覆盖。
4. 检查 max threads/depth/fork history。
5. 加 `core/src/agent/*_tests.rs` 和 `tools/handlers/multi_agents*_tests.rs`。

注意：避免让 root 和 sub-agent 共享不该共享的可变状态。

### 新增权限或沙箱能力

1. 先建模到 permission profile 或 exec policy。
2. app-server 只做边界选择和校验。
3. ToolOrchestrator 负责审批和沙箱选择。
4. shell/runtime 层只执行已选择的策略。
5. 加配置解析、策略判断、工具运行测试。

注意：不要在工具 handler 内硬编码“如果失败就无沙箱再试”。

### 修改 TUI 可见输出

1. 先确认 core event 或 app-server notification 是否正确。
2. 再改 TUI 渲染。
3. 按仓库规则更新 insta snapshot。
4. 跑 `cargo test -p codex-tui`，检查 pending snapshots。

注意：TUI 大文件很多，优先新增模块而不是继续膨胀核心 UI 文件。

## 12. 推荐的读源码验证方法

每读一个场景，都回答四个问题：

1. 入口在哪里：CLI/TUI/exec/app-server/tool/MCP？
2. 状态属于谁：thread、session、turn、item、tool call、MCP connection 还是 agent registry？
3. 风险在哪里：审批、权限、沙箱、网络、历史污染、上下文超限、并发？
4. 证据在哪里：事件、持久化记录、测试、schema、snapshot？

如果四个问题都能回答，这个场景基本就读透了。
