# 核心 Agent 层的 DDD 闭环建模

这份文档用 DDD 的思想整理 Codex 核心 agent 层的业务概念。它不是代码结构说明，而是把核心层看成一个领域系统：有哪些限界上下文、聚合、实体、值对象、领域服务、领域事件，以及每个闭环如何完成业务目标。

这里的“实体”不完全等同于源码里的 struct。本文更关注逻辑实体：只要它有身份、生命周期、状态变化、业务规则，就把它当成领域实体或聚合来理解。

## 1. 总览：核心 Agent 层的领域地图

核心 agent 层可以拆成这些限界 Context：

1. **会话**：管理 thread/session/turn 的生命周期。
2. **任务执行**：把用户目标转成模型采样、工具调用和最终结果。
3. **上下文构造**：决定模型本轮看到什么。
4. **工具调用**：管理工具暴露、路由、执行、结果回灌。
5. **权限与审批**：决定副作用能否发生。
6. **外部能力**：管理 MCP、apps/connectors、dynamic tools。
7. **知识注入**：管理 AGENTS.md、skills、plugins 对模型行为的影响。
8. **持久化与恢复**：保存、恢复、fork、rollback、archive。
9. **多 Agent **：管理 sub-agent 的创建、通信、等待和关闭。
10. **目标与记忆**：管理跨 turn 的 goal、memory、长期状态。
11. **事件与观测**：向上层界面、日志、trace、analytics 输出领域事实。

这些 Context 不是相互隔离的微服务，而是一个进程内领域模型。它们通过一些核心对象协作：

- `Thread` 是长期业务边界。
- `Session` 是运行时业务边界。
- `Turn` 是一次任务处理边界。
- `Item` 是过程记录边界。
- `ToolCall` 是副作用请求边界。
- `PermissionProfile` 是能力边界。
- `Event` 是领域事实边界。

## 2. 会话闭环

### 业务目标

把用户的一段持续工作组织成可继续、可恢复、可 fork、可归档的会话。

### 聚合根：Thread

Thread 是会话闭环的聚合根。它代表一条可持久化的 agent 工作线。

身份：

- thread id。

核心状态：

- 当前工作目录。
- 当前模型和 provider。
- 当前权限配置。
- thread source，例如 TUI、app-server、sub-agent。
- thread name。
- memory mode。
- goal。
- 历史 turns/items。
- 是否 ephemeral。
- fork 来源。
- archive 状态。

能力：

- start：创建新 thread。
- resume：从持久化历史恢复 thread。
- fork：基于旧 thread 创建新 thread。
- archive/unarchive：改变可见生命周期。
- read/list：查询 thread。
- rename：更新可读名称。
- update metadata：更新 git info、memory mode 等元数据。
- shutdown：释放运行时资源。

业务规则：

- thread id 是持久身份，不能依赖路径作为唯一身份。
- ephemeral thread 不应依赖长期落盘。
- resume 时必须尊重历史和配置快照，不能把运行中 thread 的半截状态当作完成状态。
- fork 必须明确历史切点，不能隐式复制一个不一致的中间状态。
- archive 只改变 thread 的管理状态，不应该破坏历史可读性。

### 实体：Session

Session 是 thread 被加载到内存后的运行实体。

身份：

- session id。
- 所属 thread id。

核心状态：

- active turn。
- conversation history。
- MCP connection manager。
- tools manager。
- skills/plugins manager。
- model client。
- pending approvals。
- pending dynamic tool responses。
- running background processes。
- rollout writer。
- agent control。

能力：

- 接收 operation。
- 发出 event。
- 创建 turn。
- 中断 active turn。
- 关闭 MCP 和进程。
- 刷新配置或 MCP。
- 持久化历史。

业务规则：

- 一个 session 同一时刻只能有受控的 active turn 状态。
- session 关闭时必须清理它拥有的子进程、MCP client、后台任务和持久化 writer。
- session 是运行时状态，不等于持久 thread；恢复时可以创建新的 session。

