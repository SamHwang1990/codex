# 首次发给模型的 Prompt 结构

这份笔记解释“首次发给模型的 prompt 里包含什么”。这里的“首次”指一个 regular user turn 的第一次模型采样；如果这是新 thread 的第一个真实用户 turn，则还会注入完整初始上下文。后续采样会额外包含工具调用和工具结果，不是本文重点。

本文尽量用通用语言表达，不依赖 Codex 类名理解。

## 1. 总体结论

首次采样发给模型的不是一段纯文本，而是一个结构化请求：

```ts
interface FirstSamplingModelRequest {
  model: string;
  instructions: string;
  input: PromptInputItem[];
  tools: ModelVisibleTool[];
  toolChoice: "auto";
  parallelToolCalls: boolean;
  reasoning?: ReasoningRequest;
  output?: OutputFormatRequest;
  stream: true;
  serviceTier?: string;
  promptCacheKey?: string;
  metadata?: Record<string, string>;
}
```

可以把它拆成四层：

| 层 | 数据 | 来源 | 作用 |
| --- | --- | --- | --- |
| 指令层 | `instructions` | session 的 base instructions | 定义 agent 的基础行为、工作规范、通用规则 |
| 输入层 | `input` | session history + 本轮上下文 + 用户输入 + hook/skill/plugin 注入 | 给模型看的对话历史和当前任务上下文 |
| 工具层 | `tools` / `toolChoice` / `parallelToolCalls` | 本轮工具注册表、MCP、dynamic tools、apps/connectors、配置开关 | 告诉模型可调用哪些动作 |
| 采样配置层 | `model` / `reasoning` / `output` / `stream` / `serviceTier` | turn context、model info、provider/config | 控制模型、推理强度、输出格式、流式返回等 |

关键结论：

> prompt 不是 thread JSONL 的原样内容；它是在每次采样前，从 session 内存历史、当前 turn context、工具注册表和配置重新计算出来的结构化请求。

## 2. 首次采样前的处理流程

```text
收到用户输入
  -> 创建 turn context
  -> 可能执行 pre-sampling compact
  -> 写入本轮上下文 baseline 或上下文 diff
  -> 解析用户输入里的 skill/plugin/app mention
  -> 执行 session-start / user-prompt-submit hooks
  -> 把当前用户输入写入 session history
  -> 注入 hook 附加上下文、skill 内容、plugin 内容
  -> 从 session history 复制一份 prompt history
  -> 过滤、截断、归一化、剥离不支持的图片
  -> 构建当轮可见工具列表
  -> 组装 Prompt
  -> 转成 Responses API request
```

如果 hook 决定停止，本轮可能不会发出模型请求。

## 3. 顶层请求结构

```ts
interface FirstSamplingModelRequest {
  /**
   * 本轮使用的模型 slug。
   * 来源：turn context 中解析后的 model_info。
   */
  model: string;

  /**
   * 基础指令，不在 input 数组里。
   * 来源：session configuration 的 base instructions。
   */
  instructions: string;

  /**
   * 发给模型的历史和当前输入。
   * 来源：session history 经 for_prompt 处理后的结果。
   */
  input: PromptInputItem[];

  /**
   * 当前模型可见工具。
   * 来源：工具配置、MCP、dynamic tools、apps/connectors、skill/plugin 触发结果。
   */
  tools: ModelVisibleTool[];

  /**
   * 当前实现使用 auto，让模型自主决定是否调用工具。
   */
  toolChoice: "auto";

  /**
   * 是否允许模型一次返回多个工具调用。
   * 来源：模型能力。
   */
  parallelToolCalls: boolean;

  /**
   * 推理配置。只有模型/provider 支持时才有意义。
   */
  reasoning?: ReasoningRequest;

  /**
   * 最终输出格式约束，例如 JSON schema 或 verbosity。
   */
  output?: OutputFormatRequest;

  /**
   * 普通 turn 使用流式响应。
   */
  stream: true;

  /**
   * 服务层级，如 auto/default/flex 等，取决于配置和 provider。
   */
  serviceTier?: string;

  /**
   * prompt cache key 通常绑定 thread id，让同一 thread 的请求可复用缓存。
   */
  promptCacheKey?: string;

  /**
   * 客户端/安装级元数据，不是模型语义输入。
   */
  metadata?: Record<string, string>;
}
```

这个结构不是“业界所有模型统一遵守的标准入参”。它更准确地说是 Codex 的 prompt 抽象经过一次转换后，发给 OpenAI Responses API 的请求形状。

上面的 `FirstSamplingModelRequest` 用的是便于理解的 TypeScript/camelCase 表达。Codex 真正发给 OpenAI Responses API 的 wire 字段更接近下面这样：

```ts
interface OpenAIResponsesRequest {
  model: string;
  instructions?: string;
  input: ResponseItem[];
  tools: unknown[];
  tool_choice: "auto";
  parallel_tool_calls: boolean;
  reasoning?: ReasoningRequest;
  store: boolean;
  stream: true;
  include: string[];
  service_tier?: string;
  prompt_cache_key?: string;
  text?: OpenAITextControls;
  client_metadata?: Record<string, string>;
}
```

在一个通用 agent 里，建议拆成两层理解：

