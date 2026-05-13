# 源码与原始笔记对照

这个文件用于以后回查。综合专题正文尽量用通用语言表达；需要细节时再回到原始笔记和源码。

## 1. 原始笔记对照

| 综合问题 | 原始笔记 |
| --- | --- |
| agent 的整体业务闭环 | `../core-agent-business-logic-guide.md` |
| DDD 领域对象和闭环 | `../core-agent-ddd-closed-loops.md` |
| 工程架构和 crate 分工 | `../agent-development-architecture-guide.md` |
| 完整功能场景 | `../agent-feature-scenarios.md` |
| 方案设计取舍 | `../agent-design-tradeoffs.md` |
| 技术学习路线 | `../agent-implementation-roadmap.md` |
| Turn 领域深潜 | `../turn-domain-deep-dive.md` |
| thread JSONL 历史结构 | `../thread-jsonl-history-schema.md` |
| 历史问题归纳 | `../history-questions-dialogue.md` |
| 首次采样 prompt | `../first-sampling-prompt-structure.md` |
| 模型返回和 turn lifecycle | `../model-response-turn-lifecycle.md` |
| item 保存和后续采样 | `../turn-items-persistence-and-resampling.md` |
| event_msg 链路 | `../event-msg-production-consumption.md` |
| 工具发现、调用、回灌 | `../turn-tool-discovery-call-model-flow.md` |
| Web Word 实现蓝图 | `../web-word-agent-implementation-blueprint.md` |

## 2. 源码入口对照

| 主题 | 源码入口 |
| --- | --- |
| 协议基础 | `codex-rs/protocol/src/protocol.rs` |
| app-server v2 thread/turn API | `codex-rs/app-server-protocol/src/protocol/v2/` |
| app-server turn 处理 | `codex-rs/app-server/src/request_processors/turn_processor.rs` |
| thread 管理 | `codex-rs/core/src/thread_manager.rs` |
| thread facade | `codex-rs/core/src/codex_thread.rs` |
| session 操作循环 | `codex-rs/core/src/session/handlers.rs` |
| session 状态 | `codex-rs/core/src/session/session.rs` |
| turn runner | `codex-rs/core/src/session/turn.rs` |
| turn context | `codex-rs/core/src/session/turn_context.rs` |
| prompt 构造和采样 | `codex-rs/core/src/session/turn.rs` |
| 模型 stream item 处理 | `codex-rs/core/src/stream_events_utils.rs` |
| response/input item 类型 | `codex-rs/protocol/src/models.rs` |
| tool router | `codex-rs/core/src/tools/router.rs` |
| tool registry | `codex-rs/core/src/tools/registry.rs` |
| tool runtime 并发与取消 | `codex-rs/core/src/tools/parallel.rs` |
| tool context/output 类型 | `codex-rs/core/src/tools/context.rs` |
| tool spec 构造 | `codex-rs/core/src/tools/spec.rs` |
| plan tool spec | `codex-rs/core/src/tools/spec_plan.rs` |
| MCP tool exposure | `codex-rs/core/src/mcp_tool_exposure.rs` |
| MCP connection manager | `codex-rs/codex-mcp/src/mcp_connection_manager.rs` |
| 多 agent | `codex-rs/core/src/agent/` |
| 多 agent tool handlers | `codex-rs/core/src/tools/handlers/multi_agents*.rs` |
| 持久化/rollout | `codex-rs/core/src/rollout/` |
| 配置和权限 | `codex-rs/core/src/config/`, `codex-rs/core/src/exec_policy.rs` |

## 3. 后续学习建议

如果以后要继续深入，不建议从头逐行读源码。更高效的方式是按问题回查：

| 新问题 | 建议先看 |
| --- | --- |
| 为什么某个历史没进 prompt | `thread-jsonl-history-schema.md`、`turn-items-persistence-and-resampling.md` |
| 为什么模型能/不能调用某个工具 | `turn-tool-discovery-call-model-flow.md` |
| 为什么 UI 看到了事件但模型不知道 | `event-msg-production-consumption.md` |
| 为什么一个 turn 采样多次 | `turn-domain-deep-dive.md`、`model-response-turn-lifecycle.md` |
| Web Word 应该怎么设计工具 | `web-word-agent-implementation-blueprint.md`、本目录 `06-web-word-agent-design.md` |

