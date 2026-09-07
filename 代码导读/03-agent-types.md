# 精读 03：types.ts —— Agent 运行时的全部契约（一个文件读懂扩展点）

> 对应源码：`packages/agent/src/types.ts`（445 行）
> 定位：agent 包的全部核心类型定义。**这个文件里没有一行运行时逻辑，但它定义了整个运行时的"宪法"**——循环怎么调 LLM、工具长什么样、有哪些钩子、发哪些事件。读懂这个文件，你就知道 agent 运行时有哪些可插拔点。
> 前置：建议先浏览 [01-agent-loop.md](01-agent-loop.md)（这些类型在哪里被用）。

## 类型总览（按阅读顺序分八组）

| 组 | 类型 | 作用 |
| --- | --- | --- |
| 一 | `StreamFn` | 怎么调 LLM（唯一的"传输"抽象） |
| 二 | `ToolExecutionMode` / `QueueMode` / `ThinkingLevel` | 三个枚举别名 |
| 三 | `BeforeToolCallResult` / `AfterToolCallResult` + 两个 Context | 工具钩子的输入输出 |
| 四 | `ShouldStopAfterTurnContext` / `AgentLoopTurnUpdate` / `AgentLoopConfig` | 循环的全部配置点 |
| 五 | `CustomAgentMessages` / `AgentMessage` | 可扩展的消息模型 |
| 六 | `AgentState` / `AgentContext` | 状态与上下文快照 |
| 七 | `AgentToolResult` / `AgentTool` | 工具契约 |
| 八 | `AgentEvent` | 事件协议 |

---

## 一、`StreamFn`（第 18~32 行）——传输无关的关键

```ts
/**
 * Stream function used by the agent loop. `Models.streamSimple` satisfies this shape.
 *
 * Contract:
 * - Must not throw or return a rejected promise for request/model/runtime failures.
 *   （对请求/模型/运行时失败：绝不 throw、绝不返回 rejected promise）
 * - Must return an AssistantMessageEventStream.
 *   （必须返回事件流）
 * - Failures must be encoded in the returned stream via protocol events and a
 *   final AssistantMessage with stopReason "error" or "aborted" and errorMessage.
 *   （失败必须编码在流内：以 stopReason 为 error/aborted 的最终消息收尾）
 */
export type StreamFn = (
	model: Model<Api>,
	context: Context,
	options?: SimpleStreamOptions,
) => AssistantMessageEventStream | Promise<AssistantMessageEventStream>;
```

**这是整个"传输无关"设计的支点**。agent 循环不认识 HTTP、不认识 SSE、不认识任何 provider——它只认识这个函数签名。注释里写明 `Models.streamSimple`（pi-ai 的标准实现）满足这个形状，但你可以换成任何东西：

- 测试 → 假流函数（脚本化响应）；
- 浏览器 → `streamProxy`（经后端代理，见 agent 包 `src/proxy.ts`）；
- coding-agent → 包装过的 streamSimple（叠加重试/超时/扩展事件，见 [05-sdk.md](05-sdk.md)）。

**三条契约缺一不可**，尤其是"绝不 throw"——因为循环对返回流只做了"事件驱动"的处理，没有任何 try-catch 兜底（错误抛出会直接打断循环、破坏事件序列的完整性，只能靠 Agent 类的 `handleRunFailure` 合成兜底事件）。

## 二、三个枚举（第 34~50 行 + 296~301 行）

```ts
// 单条 assistant 消息里多个工具调用的执行方式：
// sequential = 一个完整跑完（备好→执行→收尾）再下一个
// parallel   = 逐个预检（顺序），然后并发执行；tool_execution_end 按完成序发，
//              而 toolResult 消息按 assistant 源顺序（持久化序）发
export type ToolExecutionMode = "sequential" | "parallel";

// 队列排空模式：
// all            = 到达注入点时把队列里所有消息一起注入
// one-at-a-time  = 只注入最旧的一条，剩下的留给下一个注入点
export type QueueMode = "all" | "one-at-a-time";

// 推理（thinking）等级。注意 xhigh/max 只有部分模型家族支持，
// 具体支持情况要用 pi-ai 的模型元数据判断
export type ThinkingLevel = "off" | "minimal" | "low" | "medium" | "high" | "xhigh" | "max";
```

