# Thread JSONL 历史数据结构

这份笔记只描述 thread JSONL 文件里会落盘的“历史记录”。这里的“历史”不是一个纯聊天数组，而是按时间追加的一组 rollout line：元数据、模型历史 item、上下文快照、compact 检查点和可恢复事件混在同一个 JSONL 文件里。

## 1. 总体形态

每一行都是一个独立 JSON 对象，带统一的写入时间 `timestamp`。业务上可以理解成：

```ts
type Json = null | boolean | number | string | Json[] | { [key: string]: Json };
type PathString = string;
type ThreadId = string;
type UnixSeconds = number;
type Milliseconds = number;
type DurationString = string;

type RolloutLine =
  | SessionMetaLine
  | ResponseItemLine
  | CompactedLine
  | TurnContextLine
  | EventMsgLine;

interface BaseLine {
  /**
   * 写入 JSONL 的 UTC 时间，不是事件自身发生时间。
   * 形如 2026-05-09T12:34:56.789Z。
   */
  timestamp: string;
}
```

顶层判别字段是 `type`，具体 payload 放在 `payload` 里：

```ts
type SessionMetaLine = BaseLine & {
  type: "session_meta";
  payload: SessionMetaPayload;
};

type ResponseItemLine = BaseLine & {
  type: "response_item";
  payload: ResponseItem;
};

type CompactedLine = BaseLine & {
  type: "compacted";
  payload: CompactedPayload;
};

type TurnContextLine = BaseLine & {
  type: "turn_context";
  payload: TurnContextPayload;
};

type EventMsgLine = BaseLine & {
  type: "event_msg";
  payload: StoredEventMsg;
};
```

## 2. 顶层场景索引

| JSONL 行类型 | 来源 | 含义 | 是否会进入模型历史 |
| --- | --- | --- | --- |
| `session_meta` | 新建、fork、resume 时初始化 thread recorder | thread 级元数据。回答“这个 thread 是谁、从哪里来、在哪个 cwd、使用什么基础指令”。 | 否 |
| `response_item` | session 把模型可见 item 记录到内存历史时同步落盘 | 真正的对话历史候选：用户消息、开发者上下文、助手消息、推理摘要、工具调用和工具结果。 | 是，但恢复和发送前还会过滤、截断、归一化 |
| `turn_context` | 每个真实 user turn 计算完上下文更新后落盘；compact 后也可能再落盘 | 该 turn 的运行配置快照。回答“当时 cwd、模型、权限、sandbox、指令、日期时区是什么”。 | 否，但用于恢复上下文 diff 基线 |
| `compacted` | 历史压缩完成时落盘 | compact 检查点。新版包含 `replacement_history`，可直接替换旧历史。 | 间接影响，会替换恢复出的历史 |
| `event_msg` | agent 对外发送事件后，经过持久化策略过滤落盘 | UI/恢复/审计事件，例如用户可见消息、turn started/complete、rollback、token usage、tool end。 | 大多否；只有 `thread_rolled_back` 会在恢复时改变历史 |

## 3. `session_meta`

```ts
type SessionMetaPayload = SessionMeta & {
  git?: GitInfo;
};

interface SessionMeta {
  id: ThreadId;
  forked_from_id?: ThreadId;
  timestamp: string;
  cwd: PathString;
  originator: string;
  cli_version: string;
  source: SessionSource;
  thread_source?: ThreadSource;
  agent_nickname?: string;
  agent_role?: string;
  agent_path?: string;
  model_provider: string | null;
  base_instructions: BaseInstructions | null;
  dynamic_tools?: DynamicToolSpec[];
  memory_mode?: string;
}

type SessionSource =
  | "cli"
  | "vscode"
  | "exec"
  | "mcp"
  | { custom: string }
  | { internal: InternalSessionSource }
  | { subagent: SubAgentSource }
  | "unknown";

type ThreadSource = "user" | "subagent" | "memory_consolidation";
type InternalSessionSource = "memory_consolidation";

type SubAgentSource =
  | "review"
  | "compact"
  | {
      thread_spawn: {
        parent_thread_id: ThreadId;
        depth: number;
        agent_path?: string;
        agent_nickname?: string;
        agent_role?: string;
      };
    }
  | { other: string };

interface BaseInstructions {
  text: string;
}

interface DynamicToolSpec {
  namespace?: string;
  name: string;
  description: string;
  inputSchema: Json;
  deferLoading: boolean;
}

interface GitInfo {
  commit_hash?: string;
  branch?: string;
  repository_url?: string;
}
```

