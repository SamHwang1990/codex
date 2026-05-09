# Agent 学习目标完成度审计

审计时间：2026-05-08

## 目标复述

用户目标是：通过当前仓库学习完整的 agent 功能场景、方案设计与权衡、技术实现思路，核心用途不是维护 Codex 仓库，而是为自研 Web Word 产品实现 agent，并把学习结果全部落盘到 `agent-learning-notes` 目录。

可验证交付物：

1. `agent-learning-notes` 目录存在。
2. 目录内有学习 agent 功能场景的笔记。
3. 目录内有学习方案设计与权衡的笔记。
4. 目录内有学习技术实现思路和源码入口的笔记。
5. 目录内有面向自研 Web Word agent 的落地实现蓝图。
6. 笔记覆盖已有仓库中 agent 的主要能力面，而不是只覆盖单点功能。
7. 笔记基于当前仓库源码，有具体源码路径作为证据。

## Prompt 到产物映射

| 用户要求 | 对应产物 | 证据 |
| --- | --- | --- |
| “通过这个仓库学习” | 所有笔记均引用 `codex-rs/` 下源码路径 | `README.md`、`agent-feature-scenarios.md`、`agent-implementation-roadmap.md` |
| “完整的 agent 功能场景” | `agent-feature-scenarios.md` | 覆盖交互式协作、exec、thread、turn override、模型流、工具、shell、patch、审批、MCP、apps、skills、plugins、多 agent、goal、user input、compact/rollback/resume/fork、realtime、事件、测试 |
| “方案设计与权衡” | `agent-design-tradeoffs.md` | 覆盖 app-server 边界、SQ/EQ、thread/session/turn、TurnContext、PermissionProfile、ToolRouter、ToolOrchestrator、MCP、skills、多 agent、持久化、compact、v2 API、codex-core 膨胀等取舍 |
| “技术实现思路” | `agent-implementation-roadmap.md` | 按协议、app-server、core lifecycle、turn、tools、安全、MCP/apps/plugins/skills、多 agent、持久化、UI、扩展任务给出源码路线 |
| “自己实现一个 agent，和公司自己的产品（web word）结合起来” | `web-word-agent-implementation-blueprint.md` | 把 Codex 的 thread/turn/tool/approval/event/persistence 设计映射到 Web Word 的文档、选区、批注、建议、版本和权限场景 |
| “深入学习 Turn 这个领域的业务逻辑、细节、流程” | `turn-domain-deep-dive.md` | 覆盖 Turn 协议、app-server 边界、TurnContext、ActiveTurn、TurnState、Task、run_turn、模型流、工具回灌、steer、中断、完成、事件和 Web Word 映射 |
| “统统落盘到 agent-learning-notes” | `agent-learning-notes/README.md` 索引所有文件 | 目录内已有 10 个 `.md` 文件 |

## 覆盖面检查

| 能力面 | 覆盖文件 |
| --- | --- |
| CLI/TUI/exec/app-server 入口 | `agent-development-architecture-guide.md`、`agent-feature-scenarios.md`、`agent-implementation-roadmap.md` |
| app-server v2 API | `agent-development-architecture-guide.md`、`agent-design-tradeoffs.md`、`agent-implementation-roadmap.md` |
| Thread/Session/Turn/Item | `core-agent-business-logic-guide.md`、`core-agent-ddd-closed-loops.md`、`agent-feature-scenarios.md` |
| 模型采样与事件流 | `agent-development-architecture-guide.md`、`agent-feature-scenarios.md`、`agent-implementation-roadmap.md` |
| 工具系统 | `agent-development-architecture-guide.md`、`core-agent-ddd-closed-loops.md`、`agent-feature-scenarios.md`、`agent-design-tradeoffs.md` |
| 审批、权限、沙箱、guardian | `core-agent-business-logic-guide.md`、`agent-feature-scenarios.md`、`agent-design-tradeoffs.md` |
| MCP、apps/connectors | `agent-development-architecture-guide.md`、`agent-feature-scenarios.md`、`agent-implementation-roadmap.md` |
| Skills、AGENTS.md、plugins | `agent-development-architecture-guide.md`、`agent-feature-scenarios.md`、`agent-design-tradeoffs.md` |
| 多 agent | `agent-development-architecture-guide.md`、`core-agent-ddd-closed-loops.md`、`agent-feature-scenarios.md`、`agent-implementation-roadmap.md` |
| 持久化、resume、fork、rollback、compact | `core-agent-business-logic-guide.md`、`agent-feature-scenarios.md`、`agent-design-tradeoffs.md` |
| 测试与验证入口 | `agent-feature-scenarios.md`、`agent-implementation-roadmap.md` |
| 自研 Web Word agent 落地 | `web-word-agent-implementation-blueprint.md` |
| Turn 领域深潜 | `turn-domain-deep-dive.md` |

## 实际源码证据

本次补写前后检查过的关键源码包括：

- `codex-rs/protocol/src/protocol.rs`
- `codex-rs/app-server-protocol/src/protocol/v2/thread.rs`
- `codex-rs/app-server-protocol/src/protocol/v2/turn.rs`
- `codex-rs/app-server/src/request_processors/turn_processor.rs`
- `codex-rs/core/src/thread_manager.rs`
- `codex-rs/core/src/codex_thread.rs`
- `codex-rs/core/src/session/handlers.rs`
- `codex-rs/core/src/session/session.rs`
- `codex-rs/core/src/session/turn.rs`
- `codex-rs/core/src/tools/router.rs`
- `codex-rs/core/src/tools/orchestrator.rs`
- `codex-rs/core/src/tools/handlers/multi_agents.rs`
- `codex-rs/core/src/agent/control.rs`

## 剩余风险

这些笔记覆盖的是 agent 架构和主要功能场景，不是逐函数 API 文档。若以后仓库继续演进，尤其是 app-server v2 schema、多 agent v2、MCP 连接管理或权限 profile 发生较大变化，需要重新抽样源码并更新对应章节。
