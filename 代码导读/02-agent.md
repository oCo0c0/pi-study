# 精读 02：agent.ts —— 有状态的 Agent 类与事件屏障

> 对应源码：`packages/agent/src/agent.ts`（592 行）
> 定位：`agent-loop.ts` 是"裸循环"，只管跑和发事件，**不管状态、不管持久化**。`Agent` 类在它外面套了一层：持有对话状态、管理 steering/follow-up 队列、提供 abort、并且实现了**事件屏障**（event barrier）——生产环境都用这一层。
> 前置：先读完 [01-agent-loop.md](01-agent-loop.md)。

## 为什么需要这个类？（先想清楚这个问题）

裸循环有两个问题：

1. **裸流是"观察性"的**：你只能订阅事件看它跑，但"跑完了"不等于"所有监听者都处理完了"。如果你在 `agent_end` 事件里把会话写到磁盘，裸循环不知道也不等你——`agentLoop()` 返回的流结束了，你的磁盘可能还没写完。
2. **状态散落**：对话历史、当前模型、待执行的工具调用，循环不管这些，谁用谁自己拼。

`Agent` 类解决这两件事：**事件屏障**（监听者全部完成，`prompt()` 的 Promise 才 resolve）+ **集中状态管理**。

## 文件地图

| 行号 | 内容 | 一句话职责 |
| --- | --- | --- |
| 33 | `defaultConvertToLlm` | 默认的边界转换：只保留三种标准角色 |
| 39/48 | `EMPTY_USAGE` / `DEFAULT_MODEL` | 零值常量（空对象模式） |
| 61~95 | `MutableAgentState` + 工厂 | 可变状态，tools/messages 的 setter 做防御性拷贝 |
| 98~123 | `AgentOptions` | 构造 Agent 的全部选项 |
| 125~159 | `PendingMessageQueue` | steering/follow-up 消息队列 |
| 161~165 | `ActiveRun` | 一次运行的生命周期句柄 |
| 173~592 | `Agent` 类 | 主体 |

---

## 1. 默认转换与零值常量（第 33~66 行）

```ts
// 默认的边界转换：只保留 user / assistant / toolResult 三种标准消息。
// 自定义消息（bashExecution、compactionSummary 等）默认会被【丢弃】！
// 所以宿主应用（比如 coding-agent）必须提供自己的 convertToLlm，
// 决定自定义消息怎么呈现给模型（通常是包装成 user 消息）。
function defaultConvertToLlm(messages: AgentMessage[]): Message[] {
	return messages.filter(
		(message) => message.role === "user" || message.role === "assistant" || message.role === "toolResult",
	);
}

// 零值 usage——所有字段为 0 的占位对象
const EMPTY_USAGE = { input: 0, output: 0, /* ... */ totalTokens: 0, cost: { /* ... */ total: 0 } };

// 零值模型——"还没设置模型"时的占位。Java 视角：空对象模式（Null Object Pattern）
const DEFAULT_MODEL = {
	id: "unknown",
	/* ... */
} satisfies Model<any>;
// satisfies 关键字：只检查"这个对象符合 Model 接口"，但不把字面量类型放宽成 Model。
// 纯编译期检查，运行时零开销。
```

```ts
// 可变状态类型：从 AgentState 里 Omit（减去）四个"运行时才有意义"的字段，
// 再用可变类型补回来。Java 视角：定义"持久配置"和"运行时状态"两组字段的类型手法。
type MutableAgentState = Omit<AgentState, "isStreaming" | "streamingMessage" | "pendingToolCalls" | "errorMessage"> & {
	isStreaming: boolean;
	streamingMessage?: AgentMessage;
	pendingToolCalls: Set<string>;    // 正在执行的工具调用 id 集合
	errorMessage?: string;
};
```

## 2. 状态工厂：防御性拷贝（第 68~95 行）