字段含义：

| 字段 | 来源 | 含义 |
| --- | --- | --- |
| `id` | thread 创建时生成 | thread 的持久 ID，也是恢复、fork、列表索引用的 ID。 |
| `forked_from_id` | fork 场景 | 新 thread 来自哪个父 thread。 |
| `timestamp` | session meta 创建时 | session/thread 业务时间。不同于 JSONL 行外层写入时间。 |
| `cwd` | 启动配置 | 该 thread 的工作目录。 |
| `originator` | 客户端/入口 | 谁启动了这个 session。 |
| `cli_version` | 运行时版本 | 用于兼容性和排障。 |
| `source` | 启动入口 | CLI、VS Code、exec、MCP、内部任务或 sub-agent。 |
| `thread_source` | 分析分类 | 用户线程、subagent、memory consolidation。 |
| `agent_*` | sub-agent 创建参数 | 子 agent 的昵称、角色、路径。 |
| `model_provider` | 模型配置 | provider 名称，旧记录可能为空。 |
| `base_instructions` | session 初始化 | 基础系统指令。旧记录可能为空，恢复时可回退到模型管理器渲染值。 |
| `dynamic_tools` | 启动时工具配置 | 启动时可用的动态工具列表。 |
| `memory_mode` | 记忆配置 | thread 的记忆模式。 |
| `git` | recorder 创建时读取 | 当前仓库 commit、branch、remote URL。 |

## 4. `response_item`

`response_item` 是 JSONL 里最接近“聊天历史”的结构。但它仍不是原样发给模型：落盘前会过滤 `Other`，进入内存历史时会过滤 system message，并截断过长工具输出；发送模型前还会做 call/output 配对归一化和图片能力过滤。