```ts
interface AgentPrompt {
  /**
   * agent 自己维护的通用 prompt 抽象。
   * 这层应该稳定，不直接绑定某个模型厂商。
   */
  instructions: string;
  input: PromptInputItem[];
  tools: ModelVisibleTool[];
  sampling: SamplingOptions;
}

interface ProviderRequest {
  /**
   * 具体模型厂商/API 需要的请求体。
   * 例如 OpenAI Responses API、Chat Completions、Anthropic Messages、内部模型协议。
   */
  body: unknown;
  headers?: Record<string, string>;
}

interface ProviderAdapter {
  toProviderRequest(prompt: AgentPrompt): ProviderRequest;
}
```

Codex 在这里做的是：

```text
session history / turn context / tools / config
  -> AgentPrompt
  -> OpenAI Responses API request
  -> SSE 或 WebSocket stream
```

如果换成别的模型，通常会再做 provider adapter，例如：

| Codex 抽象 | OpenAI Responses API | 其他模型常见映射 |
| --- | --- | --- |
| `instructions` | 顶层 `instructions` | 可能转成 system/developer message |
| `input` | `input: ResponseItem[]` | 可能转成 `messages: Message[]` |
| `tools` | Responses tools JSON | 可能转成 function/tool schema |
| `reasoning` | `reasoning` 字段 | 可能不支持，或映射成 vendor 私有字段 |
| `text.format` | JSON schema 输出约束 | 可能转成 response format、tool-only 输出，或 prompt 约束 |

## 4. `instructions`: 基础指令

```ts
interface BaseInstructions {
  text: string;
  source:
    | "model_default"
    | "session_override"
    | "rollout_session_meta"
    | "agent_role_override";
}
```

含义：

- 这是模型请求里的顶层 `instructions` 字段。
- 它不是 `input` 里的 message。
- 它定义 agent 的基础行为，例如如何执行任务、如何使用工具、如何提交结果。

来源：

| 场景 | 来源 |
| --- | --- |
| 新 thread | 当前模型的 base instructions，可能受 personality/template 影响 |
| 配置覆盖 | config/session configuration 的 `base_instructions` |
| resume | 可能来自 rollout/session meta 中保存的 base instructions |
| sub-agent / agent role | 可能被 agent role 配置覆盖 |

处理逻辑：

1. session 初始化时解析出本 session 的 base instructions。
2. turn 采样时从 session state 读取。
3. 转请求时写入顶层 `instructions`。
4. 不追加到 `input`，所以不会像普通历史一样被 rollback/compact 处理。

## 5. `input`: 模型可见历史和当前任务

```ts
type PromptInputItem =
  | MessageInputItem
  | ReasoningInputItem
  | FunctionCallInputItem
  | FunctionCallOutputInputItem
  | CustomToolCallInputItem
  | CustomToolCallOutputInputItem
  | LocalShellCallInputItem
  | WebSearchCallInputItem
  | ImageGenerationCallInputItem
  | ToolSearchInputItem;

interface MessageInputItem {
  type: "message";
  role: "developer" | "user" | "assistant" | string;
  content: ContentItem[];
  phase?: "commentary" | "final_answer";
}

type ContentItem =
  | { type: "input_text"; text: string }
  | { type: "output_text"; text: string }
  | { type: "input_image"; image_url: string; detail?: "auto" | "low" | "high" | "original" };
```

首次采样时，真正发给模型的 `input` 是一个数组：

```ts
type PromptInput = PromptInputItem[];
```

它不是一个带 `priorHistory`、`currentUserMessage` 字段的对象。更准确地说，`input` 是由几段逻辑内容按顺序拼接出来的：

```ts
const input: PromptInputItem[] = [
  ...priorHistoryItems,
  ...initialOrDiffContextItems,
  currentUserMessageItem,
  ...hookAdditionalContextItems,
  ...skillContextItems,
  ...pluginContextItems,
];
```

这些 `xxxItems` 只是解释构造过程时使用的逻辑分组，不是模型请求里的字段。模型最终只看到一个扁平的 `PromptInputItem[]`。

对新 thread 的第一个真实用户 turn 来说，`priorHistory` 通常为空；对 resume/fork 的第一个 turn 来说，`priorHistory` 是从 JSONL 重建后的有效历史。

## 6. `priorHistory`: 既有历史

```ts
interface PriorHistory {
  items: PromptInputItem[];
  source: "empty_new_thread" | "reconstructed_from_thread_jsonl" | "existing_session_memory";
  processing: {
    compactCheckpointApplied: boolean;
    rollbackApplied: boolean;
    nonApiItemsDropped: boolean;
    toolOutputsTruncated: boolean;
    normalizedBeforeSend: boolean;
    unsupportedImagesRemoved: boolean;
  };
}
```

来源：

- 新 thread：没有 prior history。
- resume：从 thread JSONL 重建 session history。
- fork：从父 thread 的截断快照重建。
- 同一 session 后续 turn：来自内存 session history。

处理逻辑：

1. JSONL 不是原样发送，会先 replay 成 session history。
2. compact checkpoint 可替换旧历史。
3. rollback event 会删除最后 N 个用户 turn。
4. system message、unknown item 等不能作为 API message 的内容会被过滤。
5. 工具输出会按 truncation policy 截断。
6. 发送前确保 tool call 和 tool output 配对，删除孤儿 output。
7. 如果模型不支持图片，会移除图片内容。