```ts
function createMutableAgentState(initialState?): MutableAgentState {
	// slice() = 浅拷贝顶层数组。传入的数组从此和内部状态脱钩。
	let tools = initialState?.tools?.slice() ?? [];
	let messages = initialState?.messages?.slice() ?? [];

	return {
		systemPrompt: initialState?.systemPrompt ?? "",
		model: initialState?.model ?? DEFAULT_MODEL,
		thinkingLevel: initialState?.thinkingLevel ?? "off",
		// 对象字面量里直接写 getter/setter——Java 的 Bean 属性写法：
		get tools() {
			return tools;
		},
		set tools(nextTools: AgentTool<any>[]) {
			tools = nextTools.slice();          // 赋值时也拷贝！外部换了数组不影响内部
		},
		get messages() {
			return messages;
		},
		set messages(nextMessages: AgentMessage[]) {
			messages = nextMessages.slice();
		},
		isStreaming: false,
		/* ... */
	};
}
```

**为什么处处拷贝？** JS 没有不可变集合，防"外部引用改了内部状态"只能靠拷贝。注意只拷贝**顶层**（数组本身），数组里的消息对象还是共享引用——这是有意的：消息被当成不可变值对待，改它的人自己负责拷贝。

## 3. `AgentOptions`（第 98~123 行）——装配清单

```ts
export interface AgentOptions {
	initialState?: Partial<Omit<AgentState, ...>>;  // Partial = 所有字段变可选
	convertToLlm?: (messages: AgentMessage[]) => Message[] | Promise<Message[]>;
	transformContext?: (messages: AgentMessage[], signal?) => Promise<AgentMessage[]>;
	streamFn: StreamFn;                    // 唯一必填：怎么调 LLM
	getApiKey?: (provider: string) => ...; // 动态解析 key（支持过期 token）
	onPayload?: ...; onResponse?: ...;     // 请求/响应观测钩子
	beforeToolCall?: ...;                  // 工具执行前拦截（权限门）
	afterToolCall?: ...;                   // 工具执行后改写
	shouldStopAfterTurn?: ...;             // 每轮结束后决定是否优雅停止（压缩触发点）
	prepareNextTurn?: ...;                 // 下一轮开始前替换 context/model/思考级别
	steeringMode?: QueueMode;              // 队列模式："one-at-a-time"（默认）| "all"
	followUpMode?: QueueMode;
	sessionId?: string;                    // 提供给 provider 做 prompt 缓存亲和
	thinkingBudgets?: ThinkingBudgets;     // 推理 token 预算
	transport?: Transport;                 // sse / websocket / auto
	maxRetryDelayMs?: number;              // 重试延迟上限
	toolExecution?: ToolExecutionMode;     // "parallel"（默认）| "sequential"
}
```

对照 `packages/coding-agent/src/core/sdk.ts` 的 `createAgentSession()` 看：那里就是按这个清单装配的——`convertToLlm` 传了自定义实现（处理自定义消息），`transformContext` 接了扩展事件，`streamFn` 委托给 ModelRuntime 并叠加扩展钩子。

## 4. `PendingMessageQueue`（第 125~159 行）——两种排空模式

```ts
class PendingMessageQueue {
	private messages: AgentMessage[] = [];
	public mode: QueueMode;    // "all" | "one-at-a-time"

	enqueue(message: AgentMessage): void {
		this.messages.push(message);
	}

	hasItems(): boolean {
		return this.messages.length > 0;
	}

	// "排空"——注意它不叫 take/pop：one-at-a-time 模式下一次只取最旧的一条
	drain(): AgentMessage[] {
		if (this.mode === "all") {
			const drained = this.messages.slice();   // 全取
			this.messages = [];
			return drained;
		}
		const first = this.messages[0];
		if (!first) {
			return [];
		}
		this.messages = this.messages.slice(1);      // 只取第一条
		return [first];
	}

	clear(): void {
		this.messages = [];
	}
}
```

