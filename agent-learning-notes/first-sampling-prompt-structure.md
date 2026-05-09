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
  | { type: "input_image"; imageUrl: string; detail?: "auto" | "low" | "high" | "original" };
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

### 7.2 Contextual user 上下文

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
  | { type: "image"; imageUrl: string }
  | { type: "localImage"; path: string }
  | { type: "skill"; name?: string; path?: string }
  | { type: "mention"; target: string; label?: string };

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
  jsonSchema?: {
    name?: string;
    schema: JsonSchema;
    strict: boolean;
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

## 13. 首次采样的典型 input 示例

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

## 14. 发送前的最后处理

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

## 15. Web Word Agent 可借鉴的 Prompt 结构

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
| 初始上下文构造 | `codex-rs/core/src/session/mod.rs` |
| 上下文 diff 构造 | `codex-rs/core/src/context_manager/updates.rs` |
| 环境上下文格式 | `codex-rs/core/src/context/environment_context.rs` |
| 用户输入转 ResponseInputItem | `codex-rs/protocol/src/models.rs` |
| 工具 spec 结构 | `codex-rs/tools/src/tool_spec.rs`, `codex-rs/tools/src/responses_api.rs` |