### 值对象：ThreadConfigSnapshot

表示某一刻 thread 的有效运行配置。

包含：

- 模型。
- provider。
- approval policy。
- permission profile。
- cwd。
- reasoning effort。
- personality。
- service tier。
- session source。

业务意义：

- 给 resume、fork、turn override 校验提供稳定参照。
- 让上层知道当前真正生效的配置，而不是用户原始请求。

### 领域服务：ThreadManager

ThreadManager 是会话闭环的领域服务。它不只是集合管理器，而是 thread 创建、恢复、fork、运行时注册和共享资源注入的协调者。

能力：

- 创建 Thread + Session。
- 加载已有 thread。
- 管理内存中的 thread registry。
- 注入共享服务：auth、models、environment、skills、plugins、MCP、thread store。
- 为 sub-agent 提供 spawn 能力。

### 领域事件

- ThreadStarted。
- ThreadResumed。
- ThreadForked。
- ThreadArchived。
- ThreadUnarchived。
- ThreadClosed。
- ThreadStatusChanged。
- ThreadMetadataUpdated。

### 闭环流程

```text
请求开始/恢复会话
-> ThreadManager 解析配置和历史
-> 创建或加载 Thread
-> 创建 Session
-> 注册运行时资源
-> 发出 thread started/resumed 事件
-> thread 进入可接收 turn 的状态
```

## 3. Turn 执行闭环

### 业务目标

把一次用户输入处理成一个完整工作结果。

### 聚合根：Turn

Turn 是一次用户请求的执行聚合。它不是简单消息，而是“本轮任务的全部过程”。

身份：

- turn id，通常对应提交操作 id。

核心状态：

- status：in progress、completed、interrupted、failed。
- 用户输入。
- 本轮上下文。
- 本轮模型输出。
- 本轮工具调用。
- 本轮审批。
- token usage。
- start/completion time。
- error。

能力：

- start：开始处理用户输入。
- steer：接收运行中的补充输入。
- interrupt：中断当前执行。
- append item：记录过程项。
- complete：正常结束。
- fail：异常结束。

业务规则：

- 一个用户 turn 可以包含多次模型请求。
- 工具调用完成后通常需要继续采样，直到模型给出最终回答。
- review/compact 这类特殊 turn 不能随便接受 steer。
- turn 中断后必须记录中断事实，避免恢复时误判。
- token usage 属于 turn 的结果元数据，不能只看最终消息。

### 实体：TurnContext

TurnContext 是 turn 的运行时任务包。

包含：

- cwd。
- model info。
- approval policy。
- permission profile。
- tools config。
- collaboration mode。
- dynamic tools。
- final output schema。
- telemetry state。
- cancellation token。
- turn skills。

能力：

- 为本轮构造 prompt。
- 判断工具可见性。
- 提供工具执行参数。
- 记录本轮 token/trace/metrics。

业务规则：

- TurnContext 是本轮有效配置，不应该被后续 turn 直接复用。
- 模型、权限、工具、输出 schema 等都必须以 TurnContext 为准。

### 实体：ActiveTurn

ActiveTurn 表示 session 当前正在执行的 turn。

能力：

- 接收 steer 输入。
- 响应 interrupt。
- 暴露当前 turn id。
- 控制 mailbox delivery。

业务规则：

- active turn 是互斥运行状态。
- 不可 steer 的 turn 必须拒绝 steer，而不是默默吞掉输入。

### 领域服务：TurnRunner

这是逻辑服务，不一定是单个源码类型。它代表“如何跑完一个 turn”的业务流程。

能力：

- 创建本轮 prompt。
- 发起模型采样。
- 消费模型流事件。
- 调度工具。
- 回灌工具结果。
- 判断是否需要继续采样。
- 结束或失败 turn。

### 领域事件

- TurnStarted。
- TurnCompleted。
- TurnAborted。
- TurnFailed。
- TurnDiffProduced。
- TokenUsageUpdated。
- StreamError。