TS 里这种"字符串字面量联合"就是 Java 的枚举，而且更轻——它就是字符串，可直接比较、可 JSON 序列化。`ToolExecutionMode` 的注释把 parallel 模式的**三种顺序**（预检序/完成序/持久化序）写在了类型定义里——好的类型注释就是设计文档。

## 三、工具钩子的输入输出（第 52~123 行）

```ts
// Extract 类型技巧：从 AssistantMessage 的 content 数组的元素类型里，
// 直接"抽出" type 为 "toolCall" 的那个成员。
// 等价于 Java: sealed interface Content 的 ToolCall 分支。
// 好处：pi-ai 改了 content 类型定义，这里自动跟着变。
export type AgentToolCall = Extract<AssistantMessage["content"][number], { type: "toolCall" }>;

/** beforeToolCall 的返回值 */
export interface BeforeToolCallResult {
	block?: boolean;    // true = 拦截，不执行这个工具
	reason?: string;    // 拦截原因——会成为喂回模型的错误文本（这是写给模型看的！）
	terminate?: boolean; // 提示"这批工具跑完就停"。注意：只有批内【所有】结果都
	                     // terminate=true 才会真的提前终止（见 01 的 shouldTerminateToolBatch）
}

/** afterToolCall 的返回值：字段级覆盖，省略的字段保留原值 */
export interface AfterToolCallResult {
	content?: (TextContent | ImageContent)[]; // 给定则【整体替换】content 数组
	details?: unknown;                        // 给定则整体替换 details
	isError?: boolean;                        // 给定则覆盖错误标志
	usage?: Usage;
	terminate?: boolean;
	// 注意注释明确说了：没有深合并（No deep merge）——给了 content 就整个换掉，
	// 不是往里追加。这个语义选择让钩子行为可预测。
}

/** beforeToolCall 收到的上下文 */
export interface BeforeToolCallContext {
	assistantMessage: AssistantMessage; // 请求这次工具调用的那条 assistant 消息
	toolCall: AgentToolCall;            // 原始工具调用块（未经校验的原始参数）
	args: unknown;                      // 【已通过 TypeBox 校验】的参数
	context: AgentContext;              // 预检时刻的 agent 上下文
}

/** afterToolCall 收到的上下文：比 before 多了执行结果 */
export interface AfterToolCallContext {
	assistantMessage: AssistantMessage;
	toolCall: AgentToolCall;
	args: unknown;
	result: AgentToolResult<any>;       // 覆盖前的原始执行结果
	isError: boolean;                   // 当前是否被视为错误
	context: AgentContext;
}
```

**设计模式识别**：这对钩子就是 Servlet 的 Filter / Spring AOP 的 Around——但用显式的数据结构（Context 进、Result 出）而不是注解 + 反射。`block + reason` 是"带原因的拒绝"，`AfterToolCallResult` 是"可选的 Patch"。

## 四、循环配置 `AgentLoopConfig`（第 125~294 行）——最核心的接口

先看两个配套类型：

```ts
/** shouldStopAfterTurn 收到的上下文 */
export interface ShouldStopAfterTurnContext {
	message: AssistantMessage;      // 完成本 turn 的 assistant 消息
	toolResults: ToolResultMessage[]; // 这个 turn 的工具结果
	context: AgentContext;          // turn 结束后的上下文
	newMessages: AgentMessage[];    // 如果此刻退出，本次运行会返回哪些消息
	// 注释特意说明：prompt 运行包含最初的 prompt 消息；continue 运行不包含
	// 运行前就存在的历史消息。
}

/** prepareNextTurn 的返回值：下一轮的替换状态 */
export interface AgentLoopTurnUpdate {
	context?: AgentContext;          // 替换整个上下文（压缩后的新上下文）
	model?: Model<any>;              // 换模型
	thinkingLevel?: ThinkingLevel;   // 换推理等级
}

// PrepareNextTurnContext 直接继承 ShouldStopAfterTurnContext（空接口扩展）：
// TS 的 interface 继承 = Java 的 extends，一个字段都没加，纯粹为了语义命名
export interface PrepareNextTurnContext extends ShouldStopAfterTurnContext {}
```