```ts
type ResponseItem =
  | MessageItem
  | ReasoningItem
  | LocalShellCallItem
  | FunctionCallItem
  | ToolSearchCallItem
  | FunctionCallOutputItem
  | CustomToolCallItem
  | CustomToolCallOutputItem
  | ToolSearchOutputItem
  | WebSearchCallItem
  | ImageGenerationCallItem
  | CompactionResponseItem
  | ContextCompactionResponseItem
  | OtherResponseItem;

type ContentItem =
  | { type: "input_text"; text: string }
  | { type: "input_image"; image_url: string; detail?: ImageDetail }
  | { type: "output_text"; text: string };

type ImageDetail = "auto" | "low" | "high" | "original";
type MessagePhase = "commentary" | "final_answer";

interface MessageItem {
  type: "message";
  role: string; // 常见值: user | assistant | developer | system
  content: ContentItem[];
  phase?: MessagePhase;
}

interface ReasoningItem {
  type: "reasoning";
  summary: Array<{ type: "summary_text"; text: string }>;
  content?: Array<
    | { type: "reasoning_text"; text: string }
    | { type: "text"; text: string }
  >;
  encrypted_content: string | null;
}

type LocalShellStatus = "completed" | "in_progress" | "incomplete";

interface LocalShellCallItem {
  type: "local_shell_call";
  call_id: string | null;
  status: LocalShellStatus;
  action: {
    type: "exec";
    command: string[];
    timeout_ms: number | null;
    working_directory: string | null;
    env: Record<string, string> | null;
    user: string | null;
  };
}

interface FunctionCallItem {
  type: "function_call";
  name: string;
  namespace?: string;
  /**
   * 注意：这里是 JSON 字符串，不是已经 parse 好的对象。
   */
  arguments: string;
  call_id: string;
}

interface ToolSearchCallItem {
  type: "tool_search_call";
  call_id: string | null;
  status?: string;
  execution: string;
  arguments: unknown;
}

type FunctionCallOutputContentItem =
  | { type: "input_text"; text: string }
  | { type: "input_image"; image_url: string; detail?: ImageDetail };

/**
 * 工具输出在 wire 上是 string 或结构化 content item 数组。
 * 内部还有 success，但不会序列化进 response_item.output。
 */
type FunctionCallOutputBody = string | FunctionCallOutputContentItem[];

interface FunctionCallOutputItem {
  type: "function_call_output";
  call_id: string;
  output: FunctionCallOutputBody;
}

interface CustomToolCallItem {
  type: "custom_tool_call";
  status?: string;
  call_id: string;
  name: string;
  input: string;
}

interface CustomToolCallOutputItem {
  type: "custom_tool_call_output";
  call_id: string;
  name?: string;
  output: FunctionCallOutputBody;
}

interface ToolSearchOutputItem {
  type: "tool_search_output";
  call_id: string | null;
  status: string;
  execution: string;
  tools: unknown[];
}

type WebSearchAction =
  | { type: "search"; query?: string; queries?: string[] }
  | { type: "open_page"; url?: string }
  | { type: "find_in_page"; url?: string; pattern?: string }
  | { type: "other" };

interface WebSearchCallItem {
  type: "web_search_call";
  status?: string;
  action?: WebSearchAction;
}

interface ImageGenerationCallItem {
  type: "image_generation_call";
  id: string;
  status: string;
  revised_prompt?: string;
  result: string;
}

interface CompactionResponseItem {
  type: "compaction";
  encrypted_content: string;
}

interface ContextCompactionResponseItem {
  type: "context_compaction";
  encrypted_content?: string;
}

interface OtherResponseItem {
  type: "other";
}
```

主要场景：

| 场景 | 结构 | 来源 | 含义 |
| --- | --- | --- | --- |
| 用户输入 | `message(role="user")` | turn start 时把用户文本、图片、local image 标签转换成 input content | 用户本轮真实请求。也是 rollback 计算 user turn 边界的重要依据。 |
| 初始/变更上下文 | `message(role="developer")` 或 `message(role="user")` | 首轮或配置变化时构造的上下文注入 | AGENTS、skills、plugins、环境、权限、日期等模型可见上下文。 |
| 助手输出 | `message(role="assistant")` | 模型响应流完成/聚合后 | 用户可见回答。`phase` 可区分中途 commentary 和最终答案。 |
| 推理 | `reasoning` | 模型响应 | 推理摘要和加密推理内容。是否可见取决于模型/provider。 |
| 工具调用 | `function_call` / `custom_tool_call` / `local_shell_call` / `web_search_call` / `image_generation_call` | 模型要求调用工具 | 模型提出的动作，不等于动作已经完成。 |
| 工具结果 | `function_call_output` / `custom_tool_call_output` / `tool_search_output` | 工具执行完后回灌 | 下一次采样时模型可见的工具结果。过长输出会在进入内存历史时被截断。 |
| compact 响应项 | `compaction` / `context_compaction` | 压缩相关模型响应 | 压缩任务的模型返回，不等同于顶层 `compacted` 检查点。 |
| 未识别项 | `other` | 兼容未知 Responses API item | 不持久化为有效历史，也不会进入内存历史。 |

## 5. `turn_context`