### 闭环流程

```text
用户输入
-> 创建 TurnContext
-> 构造模型上下文
-> 调用模型
-> 接收模型输出
-> 如果有工具调用，进入工具闭环
-> 工具结果回灌给模型
-> 继续采样或完成
-> 记录 token usage 和最终状态
-> 发出 turn completed/failed/interrupted
```

## 4. 上下文构造闭环

### 业务目标

让模型看到“完成本轮任务所需且被允许看到的信息”。

### 聚合根：PromptContext

PromptContext 是逻辑聚合，不一定有同名源码结构。它代表一次模型请求的完整上下文包。

核心组成：

- 用户输入。
- 历史消息。
- 系统/开发者/项目指令。
- AGENTS.md。
- skills 注入。
- plugins/apps 说明。
- 当前工作环境。
- 权限和审批说明。
- 可用工具规格。
- 输出 schema。
- personality/collaboration mode。

能力：

- 合并多来源指令。
- 选择历史片段。
- 注入当前环境。
- 注入工具说明。
- 控制上下文大小。
- 为不同模型能力调整输入。

业务规则：

- 持久化历史不等于模型可见历史。
- 模型不应看到当前权限不允许或本轮不相关的工具。
- 项目指令、用户指令、开发者指令需要有明确优先级。
- 大输出和长历史需要压缩或截断。
- 如果启用 plan/review/compact 等模式，本轮上下文应体现模式差异。

### 实体：ConversationHistory

代表已发生的对话和工具事实。

能力：

- append response item。
- clone for prompt。
- reconstruct from rollout。
- compact。
- rollback。

业务规则：

- 已完成事实应该保存。
- 被中断事实也应该保存，但必须带中断语义。
- 给模型的历史可以经过筛选和压缩，但不能编造未发生的事实。

### 实体：InstructionSource

逻辑实体，代表一类指令来源。

常见来源：

- base instructions。
- developer instructions。
- AGENTS.md。
- collaboration mode instructions。
- skills。
- plugin/app instructions。
- permissions instructions。

业务规则：

- 指令来源需要保留来源感，便于解释和调试。
- 指令冲突时必须遵守优先级。

### 值对象：ToolSpecSet

表示本轮模型可见工具规格集合。

业务规则：

- 工具规格是模型合约。
- 延迟加载工具不应提前暴露。
- 同名工具、同名 skill/app mention 需要消歧。

### 领域服务：ContextManager

管理历史和上下文预算。

能力：

- 选择 prompt 输入。
- 统计 token。
- 处理 compact 后历史。
- 维护上下文使用状态。

### 领域事件

- ContextBuilt。
- ContextCompacted。
- TokenCountUpdated。
- InstructionLoaded。
- SkillInjected。

### 闭环流程

```text
TurnContext 准备完成
-> 收集指令、历史、环境、工具
-> 过滤和压缩
-> 构造 PromptContext
-> 发给模型
-> 模型输出和工具结果再进入历史
-> 下一次采样重新构造上下文
```

## 5. 工具调用闭环

### 业务目标

把模型的“意图”转成受控的真实动作，并把结果反馈回模型。

### 聚合根：ToolCall

ToolCall 是工具调用闭环的聚合根。

身份：

- call id。

核心状态：

- tool name。
- namespace。
- raw arguments。
- parsed payload。
- source。
- status：requested、approved、running、completed、failed、denied、cancelled。
- result。
- error。

能力：

- parse。
- validate。
- request approval。
- execute。
- stream output。
- complete。
- convert result to model input。

业务规则：

- 模型请求的工具必须先被路由和校验。
- 无法解析的参数应反馈给模型，而不应该导致整个 session 崩溃。
- 有副作用的工具必须经过权限/审批闭环。
- 工具结果必须变成模型可消费内容。
- 工具执行产生的过程也必须可见和可持久化。

### 实体：ToolRouter

ToolRouter 是工具选择和路由实体。