## 7. `initialOrDiffContext`: 本轮上下文注入

首次发给模型前，agent 会把“当前运行环境和规则”写成模型可见 message。新 thread 第一个 turn 通常是完整注入；已有 baseline 的后续 turn 通常只注入 diff。

```ts
interface ContextInjection {
  mode: "full_initial_context" | "settings_diff";
  developerMessages: DeveloperContextMessage[];
  contextualUserMessages: ContextualUserMessage[];
  persistedTurnContextSnapshot: TurnContextSnapshot;
}

interface DeveloperContextMessage extends MessageInputItem {
  role: "developer";
  content: Array<{ type: "input_text"; text: string }>;
}

interface ContextualUserMessage extends MessageInputItem {
  role: "user";
  content: Array<{ type: "input_text"; text: string }>;
}
```

它的来源不是 thread JSONL 里的某一条现成 prompt，而是当前 `TurnContext` 和 session 状态实时渲染出来的。

```ts
interface TurnContextSnapshot {
  model: string;
  cwd: string;
  environments: Array<{ id?: string; cwd: string; shell?: string }>;
  approvalPolicy: string;
  permissionProfile: string;
  collaborationMode: string;
  realtimeActive?: boolean;
  personality?: string;
  currentDate?: string;
  timezone?: string;
  network?: {
    allowedDomains: string[];
    deniedDomains: string[];
  };
}

interface ContextBaselineState {
  /**
   * session 内存里的“上一次上下文快照”。
   * 用来判断本轮是否需要 full initial context，还是只需要 settings diff。
   */
  referenceContextItem?: TurnContextSnapshot;

  /**
   * 持久化到 thread JSONL 的最新 turn_context。
   * resume 后可以恢复 referenceContextItem。
   */
  persistedTurnContextItems: TurnContextSnapshot[];
}
```

完整流程：

```text
本轮 TurnContext 创建完成
  -> 读取 session.referenceContextItem
  -> 如果没有 baseline：build_initial_context(turnContext)
  -> 如果已有 baseline：build_settings_update_items(previous, current)
  -> 把生成的 context message 写入 session history
  -> 无论有没有 diff，都把 current TurnContext 持久化为 turn_context
  -> session.referenceContextItem = current TurnContext
```

### 7.1 Developer 上下文

```ts
interface DeveloperContextSections {
  modelSwitch?: string;
  permissions?: PermissionInstructions;
  developerInstructions?: string;
  memoryToolInstructions?: string;
  collaborationModeInstructions?: string;
  realtimeInstructions?: string;
  personalityInstructions?: string;
  appsInstructions?: string;
  availableSkillsInstructions?: string;
  availablePluginsInstructions?: string;
  commitAttributionInstructions?: string;
  multiAgentUsageHint?: string;
  guardianPolicy?: string;
}
```

含义：

| 数据 | 来源 | 处理逻辑 |
| --- | --- | --- |
| `modelSwitch` | 上一 turn 模型与当前模型不同 | 如果模型变了，补充新模型行为说明；新 thread 通常没有 |
| `permissions` | permission profile、approval policy、exec policy、cwd | 渲染成开发者指令，说明能做什么、何时请求审批 |
| `developerInstructions` | config / agent role / session overrides | 非空时加入；guardian 子任务会拆成单独 developer message |
| `memoryToolInstructions` | memory feature + memory config | 启用 memory tool 时加入 |
| `collaborationModeInstructions` | collaboration mode | 有非空说明时加入 |
| `realtimeInstructions` | realtime 状态 | realtime 启停时加入 |
| `personalityInstructions` | personality feature + model messages | 如果 personality 没有 baked in base instructions，则单独加入 |
| `appsInstructions` | enabled apps/connectors | 说明可用 app/connectors 能力 |
| `availableSkillsInstructions` | 当前可用 skills | 在 token budget 内渲染 skill 摘要，可能伴随 warning |
| `availablePluginsInstructions` | 已加载 plugins | 渲染 plugin capability summary |
| `commitAttributionInstructions` | git commit attribution config | 启用 commit 功能时加入 |
| `multiAgentUsageHint` | multi-agent feature/source | 单独追加多 agent 使用提示 |
| `guardianPolicy` | guardian reviewer source | 作为单独 developer message，便于审计 |

处理逻辑：

1. 收集多个 developer section。
2. 空 section 丢弃。
3. 大多数 section 聚合成一个 `role="developer"` message。
4. multi-agent usage hint 可能单独成为一个 developer message。
5. guardian policy 在 guardian 场景下单独成为一个 developer message。

Developer 上下文的语义：

- 它是“应用/开发者给模型的运行规则”，优先级高于普通 user message。
- 它不是用户当前任务，而是 agent 框架对模型的约束、能力说明、权限说明和运行策略。
- 模型并不会通过某个隐藏 API “理解 availableSkillsInstructions 是 skills 列表”；它看到的是一个 `role="developer"` 的 message，内容用 XML/Markdown 风格标签组织，模型按 developer 指令优先级解释。
- 如果你的 Web Word agent 没有 developer role，可以把这类内容映射到 system message，或放在比用户任务更高优先级的内部指令通道。