`one-at-a-time`（默认）的语义：**每个注入点只送一条话进去**——用户连插三句，agent 跑完一个 turn 只听到第一句，处理完再听下一句。`all` 则是三句一起进上下文。这是"人在环中"的产品级细节。

## 5. `ActiveRun` 与 Agent 类字段（第 161~238 行）

```ts
// 一次运行的句柄：promise 是"全部跑完（含监听者）"的信号，
// abortController 是这次运行的取消源
type ActiveRun = {
	promise: Promise<void>;
	resolve: () => void;
	abortController: AbortController;
};
```

```ts
export class Agent {
	private _state: MutableAgentState;
	// 监听者集合。Set 保证不重复；迭代顺序 = 插入顺序 = await 顺序
	private readonly listeners = new Set<(event: AgentEvent, signal: AbortSignal) => Promise<void> | void>();
	private readonly steeringQueue: PendingMessageQueue;
	private readonly followUpQueue: PendingMessageQueue;
	// ...（convertToLlm / streamFunction / 各钩子都是 public 字段，构造时从 options 拷入，
	//     运行期可以直接改——比如 AgentSession 切换模型就是改 agent 的字段）
	private activeRun?: ActiveRun;

	constructor(options: AgentOptions) {
		// 兼容旧调用方可能不传 options：?? {} 兜底
		const runtimeOptions: Partial<AgentOptions> = options ?? {};
		this._state = createMutableAgentState(runtimeOptions.initialState);
		this.convertToLlm = runtimeOptions.convertToLlm ?? defaultConvertToLlm;
		this.streamFunction = runtimeOptions.streamFn ?? getDefaultStreamFn();
		// ...其余字段逐个拷入，队列模式默认 one-at-a-time，工具执行默认 parallel
		this.steeringQueue = new PendingMessageQueue(runtimeOptions.steeringMode ?? "one-at-a-time");
		this.followUpQueue = new PendingMessageQueue(runtimeOptions.followUpMode ?? "one-at-a-time");
		this.toolExecution = runtimeOptions.toolExecution ?? "parallel";
	}
}
```

## 6. 订阅、队列与控制 API（第 240~345 行）

```ts
	/**
	 * Subscribe to agent lifecycle events.
	 * Listener promises are awaited in subscription order and are included in
	 * the current run's settlement. ...
	 */
	subscribe(listener: (event: AgentEvent, signal: AbortSignal) => Promise<void> | void): () => void {
		this.listeners.add(listener);
		return () => this.listeners.delete(listener);   // 返回"取消订阅"函数（常见模式）
	}
	// 注释里的关键承诺：监听者的 Promise 会按订阅顺序被 await，
	// 并且【算进当前运行的结算】——这就是事件屏障的入口。

	/** 运行中插话：当前批工具执行完、下次调 LLM 前注入 */
	steer(message: AgentMessage): void {
		this.steeringQueue.enqueue(message);
	}

	/** 追发：等 agent 本来要停时再跑一轮 */
	followUp(message: AgentMessage): void {
		this.followUpQueue.enqueue(message);
	}

	/** 当前运行的取消信号（没有运行时是 undefined） */
	get signal(): AbortSignal | undefined {
		return this.activeRun?.abortController.signal;
	}

	/** 中止当前运行 */
	abort(): void {
		this.activeRun?.abortController.abort();
	}

	/**
	 * 等"当前运行 + 所有被 await 的监听者"全部结束。
	 * 在 agent_end 的监听者们 settle 之后才 resolve。
	 */
	waitForIdle(): Promise<void> {
		return this.activeRun?.promise ?? Promise.resolve();
	}
```

## 7. `prompt` / `continue`（第 347~388 行）

