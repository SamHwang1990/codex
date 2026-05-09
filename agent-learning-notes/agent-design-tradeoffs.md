# Codex Agent 方案设计与权衡

本文整理本仓库 agent 架构中反复出现的设计取舍。它关注“为什么这样设计”，而不是逐行解释代码。

## 1. 统一 app-server 边界 vs 各产品直接调用 core

当前倾向：TUI、exec、IDE/桌面等富客户端尽量通过 app-server 或 in-process app-server client 进入 core。

收益：

- 产品入口复用 thread/turn/item 模型。
- v2 协议可以导出 TypeScript/schema，方便客户端稳定集成。
- 审批、权限、MCP、工具、事件通知语义集中。
- exec 和 TUI 虽然输出形态不同，但不会分叉两套 agent 运行逻辑。

代价：

- 简单入口也要理解 JSON-RPC/app-server 语义。
- app-server processor 会承担大量边界校验和事件映射。
- 需要维护 core `EventMsg` 与 v2 notification 的映射。

替代方案是各入口直接调用 `codex-core`。短期会更简单，长期会造成 TUI、exec、IDE 行为漂移。

## 2. SQ/EQ 异步模型 vs 同步 request/response

当前倾向：外部提交 `Submission { id, op, trace }`，core 异步产出 `Event { id, msg }`。

收益：

- 适合长时间运行的 agent turn。
- 可以流式展示模型输出、工具输出和审批请求。
- 中断、steer、approval response 可以作为后续 op 回到同一 session。
- 事件天然可被持久化和 replay。

代价：

- 调用方必须处理乱序、状态机和 turn id 关联。
- 错误不一定在 submit 时返回，可能以事件形式出现。
- 测试需要断言事件序列，而不是单一返回值。

关键源码：`codex-rs/protocol/src/protocol.rs`、`codex-rs/core/src/session/handlers.rs`。

## 3. Thread/Session/Turn 分层 vs 单一 Conversation 对象

当前倾向：

- Thread：持久身份和历史边界。
- Session：运行时资源和状态。
- Turn：一次用户任务边界。
- Item：turn 内过程事实。

收益：

- resume/fork/archive/rollback 可以围绕 thread 建模。
- active turn、MCP、pending approvals、background processes 留在 session。
- 工具调用、模型输出、用户输入作为 item 可审计。
- 多 agent 可以复用“sub-agent 是新 thread”的模型。

代价：

- 对新读者概念较多。
- 有些数据要在 thread metadata、session config、turn context 之间同步。
- API 设计必须明确字段是 thread-sticky 还是 turn-scoped。

## 4. TurnContext 快照 vs 全局可变配置

当前倾向：每轮创建 `TurnContext`，本轮模型、cwd、permissions、tools、environment、schema 等都以它为准。

收益：

- 一次 turn 内的执行语义稳定。
- override 可以按提交顺序排队。
- 工具执行不必到处读取全局可变配置。
- 测试可以围绕 turn context 构造场景。

代价：

- 配置更新要通过 `SessionSettingsUpdate` 显式应用。
- 一些运行时状态需要小心区分“本轮快照”和“下轮 sticky 配置”。

关键源码：`codex-rs/core/src/session/session.rs`、`codex-rs/core/src/session/turn_context.rs`。

## 5. PermissionProfile 优先 vs 直接暴露 sandboxPolicy

当前倾向：app-server v2 推荐 `permissions` profile selection；旧 `sandboxPolicy` 仍保留兼容，并且不能和 `permissions` 同时出现。

收益：

- permission profile 能表达文件系统、网络、审批、平台沙箱等更完整权限。
- active profile 可以携带来源和显示信息。
- 管理策略可以约束可选 profile。
- 旧 sandbox policy 只是投影，不适合作为完整权限模型。

代价：

- API 和配置解析更复杂。
- 需要兼容旧客户端。
- turn/start 必须做同步校验，避免用户输入被接受后才失败。

关键源码：`codex-rs/app-server/src/request_processors/turn_processor.rs`、`codex-rs/core/src/config/permissions.rs`。

## 6. ToolRouter + ToolRegistry vs 工具散落在 turn 主循环

当前倾向：`ToolRouter` 负责把模型 output item 转成统一 `ToolCall`，`ToolRegistry` 负责 handler 分发。

收益：

- 模型输出形态、MCP 工具、dynamic tools、local shell、custom tool 可以统一处理。
- 工具规格和运行时分发保持对应。
- 支持参数 delta consumer、并行能力判断、deferred tools。
- 新工具有明确的 spec、handler、runtime、test 位置。

代价：

- 新增工具要理解 spec builder、registry、handler 输出格式。
- 一些工具既影响上下文又影响运行时，需要在多个位置接入。

关键源码：`codex-rs/core/src/tools/router.rs`、`codex-rs/core/src/tools/registry.rs`、`codex-rs/core/src/tools/spec_plan.rs`。

## 7. 中央 ToolOrchestrator vs 每个工具自管审批沙箱

当前倾向：工具运行统一经过 `ToolOrchestrator`，由它处理 approval、permission hooks、guardian、sandbox selection、network approval、retry。

收益：

- 安全策略一致。
- 工具 handler 不会各自实现审批逻辑。
- 可以集中记录 telemetry 和 decision source。
- sandbox denial 后的升级/重试语义可控。

代价：

- ToolRuntime trait 需要提供 approval requirement、sandbox preference、network approval spec 等信息。
- 对纯读工具而言接入路径看起来偏重。

关键源码：`codex-rs/core/src/tools/orchestrator.rs`、`codex-rs/core/src/tools/sandboxing.rs`。