`availableSkillsInstructions` 和后续 Skill 注入不是一回事：

```ts
interface AvailableSkillsInstructionsBlock {
  role: "developer";
  purpose: "告诉模型有哪些 skill 可用，以及如何触发/加载 skill";
  contains: Array<{
    name: string;
    description?: string;
    path: string;
  }>;
}

interface InjectedSkillContextBlock {
  role: "user";
  purpose: "用户本轮已经显式选中或提到某个 skill，把该 skill 的完整 SKILL.md 内容注入给模型";
  contains: {
    name: string;
    path: string;
    contents: string;
  };
}
```

区别：

| 项 | `availableSkillsInstructions` | 后续 Skill 注入 |
| --- | --- | --- |
| role | developer | user |
| 时机 | 初始上下文或 full reinjection | 当前 turn 解析到 skill mention 后 |
| 内容粒度 | skill 名称、描述、路径、使用规则 | 具体某个 skill 的完整内容 |
| 作用 | 让模型知道“有哪些能力可选” | 让模型真正获得“这个能力怎么用”的详细说明 |
| token 成本 | 受 metadata budget 限制，尽量轻 | 只注入被触发的 skill，内容更重 |

### 7.2 initial 的来源和组装

```ts
interface InitialContextAssembly {
  source: {
    turnContext: TurnContextSnapshot;
    sessionConfiguration: {
      baseInstructions: string;
      collaborationMode: string;
      sessionSource: string;
    };
    runtimeServices: {
      execPolicy: string;
      shell: string;
      mcpConnectors: unknown[];
      skills: unknown[];
      plugins: unknown[];
      subagents?: string;
    };
    projectInstructions?: string;
  };
  output: Array<DeveloperContextMessage | ContextualUserMessage>;
}
```

组装算法：

```text
build_initial_context(turnContext)
  -> 初始化 developerSections = []
  -> 初始化 contextualUserSections = []
  -> 如果模型相对上一 turn 变化：加入 modelSwitch
  -> 如果启用权限说明：加入 permissions
  -> 如果有 developer instructions：加入 developerInstructions
  -> 如果启用 memory：加入 memoryToolInstructions
  -> 如果有 collaboration mode 指令：加入 collaborationModeInstructions
  -> 如果 realtime 需要说明：加入 realtimeInstructions
  -> 如果 personality 没有 baked in base instructions：加入 personalityInstructions
  -> 如果启用 apps/connectors：查询 MCP/app connector，加入 appsInstructions
  -> 如果启用 skills instructions：渲染可用 skill 摘要，加入 availableSkillsInstructions
  -> 查询已加载 plugins：加入 availablePluginsInstructions
  -> 如果启用 commit attribution：加入 commitAttributionInstructions
  -> 如果有 AGENTS.md/用户项目指令：加入 contextual user sections
  -> 如果启用 environment context：加入 cwd/shell/date/timezone/network/subagents
  -> developerSections 聚合成 developer message
  -> contextualUserSections 聚合成 user message
  -> 特殊场景下 multi-agent hint / guardian policy 拆成独立 developer message
```

注意：`instructions` 顶层字段不在这里组装。`build_initial_context` 只负责 `input` 数组里的上下文 message。

### 7.3 diff 的维护和组装

```ts
interface ContextDiffAssembly {
  previous: TurnContextSnapshot;
  current: TurnContextSnapshot;
  emittedItems: Array<DeveloperContextMessage | ContextualUserMessage>;
  persistedCurrentSnapshot: TurnContextSnapshot;
}
```

如果在 turn 运行期间多次修改设置，要区分两类：

1. **已经创建的当前 turn**：它使用创建时解析出来的 `TurnContext`。普通配置刷新不会随意改写正在采样的 prompt。
2. **后续 turn**：新的用户输入到来时，会用最新配置创建新的 `TurnContext`，再和 session 里的 `referenceContextItem` 做 diff。

session 内部维护的是“最新 baseline”，不是一串待发送的设置修改事件：

```text
turn A 开始
  -> current = TurnContext(A)
  -> previous = session.referenceContextItem
  -> 发送 full 或 diff
  -> 持久化 turn_context(A)
  -> session.referenceContextItem = A

turn A 运行中配置被修改多次
  -> 已经发出去的 prompt 不回滚
  -> 已运行的工具/采样按 turn A 的上下文继续

turn B 开始
  -> current = TurnContext(B)，读取最新配置
  -> previous = session.referenceContextItem，也就是 A
  -> 只渲染 A -> B 的最终差异
  -> 持久化 turn_context(B)
  -> session.referenceContextItem = B
```

diff 当前覆盖的主要字段：

| diff 类型 | 比较逻辑 | 生成的 message |
| --- | --- | --- |
| environment | cwd/date/timezone/network/subagents 变化；shell 比较时特殊处理 | contextual user message |
| permissions | permission profile 或 approval policy 变化 | developer message |
| collaboration mode | collaboration mode 变化且新模式有说明 | developer message |
| realtime | realtime active 状态变化 | developer message |
| personality | 同模型下 personality 变化 | developer message |
| model instructions | 上一 turn 模型和当前模型不同 | developer message |

源码里也明确留了 TODO：diff 还不是 full initial context 的 100% 完整可逆差分。有些内容仍依赖持久化 baseline、resume/replay 和 full reinjection 来保证恢复。

