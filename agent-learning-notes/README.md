# Codex Agent 学习笔记索引

这个目录用于系统化学习本仓库里的 agent 能力。阅读顺序建议从“业务闭环”到“源码入口”，再到“方案权衡”和“扩展实践”。

## 已完成笔记

| 文件 | 解决的问题 | 适合什么时候读 |
| --- | --- | --- |
| `core-agent-business-logic-guide.md` | 用业务语言解释核心 agent 层如何接收目标、构造上下文、调用模型、执行工具、处理审批、保存历史。 | 第一次建立整体概念。 |
| `core-agent-ddd-closed-loops.md` | 用 DDD 方式拆解 thread、session、turn、tool call、MCP、多 agent、持久化等领域闭环。 | 想理解领域边界和状态变化。 |
| `agent-development-architecture-guide.md` | 从 CLI、TUI、app-server、core、tools、MCP、skills、plugins、多 agent、持久化串起工程架构。 | 准备读源码或做开发。 |
| `agent-feature-scenarios.md` | 按真实功能场景列出用户入口、核心流程、源码位置、设计风险。 | 想知道“agent 到底有哪些完整场景”。 |
| `agent-design-tradeoffs.md` | 汇总本仓库 agent 方案里的关键设计取舍、为什么这么做、替代方案和代价。 | 做方案设计或评审时参考。 |
| `agent-implementation-roadmap.md` | 给出按源码入口学习和扩展 agent 功能的路线图。 | 准备修改代码、定位问题、加能力。 |
| `web-word-agent-implementation-blueprint.md` | 把 Codex 的 agent 设计抽象成可用于自研 Web Word 产品的落地蓝图。 | 准备自己实现 agent，而不是维护 Codex 仓库。 |
| `pure-frontend-agent-design.md` | 基于本目录学习成果，给出纯前端 agent 的技术方案、架构方案、产品功能交互方案，并重点说明扁平 DDD core 如何串联调度。 | 准备评审或落地浏览器内 agent 架构。 |
| `turn-domain-deep-dive.md` | 深入拆解 Turn 的业务逻辑、状态、流程、事件、工具回灌和中断。 | 设计自己 agent 的“单次任务执行”核心。 |
| `thread-jsonl-history-schema.md` | 用 TypeScript 表达 thread JSONL 里会保存的历史结构，并说明每类结构的来源、含义和恢复语义。 | 想搞清 thread/session/turn/prompt 里的“历史”分别是什么。 |
| `first-sampling-prompt-structure.md` | 详细拆解 regular user turn 首次发给模型的 prompt/request 包含哪些信息、来源和处理逻辑，并给出 TypeScript 数据结构。 | 想自己实现 prompt 构造器和 Web Word agent 首次采样请求。 |
| `model-response-turn-lifecycle.md` | 解释模型收到复合 prompt 后如何通过 stream 返回推理、工具、plan、用户输入请求和最终消息，以及 agent 如何转成事件、历史和 turn complete。 | 想理解模型返回后的处理过程和 Web Word agent 的运行时事件模型。 |
| `turn-items-persistence-and-resampling.md` | 解释 turn 过程中的 item 如何写入 thread JSONL、哪些会保存、保存前后如何处理，以及哪些会在同 turn 或后续 turn 的采样中再次发给模型。 | 想把 JSONL 运行日志、session 内存历史和模型 prompt 历史彻底分清。 |
| `event-msg-production-consumption.md` | 专门拆解 `event_msg` 的生产者、统一发送链路、落盘策略、app-server 实时消费、JSONL 重放消费，以及它如何间接影响模型历史。 | 看完 item/历史笔记后，想继续搞清 UI 事件、控制流事件和持久化事件。 |
| `turn-tool-discovery-call-model-flow.md` | 详细列出 turn 期间工具的发现、direct/deferred 暴露、ToolRouter/ToolRegistry、模型 tool call、ToolRuntime 执行、权限/hooks、结果回灌和后续采样逻辑。 | 想设计自己 agent 的工具系统，尤其是 Web Word 的读写工具、审批和动态工具。 |
| `history-questions-dialogue.md` | 记录本轮围绕“历史”的连续疑问、问题脉络和当前结论。 | 想回看自己为什么会困惑，以及每个问题对应哪份笔记。 |
| `completion-audit.md` | 把用户目标映射到目录内具体产物和证据。 | 检查学习资料覆盖面。 |
| `codex-agent-learning-synthesis/` | 把本次围绕 Codex agent 的完整学习过程二次整理成连续专题：学习地图、领域模型、turn/history、prompt/model/tools、持久化/UI 事件、Web Word 落地。 | 想快速回顾整轮学习结论，而不是逐篇翻原始笔记。 |