能力：

- 根据模型输出识别工具类型。
- 区分内置 function、local shell、MCP、dynamic tool、tool search、custom tool。
- 查询工具规格。
- 判断是否支持并行。
- 创建参数 diff consumer。

业务规则：

- 路由失败时要给出可反馈给模型的错误。
- MCP 工具和普通 function 可能共享形态，必须按 server/tool metadata 解析。

### 实体：ToolRegistry

ToolRegistry 代表系统知道如何执行哪些工具。

能力：

- 注册工具 handler。
- 分发工具调用。
- 返回统一工具结果。

### 实体：ToolResult

工具执行结果。

常见内容：

- 文本输出。
- 结构化 JSON。
- exit code。
- duration。
- stdout/stderr。
- 文件变更。
- MCP content items。
- 错误原因。

业务规则：

- 给模型的结果要足够明确。
- 大结果需要截断。
- 失败结果也应作为事实反馈给模型。

### 领域服务：ToolRuntime

协调工具执行过程。

能力：

- 并行或串行执行工具。
- 挂接 cancellation。
- 关联 turn/session。
- 记录 telemetry。
- 产出事件。

### 领域事件

- ToolCallStarted。
- ToolCallDelta。
- ToolCallCompleted。
- ToolCallFailed。
- ToolCallDenied。
- ToolOutputProduced。

### 闭环流程

```text
模型输出工具调用
-> ToolRouter 识别工具
-> ToolRegistry 找到执行器
-> 权限与审批闭环判断能否执行
-> ToolRuntime 执行
-> 产生工具事件和结果
-> 结果转为模型输入
-> 回到 Turn 执行闭环继续采样
```

## 6. 权限与审批闭环

### 业务目标

在保证 agent 能完成任务的同时，控制文件、网络、命令等副作用风险。

### 聚合根：PermissionDecision

PermissionDecision 是逻辑聚合，代表一次副作用请求是否允许。

核心状态：

- 请求来源。
- 请求动作。
- 当前 approval policy。
- 当前 permission profile。
- sandbox policy。
- network policy。
- 是否命中已批准规则。
- 是否需要用户审批。
- 用户决策。

能力：

- evaluate。
- request approval。
- apply decision。
- persist approved rule。
- deny。

业务规则：

- 权限判断必须发生在工具执行前。
- 用户拒绝后不能绕过。
- 会话级批准和一次性批准语义不同。
- network、filesystem、shell command 的风险维度不同。
- managed config 可以限制用户请求的权限。

### 实体：PermissionProfile

表示当前运行能力边界。

包含：

- 文件系统读写规则。
- 网络规则。
- sandbox enforcement。
- 额外权限。

能力：

- 推导 sandbox policy。
- 判断文件/网络能力。
- 与 runtime 环境结合。

### 实体：ApprovalRequest

代表一次需要用户参与的决策。

类型：

- exec approval。
- patch approval。
- request permissions。
- MCP elicitation。
- network approval。

能力：

- 展示风险。
- 接收用户决定。
- 转成后续执行许可或拒绝结果。

### 值对象：ApprovalPolicy

决定什么时候问用户。

业务意义：

- never：不询问，不能做就失败或降级。
- on-request：需要时询问。
- unless-trusted / auto-review：在特定条件下自动处理。

### 领域事件

- ApprovalRequested。
- ApprovalGranted。
- ApprovalDenied。
- PermissionProfileChanged。
- NetworkRuleSaved。
- CommandPrefixApproved。

### 闭环流程

```text
工具请求副作用
-> 收集当前权限和审批策略
-> 判断是否已允许
-> 若需要用户，发出审批请求
-> 用户响应
-> 允许则执行工具
-> 拒绝则把拒绝结果反馈给模型
-> 可选保存会话级规则
```

## 7. Shell 执行闭环

### 业务目标

让 agent 能运行命令，同时控制平台差异、权限、沙箱、输出规模和失败语义。

### 聚合根：CommandExecution