### 7.4 Contextual user 上下文

```ts
interface ContextualUserSections {
  userInstructions?: ProjectInstructions;
  environmentContext?: EnvironmentContext;
}

interface ProjectInstructions {
  directory: string;
  text: string;
  source: "AGENTS.md" | "global_user_instructions" | "project_doc";
}

interface EnvironmentContext {
  environments:
    | { type: "single"; cwd: string; shell: string }
    | { type: "multiple"; items: Array<{ id: string; cwd: string; shell: string }> }
    | { type: "none" };
  currentDate?: string;
  timezone?: string;
  network?: {
    enabled: true;
    allowedDomains: string[];
    deniedDomains: string[];
  };
  subagents?: string;
}
```

含义：

- `userInstructions`：项目/用户指令，例如 AGENTS.md。它用 user role 注入，但属于上下文，不是当前用户任务。
- `environmentContext`：当前 cwd、shell、日期、时区、网络策略、subagent 状态等。

处理逻辑：

1. 如果配置关闭 environment context，则不注入。
2. 如果有多个 environment，会以多 environment 结构表达。
3. 日期和时区来自 turn context 创建时的本地时间上下文。
4. 网络域名来自 config requirements。
5. subagents 信息来自 agent control 的运行状态。

## 8. `currentUserMessage`: 当前用户输入

```ts
type UserInput =
  | { type: "text"; text: string; textElements?: TextElement[] }
  | { type: "image"; image_url: string }
  | { type: "local_image"; path: string }
  | { type: "skill"; name: string; path: string }
  | { type: "mention"; name: string; path: string };

interface CurrentUserMessage extends MessageInputItem {
  type: "message";
  role: "user";
  content: ContentItem[];
}

interface TextElement {
  range?: unknown;
  kind?: string;
  data?: unknown;
}
```

处理逻辑：

| 输入类型 | 进入模型 input 的方式 |
| --- | --- |
| text | 转成 `{ type: "input_text", text }` |
| image URL | 转成 `input_text("<image>") + input_image + input_text("</image>")` |
| local image | 读取本地文件，编码成模型可用 image；失败则转成错误占位文本 |
| skill mention | 不直接进入当前 user message，后续解析成 skill context |
| plugin/app mention | 不直接进入当前 user message，后续解析成 plugin/app context 或显式 connector selection |
| text elements | 保留给 UI/event；`ResponseItem::Message` 本身不携带这些 span |

当前用户输入会被写入 session history，再参与首次采样。

“skill mention 不直接进入当前 user message”的意思是：用户输入数组里可以有结构化的 skill 选择项，但把 `Vec<UserInput>` 转成模型可见 `CurrentUserMessage` 时，`skill` 和 `mention` 两类 item 会返回空内容。

```ts
const userInputs: UserInput[] = [
  { type: "text", text: "用这个规范检查当前实现" },
  { type: "skill", name: "code-review", path: "/skills/code-review/SKILL.md" },
];

const currentUserMessage: CurrentUserMessage = {
  type: "message",
  role: "user",
  content: [
    { type: "input_text", text: "用这个规范检查当前实现" }
  ]
};

const laterSkillContext: MessageInputItem = {
  type: "message",
  role: "user",
  content: [
    {
      type: "input_text",
      text: "<skill>\\n<name>code-review</name>\\n<path>/skills/code-review/SKILL.md</path>\\n...完整 SKILL.md 内容...\\n</skill>"
    }
  ]
};
```

这样做的好处是：当前用户任务保持干净；skill 内容作为独立上下文块注入，便于记录、审计、预算控制和去重。

`CurrentUserMessage.content` 的顺序有要求。它按用户输入数组顺序展开，图片会被包在文本标签之间：

```ts
const userInputs: UserInput[] = [
  { type: "text", text: "看这张图：" },
  { type: "image", image_url: "data:image/png;base64,..." },
  { type: "text", text: "然后总结问题" }
];

const content: ContentItem[] = [
  { type: "input_text", text: "看这张图：" },
  { type: "input_text", text: "<image>" },
  { type: "input_image", image_url: "data:image/png;base64,...", detail: "high" },
  { type: "input_text", text: "</image>" },
  { type: "input_text", text: "然后总结问题" }
];
```

数组里允许 `type` 重复，例如多个 `input_text`、多张 `input_image` 都可以。语义靠顺序表达。

“at 文件、代码段”在当前 Codex 这层没有独立的 `file` 或 `code_block` content type。常见表达方式是：

```ts
type RichUserInputStrategy =
  | {
      kind: "plain_text_expansion";
      meaning: "把 @file 或代码段直接渲染进 text，例如 Markdown fenced code block";
    }
  | {
      kind: "structured_mention";
      meaning: "用 UserInput.mention 保存结构化目标，例如 app://、plugin://、skill://，后续解析成独立上下文或工具选择";
    }
  | {
      kind: "text_element_metadata";
      meaning: "UI 层保留 span/placeholder，模型请求里通常只看到 text";
    };
```

映射到 Web Word，可以把 `@段落`、`@评论`、`@选区`设计成结构化 mention，但在发模型前最好解析成独立上下文块，而不是只把 `@xxx` 原样塞进用户文本。

## 9. Hook 附加上下文