然后是主角（逐字段注释）：

```ts
export interface AgentLoopConfig extends SimpleStreamOptions {  // 继承了 LLM 请求选项（超时、重试、transport 等）
	model: Model<any>;   // 唯一必填字段①：用哪个模型

	/**
	 * 必填字段②：AgentMessage[] → Message[] 的边界转换。
	 * 每条 AgentMessage 必须转成 user/assistant/toolResult 之一；
	 * 无法转换的（纯 UI 通知类消息）应该被过滤掉。
	 * 契约：不许 throw——返回安全兜底值代替。抛出会打断循环且不产生正常事件序列。
	 */
	convertToLlm: (messages: AgentMessage[]) => Message[] | Promise<Message[]>;

	/**
	 * 可选：convertToLlm 之前的变换（AgentMessage[] → AgentMessage[]）。
	 * 用途：上下文窗口管理（裁剪旧消息）、注入外部上下文。
	 * 契约：同样不许 throw。
	 */
	transformContext?: (messages: AgentMessage[], signal?: AbortSignal) => Promise<AgentMessage[]>;

	/**
	 * 每次调 LLM 前动态解析 API key。
	 * 用途：短命 OAuth token（比如 GitHub Copilot 的）可能在长时间工具执行期间过期。
	 */
	getApiKey?: (provider: string) => Promise<string | undefined> | string | undefined;

	/**
	 * turn 完全结束（turn_end 已发）后调用。返回 true → 发 agent_end 退出，
	 * 不再轮询队列、不再发起下一次 LLM 调用。当前回复和工具执行正常收尾。
	 * 典型用途：上下文快满时优雅停止（压缩的触发点）。
	 */
	shouldStopAfterTurn?: (context: ShouldStopAfterTurnContext) => boolean | Promise<boolean>;

	/**
	 * 循环将继续时、下一个 turn 开始前调用（在 shouldStopAfterTurn 之后）。
	 * 返回替换的 context/model/thinkingLevel；返回 undefined = 保持现状。
	 */
	prepareNextTurn?: (context: PrepareNextTurnContext) => AgentLoopTurnUpdate | undefined | Promise<AgentLoopTurnUpdate | undefined>;

	/** 轮询 steering 消息（运行中插话）。契约：返回 [] 表示没有。 */
	getSteeringMessages?: () => Promise<AgentMessage[]>;

	/** 轮询 follow-up 消息（agent 本来要停时才问）。契约：返回 [] 表示没有。 */
	getFollowUpMessages?: () => Promise<AgentMessage[]>;

	/** 工具执行模式，默认 "parallel" */
	toolExecution?: ToolExecutionMode;

	/** 工具执行前钩子（参数校验之后）。返回 {block:true} 拦截。 */
	beforeToolCall?: (context: BeforeToolCallContext, signal?) => Promise<BeforeToolCallResult | undefined>;

	/** 工具执行后钩子（tool_execution_end 和结果消息发出之前）。可部分覆盖结果。 */
	afterToolCall?: (context: AfterToolCallContext, signal?) => Promise<AfterToolCallResult | undefined>;
}
```

**读后体会**：整个 agent 循环的可定制点只有这 10 个左右，而且**每个钩子的注释都写明了契约**（尤其"不许 throw"出现了三次）。这是接口设计的范本：宁可让实现方多读注释，也不在运行时做防御性 try-catch。

**Java 视角**：`AgentLoopConfig` 就是一个"策略集合对象"（一次传入所有策略），而不是 Java 常见的多个 setter 注入。好处是不可变、一次快照、循环内可整体替换（`prepareNextTurn` 换 config）。

## 五、可扩展消息模型（第 303~326 行）——本文件最"TS 味"的部分

```ts
/**
 * 自定义应用消息的可扩展接口。应用通过 declaration merging（声明合并）扩展：
 */
export interface CustomAgentMessages {
	// 默认为空——应用通过声明合并添加成员
}

/**
 * AgentMessage = LLM 标准消息 + 自定义消息的联合。
 */
export type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
```

**declaration merging 是什么？**（Java 没有对应物，值得单独讲）

TypeScript 里，同名 interface 的声明会自动合并。所以下游应用可以这样"往别人的接口里加字段"：