```ts
type AskForApproval =
  | "untrusted"
  | "on-failure"
  | "on-request"
  | "never"
  | { granular: GranularApprovalConfig };

interface GranularApprovalConfig {
  sandbox_approval: boolean;
  rules: boolean;
  skill_approval: boolean;
  request_permissions: boolean;
  mcp_elicitations: boolean;
}

type NetworkAccess = "restricted" | "enabled";

type SandboxPolicy =
  | { type: "danger-full-access" }
  | { type: "read-only"; network_access?: boolean }
  | { type: "external-sandbox"; network_access?: NetworkAccess }
  | {
      type: "workspace-write";
      writable_roots?: PathString[];
      network_access?: boolean;
      exclude_tmpdir_env_var?: boolean;
      exclude_slash_tmp?: boolean;
    };

type PermissionProfile = Json;
type FileSystemSandboxPolicy = Json;
type Personality = string;
type CollaborationMode = Json;
type ReasoningEffortConfig = string;
type ReasoningSummaryConfig = string | Json;

interface TurnContextPayload {
  turn_id?: string;
  cwd: PathString;
  workspace_roots?: PathString[];
  current_date?: string;
  timezone?: string;
  approval_policy: AskForApproval;
  sandbox_policy: SandboxPolicy;
  permission_profile?: PermissionProfile;
  network?: {
    allowed_domains: string[];
    denied_domains: string[];
  };
  file_system_sandbox_policy?: FileSystemSandboxPolicy;
  model: string;
  personality?: Personality;
  collaboration_mode?: CollaborationMode;
  realtime_active?: boolean;
  effort?: ReasoningEffortConfig;
  /**
   * 兼容旧版本反序列化用。当前恢复逻辑不再读取它重建 context。
   */
  summary: ReasoningSummaryConfig;
}
```

字段含义：

| 字段 | 来源 | 含义 |
| --- | --- | --- |
| `turn_id` | turn 创建 | 把事件、工具调用、日志和该 turn 关联起来。 |
| `cwd` | 当前 turn 运行环境 | 工具执行和上下文说明的工作目录。 |
| `workspace_roots` | 当前环境/权限解析 | 物化 permission profile 里的 `:workspace_roots` 符号权限，保证恢复时能按当时工作区边界解释文件系统权限。 |
| `current_date` / `timezone` | 环境上下文 | 发给模型的日期和时区上下文来源。 |
| `approval_policy` | session/config/权限选择 | 决定 shell、权限、MCP elicitation 等是否向用户确认。 |
| `sandbox_policy` | session/config | 旧版 sandbox 策略。恢复时可推导权限 profile。 |
| `permission_profile` | 当前权限模型 | 新版细粒度权限快照。旧记录缺失时，会从 `sandbox_policy`、`file_system_sandbox_policy` 和网络策略推导。 |
| `network` | 网络权限 | 本 turn 允许/拒绝的域名集合。 |
| `file_system_sandbox_policy` | 文件系统权限 | 细粒度读写范围。 |
| `model` / `effort` / `summary` | 模型选择 | 采样时使用的模型和推理设置。 |
| `personality` / `collaboration_mode` | agent 行为配置 | 影响基础指令和协作方式。 |
| `realtime_active` | realtime 状态 | 该 turn 是否处于实时对话。 |
| `summary` | 兼容字段 | 当前源码注释说明它主要用于旧版本反序列化兼容，不再作为 context reconstruction 的输入。 |

业务上，`turn_context` 不是聊天消息。它是“下一次恢复时如何重新构造上下文 diff”的基线。

注意：运行时 `TurnContext` 里还会有 trace、输出 schema、截断策略等信息，但这些不等于 `protocol::TurnContextItem` 的 JSONL 落盘字段。读 JSONL schema 时要以 `codex-rs/protocol/src/protocol.rs` 的 `TurnContextItem` 为准。

## 6. `compacted`

```ts
interface CompactedPayload {
  /**
   * 压缩后的自然语言摘要。旧恢复逻辑会把它包装成 assistant message。
   */
  message: string;
  /**
   * 新版压缩检查点。存在时，恢复历史直接替换为这组 ResponseItem。
   */
  replacement_history?: ResponseItem[];
}
```

场景：