```ts
interface HookAdditionalContext {
  source: "session_start_hook" | "user_prompt_submit_hook" | "pending_input_hook";
  disposition: "continue" | "stop" | "block_pending_input";
  additionalContextItems: MessageInputItem[];
}
```

含义：

- hooks 可以在模型采样前追加上下文。
- hooks 也可以停止本轮，不让 prompt 发给模型。

处理逻辑：

1. session-start hook 可能在 turn 初期运行。
2. user-prompt-submit hook 在当前用户输入记录前后参与判断。
3. 如果 hook 返回 stop，则记录必要上下文后直接结束，不采样。
4. 如果 hook 返回 additional context，则写入 session history，再进入 prompt input。

## 10. Skill / Plugin / App 注入

```ts
interface SkillContextInjection {
  mentionedSkills: SkillMention[];
  injectedItems: MessageInputItem[];
  warnings: string[];
}

interface SkillMention {
  name?: string;
  path?: string;
  source: "explicit_user_input" | "resolved_from_mention";
}

interface PluginContextInjection {
  mentionedPlugins: PluginMention[];
  injectedItems: MessageInputItem[];
  enabledConnectorIds: string[];
}

interface PluginMention {
  id?: string;
  name?: string;
  source: "explicit_user_input" | "resolved_from_mention";
}
```

来源：

- 用户输入里的 skill items。
- 用户输入里的 mention items。
- 当前 session 已加载 plugins。
- MCP/app connector inventory。

处理逻辑：

1. 先根据用户输入找显式 skill/plugin/app mention。
2. skill mention 会加载对应 skill 内容或摘要，并作为上下文 item 注入。
3. plugin mention 会转换成 plugin capability/context。
4. app connector mention 会影响本轮可见 connector 工具集合。
5. 如果 skill 有环境变量/MCP 依赖，可能在采样前请求安装或提示。

## 11. `tools`: 当前可见工具

```ts
type ModelVisibleTool =
  | FunctionTool
  | NamespaceTool
  | ToolSearchTool
  | LocalShellTool
  | ImageGenerationTool
  | WebSearchTool
  | FreeformTool;

interface FunctionTool {
  type: "function";
  name: string;
  description: string;
  strict: boolean;
  parameters: JsonSchema;
  deferLoading?: boolean;
}

interface NamespaceTool {
  type: "namespace";
  name: string;
  description: string;
  tools: FunctionTool[];
}

interface ToolSearchTool {
  type: "tool_search";
  execution: string;
  description: string;
  parameters: JsonSchema;
}

interface LocalShellTool {
  type: "local_shell";
}

interface ImageGenerationTool {
  type: "image_generation";
  outputFormat: string;
}

interface WebSearchTool {
  type: "web_search";
  externalWebAccess?: boolean;
  filters?: unknown;
  userLocation?: unknown;
  searchContextSize?: string;
  searchContentTypes?: string[];
}

interface FreeformTool {
  type: "custom";
  name: string;
  description: string;
  format: {
    type: string;
    syntax: string;
    definition: string;
  };
}

type JsonSchema = Record<string, unknown>;
```

来源：

| 工具来源 | 说明 |
| --- | --- |
| built-in tools | shell、apply_patch、view_image、plan、goal、multi-agent 等 |
| MCP tools | 外部 MCP server 暴露的工具 |
| dynamic tools | session/thread 启动时传入的动态工具 |
| app/connectors | 产品或插件暴露的 app 工具 |
| web/image tools | provider/model 能力和配置决定是否启用 |
| unavailable dummy tools | 对历史里曾出现但当前不可用的工具做兼容提示 |

处理逻辑：

1. 读取当前 turn 的 tools config。
2. 列出 MCP tools 和 connector tools。
3. 根据 apps 是否启用、显式 mention、connector 权限过滤可见工具。
4. deferred tools 不一定完整暴露，而是通过 tool search 或延迟加载机制给模型发现。
5. 生成模型可见 tool specs。
6. 如果模型支持 parallel tool calls，则开启 `parallelToolCalls`。

## 12. 推理和输出配置

```ts
interface ReasoningRequest {
  effort?: "minimal" | "low" | "medium" | "high" | string;
  summary?: "auto" | "concise" | "detailed" | string;
}

interface OutputFormatRequest {
  verbosity?: "low" | "medium" | "high" | string;
  format?: {
    type: "json_schema";
    name: string;
    strict: boolean;
    schema: JsonSchema;
  };
}
```

来源：

- reasoning effort：turn context / config / model default。
- reasoning summary：turn context / config。
- verbosity：模型能力 + config。
- JSON schema：turn context 的 final output schema。

处理逻辑：

1. 只有模型支持的字段才会写入请求。
2. 如果有 reasoning 配置，响应请求会要求 include encrypted reasoning content。
3. 如果有 final output JSON schema，会构造 text/output format。
4. guardian reviewer 这类特殊来源会关闭严格 schema 校验。

Codex 发给 OpenAI Responses API 时，对应的是 `text` 字段：

```ts
interface OpenAITextControls {
  verbosity?: "low" | "medium" | "high";
  format?: {
    type: "json_schema";
    name: "codex_output_schema";
    strict: boolean;
    schema: JsonSchema;
  };
}
```

例子 1：只控制输出详细程度。