身份：

- call id 或 process id。

核心状态：

- command。
- cwd。
- env。
- timeout。
- sandbox type。
- network setting。
- stdout/stderr。
- exit code。
- duration。
- timed out。
- status。

能力：

- normalize command。
- select sandbox。
- spawn。
- stream output。
- terminate。
- aggregate output。
- format result for model。

业务规则：

- 命令不能为空。
- 命令 cwd 必须符合当前权限。
- 平台沙箱选择必须按当前 permission profile 推导。
- 超时不是普通失败，需要明确告诉模型。
- 输出给模型前要截断，但事件流可以展示过程。

### 逻辑实体：SandboxTransform

把抽象权限转换成平台可执行命令。

平台差异：

- macOS Seatbelt。
- Linux bubblewrap/Landlock。
- Windows restricted/elevated sandbox。
- 无沙箱。

业务规则：

- 不能因为平台不支持就静默降级到更危险权限。
- 运行失败应给出明确错误。

### 领域事件

- CommandStarted。
- CommandOutputDelta。
- CommandCompleted。
- CommandTimedOut。
- CommandFailed。

### 闭环流程

```text
模型请求运行命令
-> 工具路由识别 shell call
-> 权限与审批闭环
-> 选择沙箱和环境
-> 启动进程
-> 流式收集输出
-> 聚合 exit code/duration/output
-> 格式化给模型
```

## 8. MCP / 外部能力闭环

### 业务目标

把外部系统能力接入 agent，并保持启动、发现、调用、失败、关闭都可控。

### 聚合根：McpServerConnection

身份：

- server name。

核心状态：

- configured server。
- transport。
- startup status。
- tools。
- resources。
- auth status。
- metadata。
- elicitation state。

能力：

- start。
- list tools。
- list resources。
- call tool。
- resolve elicitation。
- shutdown。

业务规则：

- disabled server 不应启动。
- 启动失败只影响该 server 的能力，不必让整个 thread 失败。
- 工具列表需要和 server name 绑定，避免工具名冲突。
- elicitation 必须回到用户或 reviewer 决策，不能假装成功。

### 实体：ExternalTool

代表 MCP、app connector 或 dynamic tool 暴露给模型的外部能力。

核心状态：

- tool name。
- provider/server。
- schema。
- visibility。
- defer loading。
- parallel support。

能力：

- expose to model。
- call。
- return content items。

### 领域服务：ExternalCapabilityManager

逻辑服务，综合 MCP、apps、plugins、dynamic tools。

能力：

- 聚合可访问工具。
- 根据 mention 和配置过滤。
- 处理 tool search。
- 缓存或刷新工具列表。

### 领域事件

- McpStartupStarted。
- McpStartupReady。
- McpStartupFailed。
- McpToolListed。
- McpToolCallStarted。
- McpToolCallCompleted。
- McpElicitationRequested。

### 闭环流程

```text
加载 thread/session
-> 根据配置启动 MCP
-> 收集工具和资源
-> 本轮根据输入和配置筛选可见工具
-> 模型调用外部工具
-> 通过 MCP/app/dynamic tool 执行
-> 结果回灌模型
-> session 结束时关闭连接
```

## 9. 知识注入闭环

### 业务目标

让 agent 在当前项目和任务类型下遵守正确工作方式。

### 聚合根：InstructionBundle

InstructionBundle 是本轮模型行为约束的集合。

来源：

- base instructions。
- developer instructions。
- AGENTS.md。
- skills。
- plugin instructions。
- permissions instructions。
- collaboration mode instructions。

能力：

- load。
- merge。
- resolve priority。
- render for model。
- track injection side effects。

业务规则：

- 不同来源优先级不能混乱。
- 项目本地规则必须按 cwd 作用域加载。
- skill 只有被显式或隐式触发时才应完整注入。
- 缺少 skill 依赖时可以请求用户输入。

### 实体：ProjectInstruction

代表 AGENTS.md 等项目规则。