| 场景 | 数据 | 来源 | 恢复语义 |
| --- | --- | --- | --- |
| 新版 compact | `message + replacement_history` | compact 完成后把内存历史替换为压缩后的历史 | 恢复时找到最新仍然有效的 compact 检查点，从 `replacement_history` 开始，再重放后续 `response_item`。 |
| 旧版 compact | 只有 `message` | 旧 rollout | 恢复时用 `message` 和已收集的用户消息重建一个压缩历史，精度较低。 |
| compact 后上下文重建 | `compacted` 后紧跟可能的 `turn_context` | compact 后重新建立上下文 baseline | 恢复时用这个 `turn_context` 作为后续 diff 基线。 |

## 7. `event_msg`

事件使用二级判别字段 `payload.type`。源码里的事件枚举名是 `TurnStarted` / `TurnComplete`，但 JSONL wire 字段不会写成 `turn_started` / `turn_complete`；它们按旧协议名称保存为 `task_started` / `task_complete`，同时反序列化时兼容 `turn_started` / `turn_complete`。

```ts
type StoredEventMsg = LimitedStoredEventMsg | ExtendedOnlyStoredEventMsg;

type LimitedStoredEventMsg =
  | UserMessageEvent
  | AgentMessageEvent
  | AgentReasoningEvent
  | AgentReasoningRawContentEvent
  | PatchApplyEndEvent
  | TokenCountEvent
  | ContextCompactedEvent
  | EnteredReviewModeEvent
  | ExitedReviewModeEvent
  | McpToolCallEndEvent
  | ThreadRolledBackEvent
  | TurnAbortedEvent
  | TaskStartedEvent
  | TaskCompleteEvent
  | WebSearchEndEvent
  | ImageGenerationEndEvent
  | ItemCompletedPlanEvent;

type ExtendedOnlyStoredEventMsg =
  | ErrorEvent
  | GuardianAssessmentEvent
  | ExecCommandEndEvent
  | ViewImageToolCallEvent
  | CollabAgentSpawnEndEvent
  | CollabAgentInteractionEndEvent
  | CollabWaitingEndEvent
  | CollabCloseEndEvent
  | CollabResumeEndEvent
  | DynamicToolCallRequestEvent
  | DynamicToolCallResponseEvent;
```

### 7.1 Limited 模式会保存的事件