```ts
	// TS 的函数重载：上面两行是"对外签名"，最后一个才是实现。
	// Java 视角：方法重载，但 TS 的重载只是类型层面的声明，运行时只有一个函数。
	async prompt(message: AgentMessage | AgentMessage[]): Promise<void>;
	async prompt(input: string, images?: ImageContent[]): Promise<void>;
	async prompt(input: string | AgentMessage | AgentMessage[], images?: ImageContent[]): Promise<void> {
		if (this.activeRun) {
			// ★ 正在跑的时候不允许再 prompt！不是排队，是直接抛错——
			// 错误消息告诉你该用 steer() / followUp()。这是 API 层面对用户的教育。
			throw new Error(
				"Agent is already processing a prompt. Use steer() or followUp() to queue messages, or wait for completion.",
			);
		}
		const messages = this.normalizePromptInput(input, images);
		await this.runPromptMessages(messages);
	}
```

```ts
	async continue(): Promise<void> {
		if (this.activeRun) throw new Error("Agent is already processing. ...");

		const lastMessage = this._state.messages[this._state.messages.length - 1];
		if (!lastMessage) throw new Error("No messages to continue from");

		if (lastMessage.role === "assistant") {
			// 特殊分支：最后一条是 assistant（上轮正常结束了）但队列里还有话没说——
			// 把排队的 steering（优先）或 followUp 拿出来当新的 prompt 跑
			const queuedSteering = this.steeringQueue.drain();
			if (queuedSteering.length > 0) {
				// skipInitialSteeringPoll: true —— 见下面 createLoopConfig 的解释
				await this.runPromptMessages(queuedSteering, { skipInitialSteeringPoll: true });
				return;
			}
			const queuedFollowUps = this.followUpQueue.drain();
			if (queuedFollowUps.length > 0) {
				await this.runPromptMessages(queuedFollowUps);
				return;
			}
			// 既没有排队消息，最后一条又是 assistant：无法 continue（LLM 会拒绝）
			throw new Error("Cannot continue from message role: assistant");
		}

		await this.runContinuation();
	}
```

## 8. 私有装配：快照与循环配置（第 390~484 行）

```ts
	private normalizePromptInput(input, images?): AgentMessage[] {
		if (Array.isArray(input)) return input;
		if (typeof input !== "string") return [input];
		// 字符串 → 标准用户消息（文本 + 可选图片）
		const content: Array<TextContent | ImageContent> = [{ type: "text", text: input }];
		if (images && images.length > 0) content.push(...images);
		return [{ role: "user", content, timestamp: Date.now() }];
	}

	private async runPromptMessages(messages, options = {}): Promise<void> {
		await this.runWithLifecycle(async (signal) => {
			await runAgentLoop(
				messages,
				this.createContextSnapshot(),       // 快照！
				this.createLoopConfig(options),     // 把自己编译成循环配置
				(event) => this.processEvents(event),  // ★ 所有事件进屏障
				signal,
				this.streamFunction,
			);
		});
	}

	// 给循环的是【快照】，不是内部状态的引用：
	// messages/tools 都 slice 拷贝一份。循环改它自己的副本，
	// 最终消息通过 message_end 事件【回流】到 _state.messages。
	// ——这就是"双数组"设计：循环内用快照数组，状态数组靠事件Reducer同步。
	private createContextSnapshot(): AgentContext {
		return {
			systemPrompt: this._state.systemPrompt,
			messages: this._state.messages.slice(),
			tools: this._state.tools.slice(),
		};
	}
```

```ts
	private createLoopConfig(options: { skipInitialSteeringPoll?: boolean } = {}): AgentLoopConfig {
		// 闭包标志位：让 runLoop 开头的第一次 steering 轮询返回空。
		// 场景：continue() 已经把队列里的消息 drain 出来当 prompt 传进去了，
		// 如果不跳过首次轮询，runLoop 第 168 行又会 poll 一次——
		// 好在那时队列已空。真正的问题是 runPromptMessages 传的是 steering 消息本身，
		// 再 poll 出一条就会【重复注入】。用闭包标志精确跳过那一次。
		let skipInitialSteeringPoll = options.skipInitialSteeringPoll === true;
		return {
			model: this._state.model,
			reasoning: this._state.thinkingLevel === "off" ? undefined : this._state.thinkingLevel,
			// ...其他字段逐个转发...

			// 队列通过两个回调暴露给循环——循环完全不认识 PendingMessageQueue，
			// 它只知道"问我要 steering / follow-up 消息"。依赖倒置。
			getSteeringMessages: async () => {
				if (skipInitialSteeringPoll) {
					skipInitialSteeringPoll = false;
					return [];
				}
				return this.steeringQueue.drain();
			},
			getFollowUpMessages: async () => this.followUpQueue.drain(),
		};
	}
```