```json
{
  "text": {
    "verbosity": "low"
  }
}
```

例子 2：要求最终答案符合 JSON schema。

```json
{
  "text": {
    "verbosity": "medium",
    "format": {
      "type": "json_schema",
      "name": "codex_output_schema",
      "strict": true,
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "summary": { "type": "string" },
          "actions": {
            "type": "array",
            "items": { "type": "string" }
          }
        },
        "required": ["summary", "actions"]
      }
    }
  }
}
```

对 Web Word 来说，一个常见输出 schema 可以是：

```ts
interface WebWordAgentOutput {
  summary: string;
  documentOperations: Array<
    | { type: "replace_range"; rangeId: string; text: string }
    | { type: "insert_comment"; anchorId: string; text: string }
    | { type: "suggestion"; title: string; rationale: string }
  >;
  needsUserConfirmation: boolean;
}
```

这类 schema 适合约束“最终答复结构”，但不应该替代工具调用。真正修改文档仍建议走工具，由工具校验 document revision、权限和 range id。

## 13. 模型会返回怎样的数据

模型返回不是一个单独字符串，而是一串流式事件。Codex 把 provider stream 解析成内部事件，再转成 UI/event/history 可用的 `ResponseItem`。

```ts
type ModelStreamEvent =
  | { type: "created" }
  | { type: "output_item_added"; item: ResponseItem }
  | { type: "output_item_done"; item: ResponseItem }
  | { type: "output_text_delta"; delta: string }
  | { type: "tool_call_input_delta"; itemId: string; callId?: string; delta: string }
  | { type: "reasoning_summary_delta"; delta: string; summaryIndex: number }
  | { type: "reasoning_content_delta"; delta: string; contentIndex: number }
  | { type: "reasoning_summary_part_added"; summaryIndex: number }
  | { type: "completed"; responseId: string; tokenUsage?: TokenUsage; endTurn?: boolean }
  | { type: "rate_limits"; snapshot: unknown }
  | { type: "server_model"; model: string };

type ResponseItem =
  | AssistantMessageItem
  | ReasoningItem
  | FunctionCallItem
  | CustomToolCallItem
  | LocalShellCallItem
  | WebSearchCallItem
  | ImageGenerationCallItem
  | ToolSearchCallItem
  | ToolOutputItem
  | { type: "other" };

interface AssistantMessageItem {
  type: "message";
  role: "assistant";
  content: Array<{ type: "output_text"; text: string }>;
  phase?: "commentary" | "final_answer";
}

interface ReasoningItem {
  type: "reasoning";
  summary: Array<{ type?: string; text?: string }>;
  content?: Array<{ type: "reasoning_text" | "text"; text: string }>;
  encrypted_content?: string;
}

interface FunctionCallItem {
  type: "function_call";
  name: string;
  namespace?: string;
  arguments: string; // JSON 字符串，不是已解析对象
  call_id: string;
}

interface CustomToolCallItem {
  type: "custom_tool_call";
  call_id: string;
  name: string;
  input: string;
  status?: string;
}

interface LocalShellCallItem {
  type: "local_shell_call";
  call_id?: string;
  status: string;
  action: unknown;
}

interface WebSearchCallItem {
  type: "web_search_call";
  status?: string;
  action?: unknown;
}

interface ImageGenerationCallItem {
  type: "image_generation_call";
  id: string;
  status: string;
  revised_prompt?: string;
  result: string;
}

interface ToolSearchCallItem {
  type: "tool_search_call";
  call_id?: string;
  status?: string;
  execution: string;
  arguments: unknown;
}

type ToolOutputItem =
  | { type: "function_call_output"; call_id: string; output: string | FunctionCallOutputContentItem[] }
  | { type: "custom_tool_call_output"; call_id: string; name?: string; output: string | FunctionCallOutputContentItem[] }
  | { type: "tool_search_output"; call_id?: string; status: string; execution: string; tools: unknown[] };

type FunctionCallOutputContentItem =
  | { type: "input_text"; text: string }
  | { type: "input_image"; image_url: string };

interface TokenUsage {
  inputTokens: number;
  cachedInputTokens: number;
  outputTokens: number;
  reasoningOutputTokens: number;
  totalTokens: number;
}
```

首轮采样返回后有两种大分支：

```text
模型返回 assistant message
  -> 记录 assistant message 到 session history
  -> 如果 completed/endTurn，不需要 follow-up
  -> turn 可以结束

模型返回 tool call
  -> 记录 tool call 到 session history
  -> agent 执行工具
  -> 把 tool output 写回 session history
  -> needs_follow_up = true
  -> 下一次采样把“历史 + tool output”再发给模型
```

流式 delta 和完成 item 的关系：

| 事件 | 用途 |
| --- | --- |
| `output_item_added` | 通知 UI 某个 assistant message / reasoning / tool call 开始 |
| `output_text_delta` | assistant 文本增量，用于边生成边展示 |
| `tool_call_input_delta` | 工具参数增量，用于展示工具调用参数逐步形成 |
| `reasoning_summary_delta` | reasoning 摘要增量 |
| `output_item_done` | 某个完整 item 完成；这时可记录进 history 或执行工具 |
| `completed` | 整个 response 完成；更新 token usage，判断是否还要 follow-up |