```ts
interface UserMessageEvent {
  type: "user_message";
  message: string;
  images?: string[];
  local_images: PathString[];
  text_elements: Json[];
}

interface AgentMessageEvent {
  type: "agent_message";
  message: string;
  phase: MessagePhase | null;
  memory_citation: Json | null;
}

interface AgentReasoningEvent {
  type: "agent_reasoning";
  text: string;
}

interface AgentReasoningRawContentEvent {
  type: "agent_reasoning_raw_content";
  text: string;
}

interface PatchApplyEndEvent {
  type: "patch_apply_end";
  call_id: string;
  turn_id: string;
  stdout: string;
  stderr: string;
  success: boolean;
  changes: Record<PathString, Json>;
  status: "completed" | "failed" | "declined";
}

interface TokenUsage {
  input_tokens: number;
  cached_input_tokens: number;
  output_tokens: number;
  reasoning_output_tokens: number;
  total_tokens: number;
}

interface TokenUsageInfo {
  total_token_usage: TokenUsage;
  last_token_usage: TokenUsage;
  model_context_window: number | null;
}

interface RateLimitWindow {
  used_percent: number;
  window_minutes: number | null;
  resets_at: number | null;
}

interface CreditsSnapshot {
  has_credits: boolean;
  unlimited: boolean;
  balance: string | null;
}

interface RateLimitSnapshot {
  limit_id: string | null;
  limit_name: string | null;
  primary: RateLimitWindow | null;
  secondary: RateLimitWindow | null;
  credits: CreditsSnapshot | null;
  plan_type: Json | null;
  rate_limit_reached_type:
    | "rate_limit_reached"
    | "workspace_owner_credits_depleted"
    | "workspace_member_credits_depleted"
    | "workspace_owner_usage_limit_reached"
    | "workspace_member_usage_limit_reached"
    | null;
}

interface TokenCountEvent {
  type: "token_count";
  info: TokenUsageInfo | null;
  rate_limits: RateLimitSnapshot | null;
}

interface ContextCompactedEvent {
  type: "context_compacted";
}

type ReviewRequestJson = Json;
type ReviewOutputEventJson = Json;

type EnteredReviewModeEvent = {
  type: "entered_review_mode";
  [key: string]: Json;
};

interface ExitedReviewModeEvent {
  type: "exited_review_mode";
  review_output: ReviewOutputEventJson | null;
}

interface McpInvocation {
  server: string;
  tool: string;
  arguments: Json | null;
}

interface CallToolResult {
  content: Json[];
  structuredContent?: Json | null;
  isError?: boolean | null;
  _meta?: Json | null;
}

interface McpToolCallEndEvent {
  type: "mcp_tool_call_end";
  call_id: string;
  invocation: McpInvocation;
  mcp_app_resource_uri?: string;
  duration: DurationString;
  result: { Ok: CallToolResult } | { Err: string };
}

interface ThreadRolledBackEvent {
  type: "thread_rolled_back";
  num_turns: number;
}

type TurnAbortReason = "interrupted" | "replaced" | "review_ended" | "budget_limited";

interface TurnAbortedEvent {
  type: "turn_aborted";
  turn_id: string | null;
  reason: TurnAbortReason;
  completed_at?: UnixSeconds | null;
  duration_ms?: Milliseconds | null;
}

interface TaskStartedEvent {
  type: "task_started";
  turn_id: string;
  started_at?: UnixSeconds | null;
  model_context_window: number | null;
  collaboration_mode_kind: Json;
}

interface TaskCompleteEvent {
  type: "task_complete";
  turn_id: string;
  last_agent_message: string | null;
  completed_at?: UnixSeconds | null;
  duration_ms?: Milliseconds | null;
  time_to_first_token_ms?: Milliseconds | null;
}

interface WebSearchEndEvent {
  type: "web_search_end";
  call_id: string;
  query: string;
  action: WebSearchAction;
}

interface ImageGenerationEndEvent {
  type: "image_generation_end";
  call_id: string;
  status: string;
  revised_prompt?: string;
  result: string;
  saved_path?: PathString;
}

/**
 * JSONL limited 模式只持久化 Plan item 的 item_completed。
 * 其它 item_completed 不落盘。
 */
interface ItemCompletedPlanEvent {
  type: "item_completed";
  thread_id: ThreadId;
  turn_id: string;
  item: Json; // TurnItem::Plan
  completed_at_ms: Milliseconds;
}
```

### 7.2 Extended 模式额外保存的事件

