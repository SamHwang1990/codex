# 2026-06-01 origin/main Merge 学习增量

这份笔记记录 `c54c377edb` 这次把 `origin/main` 合入 `agent-learning` 后，和 agent 学习资料最相关的变化。它不是完整 changelog，而是把值得更新心智模型的改动归类。

## 1. Thread 生命周期更完整：archive/unarchive 成为 CLI 入口

上游新增了 `codex archive` 和 `codex unarchive`。这不是单纯 UI 命令，而是把原本 app-server v2 已有的 `thread/archive`、`thread/unarchive` 能力暴露到了 CLI。

业务语义：

- archive 会把 thread 从默认可见集合移到 archived 集合，并在 thread-store 元数据里记录 `archived_at`。
- unarchive 会把 archived rollout 恢复到活跃集合，并返回更新后的 thread metadata。
- read/resume/update metadata 等路径开始显式携带 `include_archived`，避免把 archived thread 当作普通活跃 thread 误打开。
- archived thread 的历史仍然可读；archive 是管理状态，不是删除历史。

关键源码：

- `codex-rs/cli/src/main.rs`
- `codex-rs/tui/src/session_archive_commands.rs`
- `codex-rs/app-server-protocol/src/protocol/v2/thread.rs`
- `codex-rs/thread-store/src/local/archive_thread.rs`
- `codex-rs/thread-store/src/local/unarchive_thread.rs`

## 2. Thread-store 开始保存 canonical permission profile

`StoredThread` 现在直接保存 `permission_profile: PermissionProfile`。这让 list/read/resume 视角下的 thread metadata 不再只依赖旧的 approval/sandbox 字段推导权限。

学习上的调整：

- `turn_context.permission_profile` 仍然是“某个 turn 当时如何运行”的快照。
- `StoredThread.permission_profile` 是 thread-store 摘要层的“最新/规范权限视图”，用于列表、恢复和外部 API。
- 旧记录仍可从 `sandbox_policy`、`file_system_sandbox_policy`、`network` 等字段推导 profile，但新路径会尽量保留 profile 本身。

关键源码：

- `codex-rs/thread-store/src/types.rs`
- `codex-rs/thread-store/src/local/read_thread.rs`
- `codex-rs/state/src/extract.rs`
- `codex-rs/protocol/src/protocol.rs`

## 3. JSONL 的 `turn_context` 落盘字段收窄并加入 workspace roots

当前 `TurnContextItem` 的落盘结构和旧笔记相比有两点重要变化：

- 新增 `workspace_roots`，用于把 permission profile 里的符号化 `:workspace_roots` 文件系统权限物化到恢复语义里。
- `summary` 变成兼容字段；源码注释说明它只是为了旧版本能反序列化，不再参与 context reconstruction。

同时，旧笔记里把一些运行时 `TurnContext` 概念写进了 JSONL payload，例如 `trace_id`、`user_instructions`、`developer_instructions`、`final_output_json_schema`、`truncation_policy`。这些不在当前 `protocol::TurnContextItem` 的落盘结构里，应该从“JSONL schema”笔记中移除或降级为运行时上下文概念。

关键源码：

- `codex-rs/protocol/src/protocol.rs`
- `codex-rs/core/src/session/turn_context.rs`
- `codex-rs/core/src/session/rollout_reconstruction.rs`

## 4. Config 变成更明确的分层系统

这次合入包含两条相关线：

- cloud-managed config layer：新增 `ConfigLayerSource::EnterpriseManaged { id, name }`。
- requirements layers composition：把 rules、hooks、permissions 等 requirements 从多层来源组合出来。

新的心智模型：

```text
MDM / system / enterprise-managed / user / profile / project / session flags
  -> 按 precedence 合成普通 config
  -> requirements layers 额外合成 rules、hooks、permissions
  -> runtime Config / permission profile / hook requirements
```

学习重点：

- config layer source 现在不仅解释“配置值从哪里来”，还要能面向诊断展示云管控来源。
- requirements 不是普通 TOML 覆盖就完事；rules、hooks、permissions 有领域合并逻辑。
- managed config 能限制权限、hooks 和 sandbox 能力，是权限闭环的一部分。

关键源码：

- `codex-rs/config/src/config_layer_source.rs`
- `codex-rs/config/src/cloud_config_layers.rs`
- `codex-rs/config/src/requirements_layers/`
- `codex-rs/app-server-protocol/src/protocol/v2/config.rs`

## 5. `request_user_input` 变成配置可控工具

新增 `experimental_request_user_input` 配置开关后，`request_user_input` 不再只是“存在于工具系统里”的能力，而是会被 `spec_plan` 按配置决定是否暴露。

学习重点：

- 这是任务信息补全工具，不是权限审批。
- 它是否出现在模型可见工具列表里，取决于配置和客户端能力。
- 文档里讨论 request user input 时，需要区分“运行时事件/等待点”和“模型可见工具是否启用”。

关键源码：

- `codex-rs/config/src/config_toml.rs`
- `codex-rs/core/src/config/mod.rs`
- `codex-rs/core/src/tools/spec_plan.rs`
- `codex-rs/core/src/tools/handlers/request_user_input.rs`

## 6. Multi-agent v2 命名从 followup task 改成 assign task

multi-agent v2 的一个工具命名从 `followup_task` 调整为 `assign_task`。这是一个小改动，但对学习笔记很重要，因为它改变了业务语义：

- `followup_task` 听起来像“已有任务后的追问”。
- `assign_task` 更明确表示“父 agent 给子 agent 分配任务”。

更新笔记时，v2 工具列表应写成 list / spawn / assign_task / send_message / wait / close 等语义，而不是继续使用 followup。

关键源码：

- `codex-rs/core/src/tools/handlers/multi_agents_v2/assign_task.rs`
- `codex-rs/core/src/tools/handlers/multi_agents_v2/message_tool.rs`
- `codex-rs/core/src/tools/handlers/multi_agents_spec.rs`

## 7. 对 Web Word Agent 的新增借鉴

如果把这些变化映射到 Web Word agent：

1. 会话列表要把 active/archived 做成明确生命周期，而不是删除或隐藏旧记录。
2. thread metadata 应保存最新权限 profile，方便恢复、审计和列表展示。
3. turn context snapshot 要保存文档 workspace roots / document roots，不能只保存 cwd。
4. 企业/组织策略应作为独立 config layer，和用户配置、项目配置分层合成。
5. 业务补问工具要有开关，并和权限审批分开设计。
6. 多 agent 的“分配任务”应作为显式动作，区别于普通消息和追问。