## 9. 生命周期外壳 `runWithLifecycle`（第 486~509 行）

```ts
	private async runWithLifecycle(executor: (signal: AbortSignal) => Promise<void>): Promise<void> {
		if (this.activeRun) {
			throw new Error("Agent is already processing.");
		}

		// 三件套：取消源 + 手动 resolve 的 Promise。
		// new Promise<void>((resolve) => { resolvePromise = resolve })
		// 是"把 resolve 函数从构造器里拿出来"的常用手法——
		// Java 里类似自己 new 一个 CompletableFuture 存起来，之后手动 complete。
		const abortController = new AbortController();
		let resolvePromise = () => {};
		const promise = new Promise<void>((resolve) => {
			resolvePromise = resolve;
		});
		this.activeRun = { promise, resolve: resolvePromise, abortController };

		// 运行时状态复位
		this._state.isStreaming = true;
		this._state.streamingMessage = undefined;
		this._state.errorMessage = undefined;

		try {
			await executor(abortController.signal);   // 真正的循环在这里跑
		} catch (error) {
			// 循环漏出来的异常也走"合成事件序列"兜底（见下）
			await this.handleRunFailure(error, abortController.signal.aborted);
		} finally {
			this.finishRun();   // ★ 无论成败：清运行时状态 + resolve promise
		}
	}
```

**屏障链路推导**（这是本文件最重要的推理，跟着走一遍）：

```
prompt() 被 await
 → runWithLifecycle：await executor
   → runAgentLoop → runLoop：每发一个事件都 await emit
     → emit 就是 processEvents：先更新状态，再 for 顺序 await 每个监听者
       → 包括 agent_end 事件的监听者（比如"把会话写盘"）
 → executor 返回（此时 agent_end 监听者已经全部完成）
 → finally：finishRun() → activeRun.resolve()
 → prompt() 的 await 才返回
```

所以：**`await agent.prompt("...")` 返回时，你可以确定磁盘上已经有完整的会话了**。裸 `agentLoop` 做不到这一点——这就是 Agent 类存在的理由。

## 10. 失败兜底 `handleRunFailure`（第 511~527 行）

```ts
	private async handleRunFailure(error: unknown, aborted: boolean): Promise<void> {
		// 手工合成一条 stopReason 为 error/aborted 的 assistant 消息，
		// 然后把完整的生命周期事件序列（start → end → turn_end → agent_end）走一遍。
		// 目的：监听者（UI、持久化）无论循环怎么死的，看到的都是【格式完整的事件序列】，
		// 不需要为"循环半途崩了"写特殊处理。
		const failureMessage = {
			role: "assistant",
			content: [{ type: "text", text: "" }],
			api: this._state.model.api,
			provider: this._state.model.provider,
			model: this._state.model.id,
			usage: EMPTY_USAGE,
			stopReason: aborted ? "aborted" : "error",
			errorMessage: error instanceof Error ? error.message : String(error),
			timestamp: Date.now(),
		} satisfies AgentMessage;
		await this.processEvents({ type: "message_start", message: failureMessage });
		await this.processEvents({ type: "message_end", message: failureMessage });
		await this.processEvents({ type: "turn_end", message: failureMessage, toolResults: [] });
		await this.processEvents({ type: "agent_end", messages: [failureMessage] });
	}
```

## 11. 事件屏障 `processEvents`（第 529~591 行）