```ts
// 你的应用代码里写：
declare module "@earendil-works/pi-agent-core" {
	interface CustomAgentMessages {
		artifact: ArtifactMessage;       // 加一种自定义消息
		notification: NotificationMessage;
	}
}
```

编译期效果：`AgentMessage` 这个联合类型**自动**多出两个成员。不需要改 agent 包的代码，也不需要注册中心——类型系统本身就是注册中心。pi 的 harness 层自己就用这个机制注册了 `bashExecution`、`branchSummary`、`compactionSummary` 等消息（见 `harness/messages.ts`）。

代价：自定义消息**只存在于类型层面**，运行时它是普通对象，必须由你提供的 `convertToLlm` 决定它怎么呈现给 LLM（默认实现会把它们全部过滤掉——见 [02-agent.md](02-agent.md) 的 `defaultConvertToLlm`）。

## 六、`AgentState` 与 `AgentContext`（第 328~359、412~420 行）

```ts
/** 对外的 agent 状态（Agent 类的 _state 的公共视图） */
export interface AgentState {
	systemPrompt: string;              // 每次请求携带的系统提示词
	model: Model<any>;                 // 后续 turn 使用的模型
	thinkingLevel: ThinkingLevel;      // 请求的推理等级
	set tools(tools: AgentTool<any>[]);  // 接口里直接声明 getter/setter（罕见但合法）：
	get tools(): AgentTool<any>[];       // 赋值时实现会拷贝顶层数组（见 02 的工厂函数）
	set messages(messages: AgentMessage[]);
	get messages(): AgentMessage[];
	readonly isStreaming: boolean;       // 正在处理 prompt/continuation。
	                                    // 注意：直到 agent_end 的监听者都 settle 才变 false
	readonly streamingMessage?: AgentMessage;  // 当前流式响应的半成品消息
	readonly pendingToolCalls: ReadonlySet<string>; // 正在执行的工具调用 id（只读视图）
	readonly errorMessage?: string;     // 最近一次失败/中止的错误信息
}

/** 传给低层循环的上下文快照 */
export interface AgentContext {
	systemPrompt: string;          // 请求携带的系统提示词
	messages: AgentMessage[];      // 模型可见的转录
	tools?: AgentTool<any>[];      // 本次运行可用的工具
}
```

**State 与 Context 的区别**（容易混）：`AgentState` 是 Agent 类的长期状态（含运行时标志）；`AgentContext` 是**一次运行的输入快照**（只有 systemPrompt/messages/tools 三样）。Agent.prompt() 时把 state 切片成 context 交给循环（见 02 的 `createContextSnapshot`）。

## 七、工具契约 `AgentTool` / `AgentToolResult`（第 361~410 行）

```ts
/** 工具产出：最终结果或部分结果 */
export interface AgentToolResult<T> {
	content: (TextContent | ImageContent)[]; // 返回给模型的文本/图片（会进模型上下文）
	details: T;                              // 给日志/UI 的任意结构化数据（【不进】模型上下文）
	usage?: Usage;                           // 工具自身消耗的 usage（如果有）。
	                                         // 注意：不计入主 LLM 上下文记账
	addedToolNames?: string[];               // 声明"本次结果引入了哪些新工具"——
	                                         // deferred tools 机制的钩子
	terminate?: boolean;                     // "这批工具跑完就停"的提示。
	                                         // 批内全部 terminate=true 才会真的停
}

/** 工具的流式进度回调。
 *  作用域限于当前 execute() 调用；工具的 Promise settle 之后的调用会被忽略 */
export type AgentToolUpdateCallback<T = any> = (partialResult: AgentToolResult<T>) => void;

/** agent 运行时的工具定义 */
export interface AgentTool<TParameters extends TSchema = TSchema, TDetails = any>
	extends Tool<TParameters>   // 继承 pi-ai 的 Tool：name/description/parameters（TypeBox schema）
{
	label: string;              // UI 显示用的人类可读名（Tool 没有，这是 agent 层加的）
	/** schema 校验【前】的参数兼容垫片。必须返回符合 TParameters 的对象 */
	prepareArguments?: (args: unknown) => Static<TParameters>;
	/** 执行工具。失败要 throw，不要把错误编码进 content！ */
	execute: (
		toolCallId: string,
		params: Static<TParameters>,          // Static<TSchema> = 从 schema 推导出的参数类型
		signal?: AbortSignal,                 // 取消信号，工具要自己响应
		onUpdate?: AgentToolUpdateCallback<TDetails>,  // 流式进度回调
	) => Promise<AgentToolResult<TDetails>>;
	/** 本工具的执行模式（可覆盖全局默认） */
	executionMode?: ToolExecutionMode;
}
```