核心状态：

- path。
- scope。
- content。
- max bytes/truncation。

业务规则：

- 只能影响其作用域内的工作。
- 超大内容要截断。

### 实体：Skill

代表一种可复用任务流程知识。

核心状态：

- name。
- description。
- path。
- scope。
- enabled/disabled。
- dependencies。
- plugin provenance。

能力：

- match explicit mention。
- detect implicit invocation。
- render instructions。
- request dependencies。

### 实体：PluginContribution

代表插件提供的能力。

类型：

- skill roots。
- apps/connectors。
- marketplace metadata。

### 领域事件

- InstructionLoaded。
- SkillDiscovered。
- SkillInjected。
- SkillDependencyRequested。
- PluginContributionLoaded。

### 闭环流程

```text
创建 session/turn
-> 读取项目和用户配置
-> 加载 AGENTS.md、skills、plugins
-> 根据用户输入决定注入哪些知识
-> 合并成 InstructionBundle
-> 进入 PromptContext
-> 影响模型行为
```

## 10. 持久化与恢复闭环

### 业务目标

保存 agent 工作过程，并允许以后继续、审计、回滚或分支。

### 聚合根：StoredThread

StoredThread 是持久化视角下的 thread。

身份：

- thread id。

核心状态：

- metadata。
- turns。
- items。
- rollout path。
- archive status。
- thread name。

能力：

- create。
- append items。
- read。
- list。
- resume。
- archive。
- update metadata。

业务规则：

- 持久化记录必须能重建关键业务状态。
- 读历史和继续运行是不同操作。
- list 可以只返回摘要，不必加载完整 item。

### 实体：Rollout

Rollout 是事件/历史流水账。

能力：

- append。
- flush。
- reconstruct。
- truncate/rollback。

业务规则：

- 工具调用和模型输出都应作为事实记录。
- 半完成 turn 应有明确中断标记。

### 实体：StoredTurn

代表持久化后的 turn。

状态：

- status。
- items view。
- error。
- timing。

### 领域服务：ThreadStore

抽象持久化后端。

能力：

- 本地文件存储。
- 内存存储。
- 未来可替换后端。

业务规则：

- 应用层只依赖 thread id，不依赖具体存储路径。
- backend 差异不应污染核心业务语义。

### 领域事件

- ThreadPersisted。
- ThreadRead。
- ThreadHistoryLoaded。
- RolloutFlushed。
- ThreadRolledBack。

### 闭环流程

```text
turn 产生 item/event
-> 追加到运行时历史
-> 写入 rollout/thread-store
-> flush 或 shutdown
-> 后续 resume/read/list/fork 从存储重建
```

## 11. 多 Agent 闭环

### 业务目标

让一个主 agent 可以把工作拆给多个受控子 agent，并收敛结果。

### 聚合根：AgentTree

AgentTree 是一个 root thread 下的 agent 关系树。

核心状态：

- root thread。
- sub-agent registry。
- parent-child edges。
- depth。
- agent status。
- mailbox。

能力：

- spawn agent。
- list agents。
- send message。
- wait agent。
- close agent。
- update status。

业务规则：

- sub-agent 必须属于某个 root agent tree。
- 最大深度和最大数量必须受限。
- 子 agent 是独立 thread，不是普通工具调用。
- 子 agent 失败不应自动摧毁 root thread。
- 父子通信必须可追踪。

### 实体：Agent

身份：

- agent thread id。
- agent path。

核心状态：

- role。
- nickname。
- status。
- last task message。
- parent thread。

能力：

- receive initial task。
- run independently。
- report status。
- receive inter-agent message。

### 实体：MailboxMessage

代表 agent 间通信。

核心状态：

- sender。
- target。
- content。
- trigger turn。

### 领域服务：AgentControl

管理 agent tree 的控制面。

能力：

- 分配 spawn slot。
- 准备子 agent session source。
- 继承必要 shell snapshot/exec policy。
- 注册和更新 agent status。
- 处理 agent 通信。