```ts
	private finishRun(): void {
		this._state.isStreaming = false;
		this._state.streamingMessage = undefined;
		this._state.pendingToolCalls = new Set<string>();
		this.activeRun?.resolve();    // ★ 唤醒 waitForIdle / prompt 的 await
		this.activeRun = undefined;
	}

	/**
	 * Reduce internal state for a loop event, then await listeners. ...
	 */
	private async processEvents(event: AgentEvent): Promise<void> {
		// 第一阶段：reducer——根据事件类型更新内部状态。
		// 每个分支只做最少的事，逻辑一目了然：
		switch (event.type) {
			case "message_start":
			case "message_update":
				// 生成中的消息放在独立的 streamingMessage 字段，
				// 不进 _state.messages
				this._state.streamingMessage = event.message;
				break;

			case "message_end":
				// 消息【定稿】才进历史——注意状态数组只在这里追加
				this._state.streamingMessage = undefined;
				this._state.messages.push(event.message);
				break;

			case "tool_execution_start": {
				// copy-on-write：new Set(...) 拷贝再改，而不是原地 add。
				// JS 惯例：让"状态变化"总是产生新对象，便于 UI 比较变化
				const pendingToolCalls = new Set(this._state.pendingToolCalls);
				pendingToolCalls.add(event.toolCallId);
				this._state.pendingToolCalls = pendingToolCalls;
				break;
			}

			case "tool_execution_end": {
				const pendingToolCalls = new Set(this._state.pendingToolCalls);
				pendingToolCalls.delete(event.toolCallId);
				this._state.pendingToolCalls = pendingToolCalls;
				break;
			}

			case "turn_end":
				if (event.message.role === "assistant" && event.message.errorMessage) {
					this._state.errorMessage = event.message.errorMessage;
				}
				break;

			case "agent_end":
				this._state.streamingMessage = undefined;
				break;
		}

		// 第二阶段：屏障——按订阅顺序 await 每个监听者。
		// 监听者还能拿到当前运行的 abort 信号（比如落盘时响应取消）
		const signal = this.activeRun?.abortController.signal;
		if (!signal) {
			throw new Error("Agent listener invoked outside active run");
		}
		for (const listener of this.listeners) {
			await listener(event, signal);
		}
	}
```

**两个数组再强调一遍**（初读最容易糊涂的地方）：

| 数组 | 谁在改 | 什么时候有内容 |
| --- | --- | --- |
| `currentContext.messages`（循环快照） | 循环自己（streamAssistantResponse push、工具结果 push） | 运行期间，流式消息甚至半成品就占着槽位 |
| `_state.messages`（Agent 状态） | 只有 processEvents 的 `message_end` 分支 | 每条消息定稿时追加 |

运行结束时两个数组内容一致，但写入路径完全不同——循环内高效可变，状态侧靠事件收敛。外部观察者（UI）只看 `_state`，永远看到一致的状态。

---

## 关键设计点自测

1. `await agent.prompt(x)` 返回时能保证什么？——保证所有监听者（含 agent_end 的落盘）已完成。链路是什么？
2. 运行中再调 `prompt()` 会怎样？——抛错，提示用 steer/followUp。为什么不做排队？——语义分开：prompt 是"新任务"，steer/followUp 是"运行中干预"。
3. `steer` 和 `followUp` 的注入时机分别是什么？（回 01 的 runLoop 找答案）
4. 为什么 `createContextSnapshot` 要 slice？消息通过什么路径回到 `_state.messages`？
5. `handleRunFailure` 为什么要合成完整事件序列，而不是把异常抛给调用者？
6. `pendingToolCalls` 为什么用 copy-on-write 而不是原地 `add`？

**下一步**：跑 `packages/coding-agent/examples/sdk/` 下的示例（01-minimal 起），对着 `Agent` 的构造参数看每个选项在示例里怎么传；然后读 `packages/coding-agent/src/core/sdk.ts`，看真实产品是怎么装配这个类的。