**泛型读法**（Java 开发者看这段最费劲的是泛型约束）：

- `TParameters extends TSchema`：参数 schema 的类型必须是个 TypeBox schema 对象；
- `Static<TParameters>`：**从 schema 类型反推出"符合该 schema 的数据"的类型**。比如 schema 声明 `{path: Type.String()}`，`Static<typeof schema>` 就是 `{path: string}`。这是"运行时校验"和"编译期类型"共享一份定义的技巧——Java 里得靠注解处理器（如 Immutables）才能做到类似的事。
- `TDetails = any`：details 是给 UI 的，类型由工具作者自定。

**"失败要 throw"再强调一次**：`execute` 的注释和 `AgentToolResult` 的注释形成对照——错误进 content 会绕过 `isError` 标记，循环就无法正确处理。工具作者只管 throw，循环负责转成错误结果喂回模型。

## 八、`AgentEvent`（第 422~444 行）——事件协议全文

```ts
export type AgentEvent =
	// Agent 生命周期
	| { type: "agent_start" }                                    // 一次运行开始
	| { type: "agent_end"; messages: AgentMessage[] }            // 一次运行结束，携带全部新消息
	// Turn 生命周期——一个 turn = 一条 assistant 回复 + 它的工具调用/结果
	| { type: "turn_start" }
	| { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
	// 消息生命周期——对 user、assistant、toolResult 消息【都会】发
	| { type: "message_start"; message: AgentMessage }
	// 仅 assistant 消息在流式生成期间发；携带原始的 pi-ai 事件（UI 用 delta 渲染）
	| { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
	| { type: "message_end"; message: AgentMessage }
	// 工具执行生命周期
	| { type: "tool_execution_start"; toolCallId: string; toolName: string; args: any }
	| { type: "tool_execution_update"; toolCallId: string; toolName: string; args: any; partialResult: any }
	| { type: "tool_execution_end"; toolCallId: string; toolName: string; result: any; isError: boolean };
```

文件头注释里的关键语义：**`agent_end` 是一次运行发出的最后一个事件，但被 await 的 `Agent.subscribe()` 监听者仍属于运行结算的一部分——所有监听者完成后 agent 才算 idle**。（这就是 02 里推导过的事件屏障。）

**为什么用判别联合而不是类继承？** 每个 event 是 `{type: "xxx", ...}` 的普通对象，消费者 `switch (event.type)` 且编译器强制穷尽检查（漏一个 case 且没有 default 会报错，前提是开启对应编译选项）。JSON 可序列化、可持久化、可跨进程传输——Java 里等价于 sealed interface + record，而且是 Java 21 才有的能力。

---

## 自测

1. `StreamFn` 的三条契约分别是什么？违反"不许 throw"会发生什么？（回 01 的 `streamAssistantResponse`：循环没有对它 try-catch）
2. `beforeToolCall` 的 `reason` 字段最终被谁消费？（模型——它出现在喂回模型的错误 toolResult 里）
3. `AfterToolCallResult` 的合并语义是什么？如果只想给 content 追加一条，应该怎么做？（做不到——只能整体替换；先读原 content、追加后整体返回）
4. `CustomAgentMessages` 的声明合并发生在编译期还是运行时？自定义消息默认会被 `convertToLlm` 怎么处理？
5. `AgentState` 和 `AgentContext` 各自的持有者是谁？消息从哪个流向哪个？（state → context 是快照；context 的产出经 message_end 事件回流 state）
6. `content` 和 `details` 的区别？`terminate` 在什么条件下才生效？

读到这里，agent 运行时的"词汇表"你已经全部掌握了。下一篇进入 pi 最有特色的机制之一：上下文压缩。