### 领域事件

- AgentSpawned。
- AgentStatusChanged。
- AgentMessageSent。
- AgentCompleted。
- AgentClosed。

### 闭环流程

```text
主 agent 请求委派
-> AgentControl 校验深度和数量
-> 创建子 Thread/Session
-> 提交初始任务
-> 子 agent 独立运行
-> 状态和结果通过 registry/mailbox 返回
-> 主 agent 汇总或继续调度
```

## 12. 目标与记忆闭环

### 业务目标

让 thread 可以有跨 turn 的长期目标和可控记忆。

### 聚合根：Goal

Goal 表示当前 thread 正在追的长期目标。

核心状态：

- objective。
- status。
- token budget。
- usage。
- previous status。

能力：

- set。
- update。
- clear。
- pause/resume。
- apply runtime effects。
- maybe continue if idle。

业务规则：

- 同一 thread 通常只有一个当前 goal。
- 替换 objective 会重置使用统计。
- goal 完成必须基于实际目标达成，不能只因预算耗尽。

### 实体：MemoryState

代表 thread 是否允许参与长期记忆。

状态：

- enabled。
- disabled。
- polluted by external context。

业务规则：

- 外部搜索或外部工具上下文可能污染记忆。
- memory mode 是 thread 级元数据，不是单个 turn 的临时状态。

### 领域事件

- GoalSet。
- GoalUpdated。
- GoalCleared。
- GoalCompleted。
- MemoryModeChanged。
- MemoryPolluted。

### 闭环流程

```text
用户或系统设置目标
-> 目标进入 thread 元数据和 session runtime
-> 每个 turn 更新进度和 usage
-> 空闲时可能继续推进
-> 完成/暂停/清除后发出事件
-> memory 根据 thread 策略参与后续上下文
```

## 13. 事件与观测闭环

### 业务目标

把核心层发生的领域事实可靠传递给 UI、客户端、日志、trace 和持久化系统。

### 聚合根：EventStream

EventStream 是逻辑聚合，代表一个 session 对外输出的领域事实流。

核心状态：

- event id。
- event msg。
- ordering。
- target turn/thread。
- subscribers。

能力：

- send raw event。
- map event to app-server protocol。
- persist event。
- notify UI。
- record telemetry。

业务规则：

- 事件应该表达已经发生或正在请求的事实。
- 长任务要发进度事件，不能只等最终结果。
- 审批请求是事件也是业务等待点。
- event id 需要能关联到对应 operation/turn。

### 实体：TurnItemEvent

代表 item lifecycle。

类型：

- item started。
- item delta。
- item completed。

业务规则：

- started/completed 要成对或有明确失败语义。
- delta 需要能关联 item id。

### 实体：TelemetryRecord

代表用于分析和调试的运行记录。

内容：

- token usage。
- stream retry。
- model/provider。
- tool success/failure。
- latency。
- trace context。

### 领域事件

几乎所有核心输出都属于领域事件。重要的是区分：

- 给用户看的事件。
- 给客户端状态机看的事件。
- 给持久化恢复用的事件。
- 给观测系统用的 telemetry。

### 闭环流程

```text
核心层发生领域事实
-> 构造 Event
-> 发送给上层界面或 app-server
-> 必要时写入 rollout/thread-store
-> 同步记录 telemetry/trace
-> 上层 UI 或客户端更新状态
```

## 14. 各闭环之间的依赖关系

可以把核心 agent 层理解成一个由多个闭环嵌套组成的系统：

```text
会话闭环
  -> Turn 执行闭环
      -> 上下文构造闭环
      -> 模型采样闭环
      -> 工具调用闭环
          -> 权限与审批闭环
          -> Shell 执行闭环
          -> MCP/外部能力闭环
      -> 持久化与事件闭环
  -> 多 Agent 闭环
  -> 目标与记忆闭环
```

关键依赖：