```ts
interface ErrorEvent {
  type: "error";
  message: string;
  codex_error_info: Json | null;
}

type GuardianRiskLevel = "low" | "medium" | "high" | "critical";
type GuardianUserAuthorization = "unknown" | "low" | "medium" | "high";
type GuardianAssessmentStatus =
  | "in_progress"
  | "approved"
  | "denied"
  | "timed_out"
  | "aborted";

interface GuardianAssessmentEvent {
  type: "guardian_assessment";
  id: string;
  target_item_id?: string;
  turn_id: string;
  status: GuardianAssessmentStatus;
  risk_level?: GuardianRiskLevel;
  user_authorization?: GuardianUserAuthorization;
  rationale?: string;
  decision_source?: "agent";
  action: Json;
}

type ExecCommandStatus = "completed" | "failed" | "declined";
type ExecCommandSource = Json;
type ParsedCommand = Json;

interface ExecCommandEndEvent {
  type: "exec_command_end";
  call_id: string;
  process_id?: string;
  turn_id: string;
  completed_at_ms: Milliseconds;
  command: string[];
  cwd: PathString;
  parsed_cmd: ParsedCommand[];
  source: ExecCommandSource;
  interaction_input?: string;
  stdout: string;
  stderr: string;
  aggregated_output: string;
  exit_code: number;
  duration: DurationString;
  formatted_output: string;
  status: ExecCommandStatus;
}

interface ViewImageToolCallEvent {
  type: "view_image_tool_call";
  call_id: string;
  path: PathString;
}

type AgentStatus =
  | "pending_init"
  | "running"
  | "interrupted"
  | { completed: string | null }
  | { errored: string }
  | "shutdown"
  | "not_found";

interface CollabAgentStatusEntry {
  thread_id: ThreadId;
  agent_nickname?: string;
  agent_role?: string;
  status: AgentStatus;
}

interface CollabAgentSpawnEndEvent {
  type: "collab_agent_spawn_end";
  call_id: string;
  completed_at_ms: Milliseconds;
  sender_thread_id: ThreadId;
  new_thread_id: ThreadId | null;
  new_agent_nickname?: string;
  new_agent_role?: string;
  prompt: string;
  model: string;
  reasoning_effort: ReasoningEffortConfig;
  status: AgentStatus;
}

interface CollabAgentInteractionEndEvent {
  type: "collab_agent_interaction_end";
  call_id: string;
  completed_at_ms: Milliseconds;
  sender_thread_id: ThreadId;
  receiver_thread_id: ThreadId;
  receiver_agent_nickname?: string;
  receiver_agent_role?: string;
  prompt: string;
  status: AgentStatus;
}

interface CollabWaitingEndEvent {
  type: "collab_waiting_end";
  sender_thread_id: ThreadId;
  call_id: string;
  completed_at_ms: Milliseconds;
  agent_statuses: CollabAgentStatusEntry[];
  statuses: Record<ThreadId, AgentStatus>;
}

interface CollabCloseEndEvent {
  type: "collab_close_end";
  call_id: string;
  completed_at_ms: Milliseconds;
  sender_thread_id: ThreadId;
  receiver_thread_id: ThreadId;
  receiver_agent_nickname?: string;
  receiver_agent_role?: string;
  status: AgentStatus;
}

interface CollabResumeEndEvent {
  type: "collab_resume_end";
  call_id: string;
  completed_at_ms: Milliseconds;
  sender_thread_id: ThreadId;
  receiver_thread_id: ThreadId;
  receiver_agent_nickname?: string;
  receiver_agent_role?: string;
  status: AgentStatus;
}

interface DynamicToolCallRequestEvent {
  type: "dynamic_tool_call_request";
  callId: string;
  turnId: string;
  startedAtMs: Milliseconds;
  namespace: string | null;
  tool: string;
  arguments: Json;
}

type DynamicToolCallOutputContentItem =
  | { type: "inputText"; text: string }
  | { type: "inputImage"; imageUrl: string };

interface DynamicToolCallResponseEvent {
  type: "dynamic_tool_call_response";
  call_id: string;
  turn_id: string;
  completed_at_ms: Milliseconds;
  namespace: string | null;
  tool: string;
  arguments: Json;
  content_items: DynamicToolCallOutputContentItem[];
  success: boolean;
  error: string | null;
  duration: DurationString;
}
```

### 7.3 不会落盘的事件

这些事件可以在运行时发给 UI，但按当前持久化策略不会写入 thread JSONL：warnings、guardian warning、realtime 流、model reroute/verification、raw response item、session configured、thread goal update、MCP begin、exec begin/output delta/terminal interaction、approval request、request user input、elicitation、apply patch begin/updated、stream error、turn diff、voice list、MCP startup、web search begin、plan update、shutdown complete、hook started/completed、agent message delta、plan delta、reasoning delta、image generation begin、skills update、collab begin 类事件等。

这说明 JSONL 不是完整 UI 事件流，而是“足够恢复和审计的事件子集”。

## 8. 恢复时如何计算历史

恢复不是把 JSONL 原样当 prompt。核心算法可以用通用流程表示：