## 一条推荐学习路径

如果目标是自己实现一个和公司 Web Word 产品结合的 agent，推荐这样读：

1. 先读 `web-word-agent-implementation-blueprint.md`，把目标落到自己的产品架构、工具、权限和数据模型上。
2. 读 `pure-frontend-agent-design.md`，把 Web Word agent 进一步收敛成浏览器内可落地的技术、架构和产品交互方案。
3. 再读 `core-agent-business-logic-guide.md`，建立“agent 是任务执行闭环，不只是模型调用”的心智模型。
4. 读 `turn-domain-deep-dive.md`，重点掌握一次用户任务如何启动、运行、调用工具、等待审批、完成或中断。
5. 再读 `agent-feature-scenarios.md`，把聊天、代码修改、审批、MCP、skills、plugins、多 agent、resume/fork 等能力抽象成可复用场景。
6. 然后读 `agent-design-tradeoffs.md`，理解哪些设计值得借鉴，哪些对 Web Word 可以简化。
7. 最后读 `agent-development-architecture-guide.md` 和 `agent-implementation-roadmap.md`，在需要对照 Codex 源码时查入口。

## 本目录对“完整 agent 功能”的定义

这里的“完整”不是逐行复制源码，而是覆盖 agent 运行所需的主要能力面：

- 入口：CLI、TUI、exec、app-server、MCP server。
- 会话：thread start/resume/fork/archive/read/list、turn start/steer/interrupt。
- 上下文：历史、AGENTS.md、skills、plugins、apps/connectors、environment、collaboration mode、personality、output schema。
- 模型：provider、Responses API、流式事件、重试、fallback、token usage。
- 工具：shell、apply_patch、MCP、dynamic tools、tool search、image/view image、plan、goal、request user input、permissions。
- 安全：approval policy、permission profile、sandbox、network policy、guardian、hooks。
- 多 agent：spawn/send/wait/close/resume、roles、registry、mailbox、fork context、limits。
- 持久化：rollout、thread-store、state db、rollback、compact、token replay。
- 观测：event stream、app-server v2 notifications、analytics、trace。

## 笔记表达约定

后续学习笔记优先用通用业务语言和通用流程图表达，源码路径只放在“源码对照”或附录中。正文避免堆叠类名、函数名和实现细节，重点回答：

- 这个领域对象解决什么业务问题。
- 它有哪些状态和边界。
- 它如何与模型、工具、权限、事件、持久化协作。
- 如果映射到 Web Word agent，应该如何设计。

## 重要源码入口

- Core protocol: `codex-rs/protocol/src/protocol.rs`
- app-server v2 protocol: `codex-rs/app-server-protocol/src/protocol/v2/`
- app-server request processors: `codex-rs/app-server/src/request_processors/`
- Thread manager: `codex-rs/core/src/thread_manager.rs`
- Thread facade: `codex-rs/core/src/codex_thread.rs`
- Session loop: `codex-rs/core/src/session/handlers.rs`
- Turn runner: `codex-rs/core/src/session/turn.rs`
- Turn context: `codex-rs/core/src/session/turn_context.rs`
- Tool router/orchestrator: `codex-rs/core/src/tools/router.rs`, `codex-rs/core/src/tools/orchestrator.rs`
- Multi agent: `codex-rs/core/src/agent/`, `codex-rs/core/src/tools/handlers/multi_agents*.rs`
- MCP: `codex-rs/codex-mcp/src/connection_manager.rs`, `codex-rs/core/src/session/mcp.rs`
- Config and permissions: `codex-rs/core/src/config/`, `codex-rs/core/src/exec_policy.rs`