## 8. MCP 作为外部能力层 vs 内置所有工具

当前倾向：Codex 内置关键开发工具，同时通过 MCP 接入外部工具生态。

收益：

- 外部系统可以独立演进。
- tools/resources/templates/elicitation 有统一协议。
- MCP server 失败可以局部降级。
- Apps/connectors 可以基于 MCP 工具做能力发现。

代价：

- 启动状态、连接生命周期、tool name canonicalization、parallel 支持都要管理。
- 模型可见工具需要筛选，不能简单暴露所有 MCP tool。
- elicitation 让外部 server 也能反向请求用户决策，状态机更复杂。

关键源码：`codex-rs/codex-mcp/src/connection_manager.rs`、`codex-rs/core/src/session/mcp.rs`。

## 9. Skills/AGENTS.md 注入上下文 vs 把流程写死进代码

当前倾向：AGENTS.md、skills、plugin skills 作为指令来源进入上下文，模型按这些流程执行。

收益：

- 仓库规则和专业流程可以不改 core 代码。
- 用户、组织、插件都能扩展 agent 行为。
- skill 可以承载“如何做事”的知识，而工具承载“能做什么”的能力。

代价：

- 指令冲突、优先级和加载来源需要清晰。
- 隐式 skill invocation 可能影响 prompt，需要可观察。
- 流程约束由模型执行，不能替代代码级权限边界。

关键源码：`codex-rs/core/src/agents_md.rs`、`codex-rs/core/src/skills.rs`、`codex-rs/core/src/context/`。

## 10. 多 agent 用 thread 建模 vs 轻量任务 future

当前倾向：sub-agent 是独立 thread，使用 `AgentControl`、registry、mailbox 管理。

收益：

- 子 agent 有独立历史、状态、事件和持久化边界。
- 可以 fork parent context，保留可追踪来源。
- 父 agent 可以 wait/send/close/resume。
- max threads/depth 可以限制资源扩张。

代价：

- spawn 成本高于普通 async task。
- 需要处理历史继承、shell snapshot、exec policy、role config、metadata。
- 父子通信和 UI 展示需要额外事件。

关键源码：`codex-rs/core/src/agent/control.rs`、`codex-rs/core/src/tools/handlers/multi_agents.rs`。

## 11. 持久化完整过程 vs 只保存最终消息

当前倾向：保存 thread、turn、item、rollout、metadata、token usage、rollback marker 等过程事实。

收益：

- 支持 resume、fork、rollback、audit、debug。
- 可以从历史重建模型上下文。
- UI 可以展示工具、审批、diff 和 reasoning 摘要。
- token replay 和 thread list/read 更可靠。

代价：

- 存储模型更复杂。
- 需要区分持久化历史和 prompt 可见历史。
- 中断、compact、rollback 都需要明确 marker。

关键源码：`codex-rs/thread-store/src/lib.rs`、`codex-rs/core/src/rollout.rs`、`codex-rs/core/src/session/rollout_reconstruction.rs`。

## 12. Context compact vs 无限历史

当前倾向：历史完整保存，但模型输入会经过选择、截断和 compact。

收益：

- 长会话可继续工作。
- 工具大输出不会挤爆上下文。
- compact 后保留关键事实。

代价：

- compact 是有损摘要，可能丢失细节。
- 需要在 UI 和历史中标记 compact 事实。
- 测试必须覆盖恢复和 fork 后上下文是否合理。

关键源码：`codex-rs/core/src/compact.rs`、`codex-rs/core/src/context_manager/`。

## 13. v2 API 稳定 schema vs 快速内部字段

当前倾向：app-server v2 类型导出 schema/TypeScript，实验字段显式 gating。

收益：

- 客户端集成稳定。
- wire naming、optional/nullability、experimental 能被检查。
- 文档和 schema fixture 能防止 API 漂移。

代价：

- 新字段需要遵守命名、serde、ts_rs、schema 规则。
- 内部结构不能直接暴露，需要 mapper。
- API 变更要同步文档和 fixtures。

关键源码：`codex-rs/app-server-protocol/src/protocol/v2/`、`codex-rs/app-server-protocol/src/protocol/common.rs`。

## 14. 抵制继续膨胀 `codex-core`

仓库规则明确提醒：不要因为方便就把所有新概念放进 `codex-core`。

推荐判断：

- 协议字段放 `app-server-protocol`。
- 客户端边界校验放 app-server processor。
- 工具规格通用逻辑放 `codex-tools` 或 `core/src/tools`。
- MCP 连接管理优先放 `codex-mcp`。
- 插件/marketplace 放 plugin 相关 crate。
- 持久化接口放 thread-store/state 相关 crate。

收益：边界清楚、编译和测试范围可控、后续维护成本低。

代价：有时需要先抽象接口，不能只在一个文件里快速加逻辑。

## 15. 做 agent 扩展时的推荐决策顺序

1. 先判断这是产品入口、协议、core 行为、工具、MCP、插件、持久化还是 UI 展示。
2. 如果涉及新客户端能力，先设计 app-server v2 类型和 event/notification。
3. 如果涉及模型可调用能力，定义 tool spec、handler、runtime、approval requirement 和 tests。
4. 如果涉及副作用，必须经过 permission profile、approval、sandbox、event。
5. 如果涉及上下文，明确它是 instruction、history、tool spec、environment 还是 dynamic tool。
6. 如果涉及长期状态，明确 thread/session/turn/item 哪个边界拥有它。
7. 最后再决定是否需要改 `codex-core`，并尽量把改动放在更窄模块。