```text
读取 JSONL 全部行
  -> 反向扫描
      -> 找最新仍然有效的 compact replacement_history
      -> 找最新 turn_context，作为上下文 diff baseline
      -> 累计 thread_rolled_back 要丢弃的用户 turn 数
      -> 识别 task_started / task_complete / user_message / response_item 形成 turn segment
  -> 从 compact checkpoint 之后正向重放
      -> response_item 进入内存历史
      -> compact 有 replacement_history 时替换内存历史
      -> thread_rolled_back 时从内存历史丢弃最后 N 个 user turn
      -> session_meta / turn_context / 普通 event_msg 不进入聊天历史
  -> 得到 raw in-memory history
  -> 发送模型前再 normalize:
      -> 补齐缺失的工具输出
      -> 删除孤儿工具输出
      -> 模型不支持图片时剥离图片
```

进入内存历史时的过滤规则：

| 记录 | 处理 |
| --- | --- |
| `ResponseItem::Message(role="system")` | 不进入内存历史。 |
| `ResponseItem::Other` | 不进入内存历史，也不持久化为有效 response item。 |
| `function_call_output` / `custom_tool_call_output` | 进入前按 truncation policy 截断。 |
| 其它 API message、reasoning、tool call、tool output | 进入内存历史。 |
| `event_msg.thread_rolled_back` | 不作为消息进入历史，但会删除最后 N 个用户 turn。 |
| `event_msg.user_message` | 主要用于 UI/replay 分段，不能替代 `response_item.message(role="user")`。 |

## 9. Web Word Agent 可借鉴的数据模型

如果你要自己实现 agent，建议不要把 JSONL 设计成单一 `messages[]`。可以拆成同一个 append-only log 里的几类记录：

```ts
type OwnAgentLogLine =
  | { type: "thread_meta"; payload: OwnThreadMeta }
  | { type: "model_history_item"; payload: OwnModelHistoryItem }
  | { type: "turn_context_snapshot"; payload: OwnTurnContextSnapshot }
  | { type: "history_checkpoint"; payload: OwnCompactionCheckpoint }
  | { type: "recoverable_event"; payload: OwnRecoverableEvent };
```

对于 Web Word，建议至少持久化：

| 类别 | 必要字段 | 目的 |
| --- | --- | --- |
| thread meta | threadId、documentId、workspaceId、creator、agent version、base instructions version | 恢复 thread 身份和产品上下文。 |
| model history item | role、content、tool call、tool result、attachments | 构造下一次模型输入。 |
| turn context snapshot | document revision、selection、permission profile、available tools、model、date/timezone | 防止 resume 后上下文基线漂移。 |
| history checkpoint | summary、replacement history、covered item range | 支持长文档协作下的 compact。 |
| recoverable event | turn started/completed、rollback、token usage、tool end、error | UI 回放、排障、恢复 turn 状态。 |

关键设计原则：

1. `model_history_item` 是候选历史，不等于最终 prompt。
2. `turn_context_snapshot` 记录“当时如何运行”，不是给模型看的聊天消息。
3. compact 应该保存可直接替换的 `replacement_history`，不要只保存摘要文本。
4. rollback 应该作为事件落盘，并在恢复时重新计算历史，而不是物理删除 JSONL 旧行。
5. UI 事件流和持久恢复日志要分层；运行时可以有很多 delta/begin 事件，但 JSONL 只保存恢复需要的子集。

## 源码对照

| 主题 | 源码位置 |
| --- | --- |
| JSONL 行结构 `RolloutLine` / `RolloutItem` | `codex-rs/protocol/src/protocol.rs` |
| `ResponseItem` / `ContentItem` / 工具输出 wire 结构 | `codex-rs/protocol/src/models.rs` |
| recorder 写 JSONL、外层 timestamp | `codex-rs/rollout/src/recorder.rs` |
| 持久化策略 Limited / Extended | `codex-rs/rollout/src/policy.rs` |
| session 记录 response item、compact、turn context | `codex-rs/core/src/session/mod.rs` |
| 从 rollout 重建历史 | `codex-rs/core/src/session/rollout_reconstruction.rs` |
| 内存历史过滤、截断、发送前 normalize | `codex-rs/core/src/context_manager/history.rs` |
