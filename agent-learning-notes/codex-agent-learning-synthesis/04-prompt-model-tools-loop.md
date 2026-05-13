# Prompt、模型与工具闭环整合

这一页整合“每次 turn 启动时发给模型什么”“模型会返回什么”“工具如何发现、调用、回灌”这三组问题。

## 1. 每次采样前，agent 不是只发用户消息

一次模型请求通常由这些部分组成：

| 数据 | 来源 | 作用 |
| --- | --- | --- |
| 基础指令 | agent/runtime 配置 | 定义 agent 身份、行为边界、默认规则 |
| developer/context 指令 | 项目规则、产品规则、运行上下文 | 注入当前环境和高优先级约束 |
| 历史输入 | session history 加工后 | 让模型知道之前发生了什么 |
| 当前用户输入 | 本 turn 用户消息 | 定义当前目标 |
| 上下文 initial/diff | turn context 与 baseline | 告诉模型环境、权限、日期、工具、文件/文档状态变化 |
| 工具定义 | 每次采样前发现/构建 | 告诉模型可以请求哪些外部能力 |
| 推理配置 | 模型配置 | 控制 reasoning、token、模型能力 |
| 输出配置 | turn 配置 | 控制最终回答格式或结构化输出 |

通用结构可以表达成：

```ts
type ModelRequest = {
  instructions: string;
  input: PromptInputItem[];
  tools: ModelVisibleToolSpec[];
  reasoning?: ReasoningConfig;
  output?: OutputConfig;
  metadata?: RequestMetadata;
};
```

注意：

> 这不是业界统一标准结构，而是 agent runtime 的内部抽象。具体模型供应商可能会再转换成自己的 API 格式。

## 2. Prompt input 是数组，但不是普通聊天数组

`input` 可以理解成“模型可见的有序输入片段数组”。

它通常包含：

```ts
type PromptInputItem =
  | { type: "message"; role: "user" | "assistant"; content: ContentPart[] }
  | { type: "reasoning"; summary?: string[] }
  | { type: "function_call"; callId: string; name: string; arguments: string }
  | { type: "function_call_output"; callId: string; output: string }
  | { type: "custom_tool_call"; callId: string; name: string; input: string }
  | { type: "custom_tool_call_output"; callId: string; output: string }
  | { type: "context"; scope: "developer" | "user"; content: ContentPart[] };

type ContentPart =
  | { type: "text"; text: string }
  | { type: "image"; imageUrl: string }
  | { type: "file"; fileId?: string; name?: string; text?: string }
  | { type: "structured"; value: unknown };
```

数组顺序有意义：

- 先放更早历史。
- 再放上下文注入。
- 再放当前用户消息。
- 工具 call 和 output 要保持可理解的因果关系。

数组内允许重复 type，因为一次请求可能包含多个用户消息、多个工具输出、多个上下文片段。

## 3. 工具发现不是静态配置

Codex 的启发是：工具应在每次采样前重新计算，而不是启动时固定。

工具来源包括：

- 内置工具。
- 环境工具。
- MCP 工具。
- Apps / connectors。
- plugins。
- skills 贡献的能力。
- dynamic tools。
- hosted tools。
- deferred tools。
- unavailable dummy tools。

通用算法：

```text
每次采样前
  -> 读取当前 turn context
  -> 收集内置工具
  -> 收集外部连接工具
  -> 收集动态工具
  -> 根据模型能力过滤
  -> 根据权限和配置过滤
  -> 决定 direct 或 deferred 暴露
  -> 构建模型可见 tools
  -> 构建运行时 tool registry
```

## 4. direct 和 deferred

| 类型 | 模型是否直接看到 | 适合场景 |
| --- | --- | --- |
| direct tool | 是 | 高频、关键、数量少、语义明确 |
| deferred tool | 否，先通过 tool_search 发现 | 数量多、低频、外部连接器工具 |
| unavailable dummy tool | 可能可见 | 解释“这个工具当前不可用”的原因 |
| hosted tool | 可见，但由模型服务侧执行 | web search、image generation 等服务侧能力 |

Web Word 中建议：

- `document.getSelection`、`document.createSuggestion` 这类核心工具 direct。
- 模板库、知识库、CRM、合同系统等大量外部工具 deferred。
- 当前用户无权限的工具可以 dummy 化，让模型能解释限制。

## 5. 模型如何发起工具调用

模型不会直接执行工具。它只返回结构化 tool call。

```ts
type ModelOutputItem =
  | { type: "message"; role: "assistant"; content: ContentPart[] }
  | { type: "reasoning"; text?: string; summary?: string[] }
  | { type: "function_call"; callId: string; name: string; arguments: string }
  | { type: "custom_tool_call"; callId: string; name: string; input: string }
  | { type: "tool_search_call"; callId: string; query: string }
  | { type: "hosted_tool_call"; callId: string; name: string; status: string };
```

agent 收到 tool call 后做这些事：

```text
模型输出 tool call
  -> 记录 tool call item
  -> 根据 name 找 tool handler
  -> 校验参数
  -> 权限/审批/沙箱判断
  -> 执行业务能力
  -> 生成 tool output
  -> 记录 tool output
  -> 触发后续采样
```

为什么 tool call 要先记录再执行：

- UI 可以立即显示“模型准备调用什么”。
- 如果工具执行失败，历史里仍能看到模型原始意图。
- 后续采样需要 call/output 配对。
- 审计时能区分“模型请求了什么”和“工具实际返回什么”。

## 6. 工具运行时职责

工具运行时不只是 `callFunction()`。

它至少要负责：

- 路由：找到正确 handler。
- 参数校验。
- 权限判断。
- 用户审批。
- 沙箱或隔离。
- 并发控制。
- 取消和超时。
- 错误包装。
- 输出截断。
- 事件上报。
- 持久化。

```ts
type ToolCall = {
  id: string;
  name: string;
  kind: "function" | "custom" | "hosted" | "search";
  arguments: unknown;
  origin: "model";
};

type ToolOutput = {
  callId: string;
  status: "success" | "error" | "denied" | "cancelled";
  content: ContentPart[];
  metadata?: unknown;
};
```

## 7. 工具结果如何回灌模型

工具执行完成后，agent 不应该只更新 UI。它还要把工具结果变成模型可读 input item。

```text
tool output
  -> normalize / truncate
  -> append to session history
  -> persist selected item/event
  -> next prompt input includes output
  -> model continues reasoning
```

如果工具输出很大，要做：

- 截断。
- 摘要。
- 保留结构化关键字段。
- 给出可再次查询的引用 id。

Web Word 里尤其重要：

- 工具结果要包含文档位置。
- 修改建议要包含 suggestion id。
- 批注要包含 comment id。
- 长文档查询结果要包含 range 和引用来源。

## 8. 一次完整工具闭环

```text
用户：帮我把第二段改得更正式
  -> prompt 包含当前选区和 document tools
  -> 模型请求 document.getSelection
  -> agent 执行工具，返回选区文本和 range
  -> 后续采样
  -> 模型请求 document.createSuggestion
  -> agent 创建建议，不直接改正文
  -> 后续采样
  -> 模型说明已生成建议，等待用户接受
  -> turn complete
```

这里模型没有直接修改数据库；它通过工具表达意图，agent runtime 负责受控执行。