## 14. 首次采样的典型 input 示例

新 thread 的第一个用户 turn，抽象后大致是：

```ts
const input: PromptInputItem[] = [
  {
    type: "message",
    role: "developer",
    content: [
      { type: "input_text", text: "<permissions instructions>...</permissions instructions>" },
      { type: "input_text", text: "<collaboration_mode>...</collaboration_mode>" },
      { type: "input_text", text: "<available_skills>...</available_skills>" },
      { type: "input_text", text: "<available_plugins>...</available_plugins>" }
    ]
  },
  {
    type: "message",
    role: "user",
    content: [
      { type: "input_text", text: "# AGENTS.md instructions for /repo\n\n<INSTRUCTIONS>...</INSTRUCTIONS>" },
      { type: "input_text", text: "<environment_context>\n  <cwd>/repo</cwd>\n  <shell>zsh</shell>\n  <current_date>2026-05-09</current_date>\n  <timezone>Asia/Shanghai</timezone>\n</environment_context>" }
    ]
  },
  {
    type: "message",
    role: "user",
    content: [
      { type: "input_text", text: "用户当前问题或任务" }
    ]
  }
];
```

如果用户显式提到 skill/plugin/app，后面还可能追加对应上下文 item。若是 resume/fork，则这个数组前面会有从历史重建出的 prior history。

## 15. 发送前的最后处理

```ts
interface FinalPromptProcessing {
  historyNormalization: {
    ensureToolCallOutputsPresent: boolean;
    removeOrphanToolOutputs: boolean;
    stripUnsupportedImages: boolean;
  };
  outputTruncation: {
    toolOutputTruncatedOnRecord: boolean;
    policy: "bytes" | "tokens";
  };
  toolOutputFormatting: {
    reserializeShellOutputsWhenFreeformApplyPatchPresent: boolean;
  };
}
```

处理逻辑：

- `for_prompt` 会复制 session history，并做发送前归一化。
- 工具输出过长是在记录进 history 时处理，不等到最后一刻才裁剪。
- 如果当前工具列表含 freeform `apply_patch`，shell/apply_patch 输出会转成更适合模型理解的文本形式。
- 图片是否保留取决于模型 input modalities。

## 16. Web Word Agent 可借鉴的 Prompt 结构

如果映射到 Web Word，建议这样设计：

```ts
interface WebWordFirstSamplingRequest {
  model: string;
  instructions: string;
  input: Array<
    | WebWordSystemContextMessage
    | WebWordDocumentContextMessage
    | WebWordUserTaskMessage
    | WebWordToolHistoryItem
  >;
  tools: WebWordToolSpec[];
  output?: WebWordOutputContract;
  runtime: WebWordTurnRuntime;
}

interface WebWordDocumentContextMessage {
  type: "message";
  role: "user";
  content: Array<{ type: "input_text"; text: string }>;
  semanticType: "document_context";
  document: {
    id: string;
    revision: string;
    title?: string;
    selectedRange?: { start: number; end: number };
    surroundingText?: string;
    outline?: Array<{ id: string; level: number; title: string }>;
    comments?: Array<{ id: string; text: string; range?: { start: number; end: number } }>;
  };
}

interface WebWordUserTaskMessage {
  type: "message";
  role: "user";
  content: Array<{ type: "input_text"; text: string }>;
  semanticType: "current_user_task";
}

interface WebWordTurnRuntime {
  userId: string;
  workspaceId: string;
  documentId: string;
  documentRevision: string;
  locale?: string;
  timezone?: string;
  permissionProfile: {
    canRead: boolean;
    canComment: boolean;
    canEdit: boolean;
    canExport: boolean;
  };
}
```

设计建议：

1. 把基础 agent 行为放在顶层 `instructions`。
2. 把文档版本、选区、评论、outline 放进上下文 message。
3. 把当前用户任务单独作为 user message，不要和文档上下文混成一段。
4. 工具列表按权限和当前文档状态动态生成。
5. 每次采样前重新计算 prompt，不要直接复用持久日志。
6. document revision 必须进 turn runtime，防止工具结果应用到错误版本。

## 源码对照

| 主题 | 源码位置 |
| --- | --- |
| turn 首次采样前记录上下文、用户输入、skill/plugin 注入 | `codex-rs/core/src/session/turn.rs` |
| Prompt 顶层结构 | `codex-rs/core/src/client_common.rs` |
| Prompt 转 Responses API request | `codex-rs/core/src/client.rs` |
| Responses API request / text controls / stream event | `codex-rs/codex-api/src/common.rs` |
| Responses SSE completed usage 解析 | `codex-rs/codex-api/src/sse/responses.rs` |
| 初始上下文构造 | `codex-rs/core/src/session/mod.rs` |
| 上下文 diff 构造 | `codex-rs/core/src/context_manager/updates.rs` |
| 环境上下文格式 | `codex-rs/core/src/context/environment_context.rs` |
| 用户输入转 ResponseInputItem | `codex-rs/protocol/src/models.rs` |
| skill mention 解析和 skill 注入 | `codex-rs/core-skills/src/injection.rs`, `codex-rs/core/src/context/skill_instructions.rs` |
| 工具 spec 结构 | `codex-rs/tools/src/tool_spec.rs`, `codex-rs/tools/src/responses_api.rs` |