- Turn 执行依赖上下文构造和工具调用。
- 工具调用依赖权限与审批。
- MCP/外部能力为工具调用提供工具来源。
- Skills/AGENTS.md/Plugins 为上下文构造提供指令来源。
- 持久化与事件贯穿所有闭环。
- 多 agent 复用会话、turn、工具、权限、持久化这些基础闭环。
- Goal 和 memory 跨越多个 turn，影响后续上下文。

## 15. DDD 视角下的设计判断

### 哪些是聚合根

建议把这些看成聚合根：

- Thread：长期会话聚合。
- Turn：单次工作聚合。
- ToolCall：一次工具副作用聚合。
- PermissionDecision：一次权限裁决聚合。
- McpServerConnection：一个外部能力连接聚合。
- AgentTree：一个多 agent 关系聚合。
- StoredThread：持久化视角聚合。
- Goal：长期目标聚合。

### 哪些是实体

建议把这些看成实体：

- Session。
- ActiveTurn。
- TurnContext。
- ConversationHistory。
- ToolRouter。
- ToolRegistry。
- CommandExecution。
- ApprovalRequest。
- Skill。
- ProjectInstruction。
- PluginContribution。
- Agent。
- MailboxMessage。
- MemoryState。
- EventStream。

### 哪些是值对象

建议把这些看成值对象：

- ThreadId、TurnId、CallId。
- PermissionProfile。
- ApprovalPolicy。
- SandboxPolicy。
- NetworkPolicy。
- ModelInfo。
- ToolSpec。
- ToolPayload。
- ToolResult payload。
- TokenUsage。
- TraceContext。
- CollaborationMode。
- Personality。
- EnvironmentSelection。

这些对象本身更多是描述状态或配置，业务身份通常来自它们所属的聚合。

### 哪些是领域服务

建议把这些看成领域服务：

- ThreadManager：创建、恢复、管理 thread。
- TurnRunner：运行 turn。
- ContextManager：组织模型上下文。
- ToolRuntime：执行工具调用。
- PermissionEvaluator：评估权限和审批。
- SandboxManager：把权限转成平台执行环境。
- ExternalCapabilityManager：聚合 MCP/apps/dynamic tools。
- InstructionResolver：加载并合并指令。
- AgentControl：管理子 agent。
- ThreadStore：持久化抽象。
- EventPublisher：发出领域事件。

## 16. 挖源码时的阅读顺序

按 DDD 闭环继续深入时，建议不要按文件名顺序读，而是按领域问题读：

1. **Thread 如何诞生和恢复**：看 ThreadManager、CodexThread、thread-store。
2. **Turn 如何跑起来**：看 Session、submission loop、TurnContext。
3. **模型上下文如何构造**：看 context manager、skills、AGENTS.md、tool spec 构造。
4. **模型输出如何驱动工具**：看 stream event handling、ToolRouter、ToolRuntime。
5. **工具如何被安全执行**：看 approval、permission profile、exec、sandboxing。
6. **外部工具如何接入**：看 MCP connection manager、apps/connectors、dynamic tools。
7. **过程如何保存和恢复**：看 rollout、thread-store、reconstruction。
8. **多 agent 如何复用基础闭环**：看 AgentControl、registry、mailbox。
9. **目标和记忆如何跨 turn 生效**：看 goals、memory mode、context injection。

## 17. 一句话总结

从 DDD 角度看，核心 agent 层的本质是：

```text
以 Thread 为长期聚合，
以 Turn 为执行聚合，
以 ToolCall 和 PermissionDecision 控制副作用，
以 Context 和 Instruction 控制模型认知，
以 Event 和 Store 保证过程可见、可恢复，
再用 AgentTree、Goal、Memory 扩展成长期、多主体工作系统。
```

理解这些实体和闭环之后，再深入源码时就不容易陷入“函数调用细节”，而能持续追问：这个对象属于哪个领域？它守护什么业务不变量？它发出的事件意味着哪个业务事实？它和哪个闭环交接？
