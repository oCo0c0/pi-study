# 精读 01：agent-loop.ts —— Agent 循环的心脏

> 对应源码：`packages/agent/src/agent-loop.ts`（803 行）
> 定位：**整个 pi 系统最核心的一个文件**。"调用 LLM → 执行工具 → 再调 LLM"的循环、事件流、中断、并发工具执行，全部在这里。
> 建议读法：本文按源码顺序分段，每段先看注释版代码，再去源码对照原版。

## 文件地图（先看骨架）

| 行号 | 函数/类型 | 一句话职责 |
| --- | --- | --- |
| 26 | `AgentEventSink` | 事件"汇"的类型：一个接收事件的 async 函数 |
| 32 | `agentLoop` | 入口一：带新 prompt 启动循环，返回事件流（立即返回，不阻塞） |
| 65 | `agentLoopContinue` | 入口二：不加新消息、从现有上下文继续（重试用） |
| 96/121 | `runAgentLoop` / `runAgentLoopContinue` | 两个入口的实际执行体（会 await 跑完） |
| 156 | `runLoop` | **主循环本体**（双层循环），全文件最重要 |
| 279 | `streamAssistantResponse` | 调一次 LLM 并把流式事件转发出去 |
| 379 | `failToolCallsFromTruncatedMessage` | 输出被截断时，整批工具调用判失败 |
| 409 | `executeToolCalls` | 工具执行分派：决定顺序还是并行 |
| 431/487 | `executeToolCallsSequential` / `...Parallel` | 两种执行策略 |
| 607 | `prepareToolCall` | 工具预检：找工具 → 校验参数 → beforeToolCall 钩子 |
| 677 | `executePreparedToolCall` | 真正执行工具，接住抛出的错误 |
| 720 | `finalizeExecutedToolCall` | afterToolCall 钩子，允许改写结果 |
| 767-803 | 辅助函数 | 构造错误结果、构造 toolResult 消息等 |

**Java 视角总览**：这个文件就是一个"状态机 + 事件生产者"。类比 Java 的话，`runLoop` 相当于一个跑了很久的 `Callable`，但每做一步都通过 `emit()` 发一个事件出去（相当于往 `Sinks.Many` 里发信号），调用方靠消费这些事件来渲染 UI、落盘。

---

## 1. 文件头：核心设计声明（第 1~26 行）

```ts
/**
 * Agent loop that works with AgentMessage throughout.
 * Transforms to Message[] only at the LLM call boundary.
 */
// ↑ 文件级注释就是设计声明：整个循环内部统一使用 AgentMessage（agent 自己的富消息模型，
//   可以包含 bashExecution、compactionSummary 这类自定义消息），
//   只有在真正调用 LLM 的边界上（streamAssistantResponse 里）才转换成 pi-ai 的标准 Message[]。
//   这就是"内部表示丰富、对外协议收敛"的分层手法。

import {
	type AssistantMessage,      // LLM 的一条完整回复（含 usage、stopReason）
	type Context,               // 发给 LLM 的完整载荷：{ systemPrompt, messages, tools }
	EventStream,                // 事件流容器：后台往里 push，消费者 for-await 迭代
	type ToolResultMessage,     // 工具结果消息（role: "toolResult"）
	validateToolArguments,      // 用 TypeBox schema 校验工具参数
} from "@earendil-works/pi-ai";
import { getDefaultStreamFn } from "./stream-fn.ts";
import type {
	AgentContext,               // { systemPrompt, messages, tools }——循环的工作内存
	AgentEvent,                 // 判别联合：agent_start / turn_start / message_* / tool_execution_* ...
	AgentLoopConfig,            // 循环的全部可配置点（钩子、队列轮询函数等）
	AgentMessage,               // 内部富消息（标准消息 + 自定义消息的联合）
	AgentTool,                  // 工具定义：{ name, parameters, execute, executionMode... }
	AgentToolCall,              // 模型发出的工具调用请求 { id, name, arguments }
	AgentToolResult,            // 工具执行结果 { content, details, terminate?... }
	PrepareNextTurnContext,     // 传给 prepareNextTurn 钩子的上下文
	StreamFn,                   // 流函数签名：(model, context, options) => 事件流
} from "./types.ts";

// 事件汇：接收一个 AgentEvent 的函数，可以同步也可以异步（返回 Promise）。
// Java 视角：类似 Consumer<AgentEvent>，但允许异步——循环会 await 它。
export type AgentEventSink = (event: AgentEvent) => Promise<void> | void;
```

**为什么每个事件都要可 await？** 这是为了**顺序保证**：UI 渲染、会话落盘都是监听者，循环必须等它们处理完这个事件才继续下一步，否则会出现"落盘顺序和事件顺序不一致"的乱序问题。这是整个事件系统的基石，Java 里相当于每发一条消息都 `CompletableFuture.allOf(所有订阅者的处理).join()`。

---

## 2. 两个入口：`agentLoop` / `agentLoopContinue`（第 28~94 行）

```ts
/**
 * Start an agent loop with a new prompt message.
 * The prompt is added to the context and events are emitted for it.
 */
export function agentLoop(
	prompts: AgentMessage[],        // 本次要注入的新消息（通常是用户输入）
	context: AgentContext,          // 当前的系统提示词 + 消息历史 + 工具列表
	config: AgentLoopConfig,        // 所有钩子和配置
	signal: AbortSignal | undefined,// 取消信号（Java: 类似 Future.cancel 的协作式标志）
	streamFn: StreamFn,             // 怎么调 LLM（可替换！测试时用假模型，浏览器用代理）
): EventStream<AgentEvent, AgentMessage[]> {
	const stream = createAgentStream();

	// 关键手法：void 前缀 = "故意不 await"，让循环在后台跑。
	// 循环每产出一个事件就 push 进 stream；跑完后 stream.end(finalMessages) 收尾。
	// 所以本函数立即返回一个"活的"事件流对象，调用方立刻就能开始 for-await 消费。
	// Java 视角：相当于后台起一个任务往 Sinks.Many 里发信号，方法马上返回 Flux。
	void runAgentLoop(
		prompts,
		context,
		config,
		async (event) => {
			stream.push(event);       // 每个事件转发进流
		},
		signal,
		streamFn,
	).then((messages) => {
		stream.end(messages);         // 流结束，附带最终消息数组作为"结果值"
	});
	// 注意：这里没有 .catch。这是有意的契约——循环内部保证错误不抛出、
	// 而是编码成 stopReason 为 "error"/"aborted" 的消息走正常事件序列。
	// 前提是 config 里的 transformContext / convertToLlm 也遵守"不抛错"的约定。

	return stream;
}
```

```ts
/**
 * Continue an agent loop from the current context without adding a new message.
 * Used for retries - context already has user message or tool results.
 */
export function agentLoopContinue(
	context: AgentContext,
	config: AgentLoopConfig,
	signal: AbortSignal | undefined,
	streamFn: StreamFn,
): EventStream<AgentEvent, AgentMessage[]> {
	if (context.messages.length === 0) {
		throw new Error("Cannot continue: no messages in context");
	}
	// 最后一条消息不能是 assistant——因为 LLM API 要求"下一轮的开头"必须是
	// user 或 toolResult，否则 provider 直接拒绝请求。
	if (context.messages[context.messages.length - 1].role === "assistant") {
		throw new Error("Cannot continue from message role: assistant");
	}
	// ……后半段结构和 agentLoop 完全一样：创建流、后台跑、返回流
```

**两个入口的区别**：`agentLoop` = "用户说了新的一句话"；`agentLoopContinue` = "上一轮没跑完（比如中途出错了），原地重试"。后者注释里特意说明：它**没法在这里**验证"最后一条消息转成 LLM 消息后是不是 user/toolResult"——因为 `convertToLlm` 每个 turn 只会跑一次，所以只能靠约定。

---

## 3. 执行体与流工厂（第 96~151 行）

```ts
export async function runAgentLoop(
	prompts: AgentMessage[],
	context: AgentContext,
	config: AgentLoopConfig,
	emit: AgentEventSink,
	signal: AbortSignal | undefined,
	streamFn: StreamFn,
): Promise<AgentMessage[]> {
	const newMessages: AgentMessage[] = [...prompts];
	// newMessages 记录"本次运行新增的全部消息"（用户输入 + 模型回复 + 工具结果），
	// 最终作为流的 result 返回。注意循环里所有 push 都会同时写两处：
	// currentContext.messages（循环的工作内存）和 newMessages（本次运行的产出）。

	// 浅拷贝 context，不污染调用方的对象——把新消息接在历史后面
	const currentContext: AgentContext = {
		...context,
		messages: [...context.messages, ...prompts],
	};

	await emit({ type: "agent_start" });   // 事件树开始
	await emit({ type: "turn_start" });    // 第一个 turn 开始
	for (const prompt of prompts) {
		// 用户消息也要走一遍 message_start / message_end，
		// 这样监听者（UI、落盘）对"所有消息"的处理路径是统一的
		await emit({ type: "message_start", message: prompt });
		await emit({ type: "message_end", message: prompt });
	}

	// ?? 后备：如果没传 streamFn，用全局注册的默认流函数
	await runLoop(currentContext, newMessages, config, signal, emit, streamFn ?? getDefaultStreamFn());
	return newMessages;
}
```

```ts
function createAgentStream(): EventStream<AgentEvent, AgentMessage[]> {
	// EventStream 的两个参数：
	// 1) "什么事件算结束？" → agent_end
	// 2) "结束事件的哪个字段是结果值？" → agent_end.messages
	return new EventStream<AgentEvent, AgentMessage[]>(
		(event: AgentEvent) => event.type === "agent_end",
		(event: AgentEvent) => (event.type === "agent_end" ? event.messages : []),
	);
}
```

---

## 4. 主循环 `runLoop`（第 153~273 行）——全文件的核心，逐段精读

```ts
async function runLoop(
	initialContext: AgentContext,
	newMessages: AgentMessage[],
	initialConfig: AgentLoopConfig,
	signal: AbortSignal | undefined,
	emit: AgentEventSink,
	streamFunction: StreamFn,
): Promise<void> {
	let currentContext = initialContext;
	let config = initialConfig;          // config 用 let：prepareNextTurn 钩子可以整体替换它（比如换模型）
	let lastCompletedTurn: PrepareNextTurnContext | undefined;

	// 循环一启动就先轮询一次 steering 队列：
	// 用户在"按下回车"到"循环真正开跑"之间可能又打了字，这些话要被带上。
	let pendingMessages: AgentMessage[] = (await config.getSteeringMessages?.()) || [];
```

### 4.1 双层循环的结构

```ts
	// 外层循环：处理"追发"（follow-up）语义。
	// agent 本来要停了，但如果 follow-up 队列里有消息，就再跑一轮。
	while (true) {
		// 初始为 true，保证第一次一定进入内层循环（至少要调一次 LLM）
		let hasMoreToolCalls = true;

		// 内层循环：只要"还有工具要跑"或"还有要注入的消息"就继续
		while (hasMoreToolCalls || pendingMessages.length > 0) {
```

**为什么要两层？** 内层解决"模型还想用工具"（`toolUse` → 执行工具 → 再调模型）；外层解决"模型已经收尾了但用户又排了队"。两层用同一个 `pendingMessages` 变量衔接——follow-up 消息被塞进 pendingMessages 后 `continue`，就被内层当普通待注入消息处理了。

### 4.2 每轮开始：`prepareNextTurn` 钩子

```ts
			if (lastCompletedTurn) {
				// 第一轮没有 lastCompletedTurn，跳过；从第二轮起每轮开始前调用。
				// 典型用途：compaction（上下文压缩）在这里触发——
				// 钩子可以整体替换 context（换成压缩后的）、换模型、换 thinking 级别。
				const nextTurnSnapshot = await config.prepareNextTurn?.(lastCompletedTurn);
				if (nextTurnSnapshot) {
					// ?? 是"没给就用旧值"的兜底写法
					currentContext = nextTurnSnapshot.context ?? currentContext;
					config = {
						...config,                                    // 展开旧配置
						model: nextTurnSnapshot.model ?? config.model, // 覆盖 model（如果给了）
						reasoning:
							nextTurnSnapshot.thinkingLevel === undefined
								? config.reasoning
								: nextTurnSnapshot.thinkingLevel === "off"
									? undefined            // "off" 映射成 undefined（不传推理参数）
									: nextTurnSnapshot.thinkingLevel,
					};
				}
				// prepareNextTurn 可能很慢（比如压缩要调一次 LLM）。
				// 期间用户可能又 steer 了新消息，所以要再轮询一次。
				// 只在"之前那次轮询是空的"时才补轮询——否则 one-at-a-time 模式下
				// 两条消息会被拆进两个 turn，语义就变了。这是个很细的边界处理。
				if (pendingMessages.length === 0) {
					pendingMessages = (await config.getSteeringMessages?.()) || [];
				}
				await emit({ type: "turn_start" });
			}
```

### 4.3 注入待发消息 + 调用 LLM

```ts
			// 把排队中的 steering 消息注入上下文（在调 LLM 之前）
			if (pendingMessages.length > 0) {
				for (const message of pendingMessages) {
					await emit({ type: "message_start", message });
					await emit({ type: "message_end", message });
					currentContext.messages.push(message);   // 工作内存
					newMessages.push(message);               // 本次运行的产出
				}
				pendingMessages = [];
			}

			// ★ 调用 LLM，拿到这条完整的 assistant 回复（内部已经逐事件转发了）
			const message = await streamAssistantResponse(currentContext, config, signal, emit, streamFunction);
			newMessages.push(message);

			// 出错或被取消：结束本 turn 和整个 run。
			// 注意错误不是异常，而是 stopReason —— "错误是数据，不是控制流"。
			if (message.stopReason === "error" || message.stopReason === "aborted") {
				await emit({ type: "turn_end", message, toolResults: [] });
				await emit({ type: "agent_end", messages: newMessages });
				return;
			}
```

### 4.4 工具调用：执行或判失败

```ts
			// 从回复内容里过滤出工具调用（一条回复可能带多个工具调用）
			const toolCalls = message.content.filter((c) => c.type === "toolCall");

			const toolResults: ToolResultMessage[] = [];
			hasMoreToolCalls = false;   // 乐观置 false：没有工具调用就该退出内层了
			if (toolCalls.length > 0) {
				// stopReason === "length" 意味着输出被 token 上限截断。
				// 流式工具调用参数是用"尽力修复"的 JSON 解析器收尾的，
				// 截断的消息可能产出"能解析、能过校验、但内容不完整"的参数——
				// 这种调用执行起来是危险的（比如 bash 命令少了半个参数）。
				// 所以：一个都不执行，全部回错误结果，让模型重发完整版本。
				const executedToolBatch =
					message.stopReason === "length"
						? await failToolCallsFromTruncatedMessage(toolCalls, emit)
						: await executeToolCalls(currentContext, message, config, signal, emit);
				toolResults.push(...executedToolBatch.messages);
				// terminate = "整批工具都要求终止"（见 shouldTerminateToolBatch），
				// 此时不再继续循环
				hasMoreToolCalls = !executedToolBatch.terminate;

				// 工具结果进入上下文——下一轮 convertToLlm 时会一起发给模型
				for (const result of toolResults) {
					currentContext.messages.push(result);
					newMessages.push(result);
				}
			}

			await emit({ type: "turn_end", message, toolResults });

			// 记录已完成的 turn，下一轮的 prepareNextTurn 钩子要用
			lastCompletedTurn = {
				message,
				toolResults,
				context: currentContext,
				newMessages,
			};

			// 优雅停止钩子：compaction 的触发点。
			// AgentSession 配置的 shouldStopAfterTurn 会在上下文快满时返回 true，
			// 循环在这里退出，压缩在下一轮 prepareNextTurn 里完成。
			if (await config.shouldStopAfterTurn?.(lastCompletedTurn)) {
				await emit({ type: "agent_end", messages: newMessages });
				return;
			}

			// turn 结束后再轮询一次 steering（用户可能在工具执行期间插话）
			pendingMessages = (await config.getSteeringMessages?.()) || [];
		}
```

**steering 的语义就在这里**：注意 steering 消息是在**当前批工具全部执行完之后**、下一次调 LLM 之前注入的——它不会打断正在运行的工具，只会改变下一步的方向。

### 4.5 外层收尾：follow-up 检查

```ts
		// 走到这里说明内层退出了：模型不再调工具、也没有排队的 steering。
		// agent 本来要停——查一下 follow-up 队列。
		const followUpMessages = (await config.getFollowUpMessages?.()) || [];
		if (followUpMessages.length > 0) {
			// 有排队的追发消息：塞进 pendingMessages，continue 回内层再跑一轮
			pendingMessages = followUpMessages;
			continue;
		}

		// 真的没有消息了，退出
		break;
	}

	await emit({ type: "agent_end", messages: newMessages });
}
```

**自检**：steer 和 followUp 的区别现在能从代码上说出来吗？——steer 在内层循环的注入点被消费（运行中插话）；followUp 只在"内层已经退出、agent 即将停止"时被消费（追发一轮）。

---

## 5. `streamAssistantResponse`（第 275~370 行）——调一次 LLM

```ts
async function streamAssistantResponse(
	context: AgentContext,
	config: AgentLoopConfig,
	signal: AbortSignal | undefined,
	emit: AgentEventSink,
	streamFunction: StreamFn,
): Promise<AssistantMessage> {
	// 第一步：上下文变换（AgentMessage[] → AgentMessage[]）。
	// 扩展可以在这里删消息、改消息（emitContext 事件挂在这里）。
	let messages = context.messages;
	if (config.transformContext) {
		messages = await config.transformContext(messages, signal);
	}

	// 第二步：边界转换（AgentMessage[] → Message[]）。
	// 自定义消息在这里被转换/丢弃，产出 LLM 认识的标准消息。
	const llmMessages = await config.convertToLlm(messages);

	// 第三步：组装 LLM 请求载荷
	const llmContext: Context = {
		systemPrompt: context.systemPrompt,
		messages: llmMessages,
		tools: context.tools,
	};

	// 每次都解析 API key——因为可能是会过期的 OAuth token，不能缓存旧的
	const resolvedApiKey =
		(config.getApiKey ? await config.getApiKey(config.model.provider) : undefined) || config.apiKey;

	// 第四步：发起流式请求。返回的是"事件流"，不是"等待完成的回复"。
	const response = await streamFunction(config.model, llmContext, {
		...config,
		apiKey: resolvedApiKey,
		signal,
	});

	let partialMessage: AssistantMessage | null = null;  // 流式过程中的"半成品"消息
	let addedPartial = false;

	// for-await 消费事件流（Java: 类似 Flux.toIterable() 逐个拉取）
	for await (const event of response) {
		switch (event.type) {
			case "start":
				// 流开始：拿到初始 partial 消息。
				// 注意！它被直接 push 进 context.messages——也就是说
				// "正在生成中的消息"从一开始就占据了上下文里的一个槽位。
				partialMessage = event.partial;
				context.messages.push(partialMessage);
				addedPartial = true;
				// 发给监听者的是 {...partialMessage}——浅拷贝！
				// 防止监听者拿到引用后改动权威对象
				await emit({ type: "message_start", message: { ...partialMessage } });
				break;

			case "text_start":
			case "text_delta":          // 文本增量（打字机效果的来源）
			case "text_end":
			case "thinking_start":
			case "thinking_delta":      // 推理内容增量
			case "thinking_end":
			case "toolcall_start":
			case "toolcall_delta":      // 工具调用参数的流式增量
			case "toolcall_end":
				if (partialMessage) {
					partialMessage = event.partial;
					// 用更新后的 partial 覆盖上下文里的最后一个槽位（原地替换）
					context.messages[context.messages.length - 1] = partialMessage;
					await emit({
						type: "message_update",
						assistantMessageEvent: event,   // 原始事件也带给监听者（UI 用 delta 渲染）
						message: { ...partialMessage },
					});
				}
				break;

			case "done":
			case "error": {
				// 流结束（正常或出错都走这里）：从流里取最终完整消息
				const finalMessage = await response.result();
				if (addedPartial) {
					// 用最终版替换掉那个"半成品"槽位
					context.messages[context.messages.length - 1] = finalMessage;
				} else {
					// 流没发过 start 就结束了（极端情况）：直接放入完整消息
					context.messages.push(finalMessage);
				}
				if (!addedPartial) {
					await emit({ type: "message_start", message: { ...finalMessage } });
				}
				await emit({ type: "message_end", message: finalMessage });
				return finalMessage;
			}
		}
	}

	// 兜底：流没发 done/error 就枯竭了（协议异常），手动收尾
	const finalMessage = await response.result();
	// ……（与上面 done 分支相同的收尾逻辑）
	await emit({ type: "message_end", message: finalMessage });
	return finalMessage;
}
```

**本段的设计要点**：

1. **两段转换**（transformContext → convertToLlm）是"内部富模型"与"LLM 标准协议"的分界线，每次调用都走一遍。
2. **流式消息占据上下文槽位**：生成中的消息就已经在 `context.messages` 里了，每个 delta 原地覆盖，结束时替换成最终版。
3. **错误是数据**：`error` 事件和 `done` 走同一条收尾路径，最终消息带着 `stopReason: "error"` 返回——上层永远不用 try-catch LLM 调用。

---

## 6. 截断保护（第 372~404 行）

```ts
/**
 * Fail all tool calls from an assistant message that was truncated by the
 * output token limit. ...
 */
async function failToolCallsFromTruncatedMessage(
	toolCalls: AgentToolCall[],
	emit: AgentEventSink,
): Promise<ExecutedToolCallBatch> {
	const messages: ToolResultMessage[] = [];
	for (const toolCall of toolCalls) {
		// 走完整的"假装执行"事件序列（start → end），让 UI 能展示发生了什么
		await emit({
			type: "tool_execution_start",
			toolCallId: toolCall.id,
			toolName: toolCall.name,
			args: toolCall.arguments,
		});
		const finalized: FinalizedToolCallOutcome = {
			toolCall,
			result: createErrorToolResult(
				// 错误文案本身就是给模型看的提示词：
				// "响应触及输出上限，参数可能被截断，请用完整参数重新发起调用"
				`Tool call "${toolCall.name}" was not executed: the response hit the output token limit, so its arguments may be truncated. Re-issue the tool call with complete arguments.`,
			),
			isError: true,
		};
		await emitToolExecutionEnd(finalized, emit);
		const toolResultMessage = createToolResultMessage(finalized);
		await emitToolResultMessage(toolResultMessage, emit);
		messages.push(toolResultMessage);
	}
	// terminate: false —— 让循环继续，模型会看到错误并重发调用
	return { messages, terminate: false };
}
```

这是一个非常值得学的细节：**错误信息是写给模型看的**。它不是给人读的日志，而是引导模型自我修正的提示词。

---

## 7. 工具执行分派与两种策略（第 406~561 行）

```ts
async function executeToolCalls(
	currentContext: AgentContext,
	assistantMessage: AssistantMessage,
	config: AgentLoopConfig,
	signal: AbortSignal | undefined,
	emit: AgentEventSink,
): Promise<ExecutedToolCallBatch> {
	const toolCalls = assistantMessage.content.filter((c) => c.type === "toolCall");
	// 批内任何一个工具声明自己是 sequential，整批就退化为顺序执行
	const hasSequentialToolCall = toolCalls.some(
		(tc) => currentContext.tools?.find((t) => t.name === tc.name)?.executionMode === "sequential",
	);
	if (config.toolExecution === "sequential" || hasSequentialToolCall) {
		return executeToolCallsSequential(currentContext, assistantMessage, toolCalls, config, signal, emit);
	}
	return executeToolCallsParallel(currentContext, assistantMessage, toolCalls, config, signal, emit);
}
```

先看四个中间类型（第 563~587 行）——这是理解工具执行的钥匙：

```ts
// "已预检通过，待执行"：拿到了工具对象和校验后的参数
type PreparedToolCall = {
	kind: "prepared";
	toolCall: AgentToolCall;
	tool: AgentTool<any>;
	args: unknown;               // 参数已过 TypeBox 校验，但保持 unknown（谨慎）
};

// "预检阶段就失败了"：不用执行，直接拿这个错误结果
type ImmediateToolCallOutcome = {
	kind: "immediate";
	result: AgentToolResult<any>;
	isError: boolean;
};

// "已执行完成"（无论成败）
type ExecutedToolCallOutcome = {
	result: AgentToolResult<any>;
	isError: boolean;
};

// "已收尾"：工具调用 + 最终结果 + 是否错误，一一对应
type FinalizedToolCallOutcome = {
	toolCall: AgentToolCall;
	result: AgentToolResult<any>;
	isError: boolean;
};

// 并行版的关键：数组元素要么是"已完成的结果"，要么是"返回结果的异步闭包"
// （Java 视角：FinalizedToolCallOutcome | Supplier<CompletableFuture<...>>）
type FinalizedToolCallEntry = FinalizedToolCallOutcome | (() => Promise<FinalizedToolCallOutcome>);
```

**Java 视角**：`kind: "prepared" | "immediate"` 就是判别联合，等价于 `sealed interface ToolPreparation permits Prepared, Immediate` + 两个 record。

### 7.1 顺序执行（第 431~485 行）

```ts
async function executeToolCallsSequential(...): Promise<ExecutedToolCallBatch> {
	const finalizedCalls: FinalizedToolCallOutcome[] = [];
	const messages: ToolResultMessage[] = [];

	for (const toolCall of toolCalls) {
		await emit({ type: "tool_execution_start", ... });   // 1. 发 start 事件

		const preparation = await prepareToolCall(...);       // 2. 预检
		let finalized: FinalizedToolCallOutcome;
		if (preparation.kind === "immediate") {
			// 预检失败：直接用错误结果
			finalized = { toolCall, result: preparation.result, isError: preparation.isError };
		} else {
			// 预检通过：执行 + 收尾
			const executed = await executePreparedToolCall(preparation, signal, emit);
			finalized = await finalizeExecutedToolCall(currentContext, assistantMessage, preparation, executed, config, signal);
		}

		await emitToolExecutionEnd(finalized, emit);          // 3. 发 end 事件
		const toolResultMessage = createToolResultMessage(finalized);
		await emitToolResultMessage(toolResultMessage, emit); // 4. toolResult 作为消息发出
		finalizedCalls.push(finalized);
		messages.push(toolResultMessage);

		if (signal?.aborted) {
			break;   // 用户取消：不再执行后续工具
		}
	}

	return {
		messages,
		terminate: shouldTerminateToolBatch(finalizedCalls),
	};
}
```

### 7.2 并行执行（第 487~561 行）——本文件最精巧的段落

```ts
async function executeToolCallsParallel(...): Promise<ExecutedToolCallBatch> {
	const finalizedCalls: FinalizedToolCallEntry[] = [];

	// 第一遍循环：逐个"预检"（顺序进行！）
	for (const toolCall of toolCalls) {
		await emit({ type: "tool_execution_start", ... });

		const preparation = await prepareToolCall(...);
		if (preparation.kind === "immediate") {
			// 预检就失败的：立即定稿，不参与并发
			const finalized = { toolCall, result: preparation.result, isError: preparation.isError }
				satisfies FinalizedToolCallOutcome;
			// satisfies 关键字：只做类型检查、不改变量的推断类型（纯编译期）
			await emitToolExecutionEnd(finalized, emit);
			finalizedCalls.push(finalized);
			if (signal?.aborted) break;
			continue;
		}

		// 预检通过的：不是 push 结果，而是 push 一个"异步闭包"（还没执行！）
		finalizedCalls.push(async () => {
			if (signal?.aborted) {
				// 执行时才检查取消——因为排队等待期间用户可能按了停止
				const finalized = { toolCall, result: createErrorToolResult("Operation aborted"), isError: true }
					satisfies FinalizedToolCallOutcome;
				await emitToolExecutionEnd(finalized, emit);
				return finalized;
			}
			const executed = await executePreparedToolCall(preparation, signal, emit);
			const finalized = await finalizeExecutedToolCall(...);
			await emitToolExecutionEnd(finalized, emit);   // end 事件在完成时刻发出
			return finalized;
		});
		if (signal?.aborted) break;
	}

	// Promise.all = CompletableFuture.allOf：所有闭包并发执行，全部完成后继续。
	// 注意 orderedFinalizedCalls 的顺序 = finalizedCalls 数组的顺序 = assistant
	// 消息里的原始顺序——不是完成顺序！
	const orderedFinalizedCalls = await Promise.all(
		finalizedCalls.map((entry) => (typeof entry === "function" ? entry() : Promise.resolve(entry))),
	);

	const messages: ToolResultMessage[] = [];
	// 按原始顺序（而非完成顺序）发 toolResult 消息——保证持久化顺序稳定
	for (const finalized of orderedFinalizedCalls) {
		const toolResultMessage = createToolResultMessage(finalized);
		await emitToolResultMessage(toolResultMessage, emit);
		messages.push(toolResultMessage);
	}

	return { messages, terminate: shouldTerminateToolBatch(orderedFinalizedCalls) };
}
```

**这段的三个设计决策值得背下来**：

1. **预检顺序、执行并发、落盘按原序**。预检（找工具/校验/拦截钩子）逐个做，保证 `tool_execution_start` 事件顺序确定；真正耗时的执行并发跑；最终 toolResult 消息按模型给定的顺序落盘。事件顺序（开始序/完成序/持久化序）三种顺序各得其所。
2. **JS 的"并行"是 IO 并发，不是 CPU 并行**。单线程事件循环下，`Promise.all` 只是同时等多个 IO，不会有数据竞争——所以整个仓库不需要锁。
3. **取消检查放在闭包内部**而不是只在外层——排队等待期间用户 abort 了，轮到执行时才发现，也能正确产出"aborted"错误结果。

---

## 8. 预检 `prepareToolCall`（第 593~675 行）

```ts
// 旧版兼容垫片：工具可以声明 prepareArguments 对原始参数做预处理
function prepareToolCallArguments(tool: AgentTool<any>, toolCall: AgentToolCall): AgentToolCall {
	if (!tool.prepareArguments) return toolCall;
	const preparedArguments = tool.prepareArguments(toolCall.arguments);
	if (preparedArguments === toolCall.arguments) return toolCall;   // 没变就复用原对象
	return { ...toolCall, arguments: preparedArguments as Record<string, any> };
}

async function prepareToolCall(
	currentContext: AgentContext,
	assistantMessage: AssistantMessage,
	toolCall: AgentToolCall,
	config: AgentLoopConfig,
	signal: AbortSignal | undefined,
): Promise<PreparedToolCall | ImmediateToolCallOutcome> {
	// ① 按名字找工具。找不到 → 立即失败（模型可能幻觉出一个不存在的工具）
	const tool = currentContext.tools?.find((t) => t.name === toolCall.name);
	if (!tool) {
		return {
			kind: "immediate",
			result: createErrorToolResult(`Tool ${toolCall.name} not found`),
			isError: true,
		};
	}

	try {
		// ② 参数预处理垫片
		const preparedToolCall = prepareToolCallArguments(tool, toolCall);
		// ③ TypeBox schema 校验（含基本类型强制转换，比如把 "5" 转成 5）
		const validatedArgs = validateToolArguments(tool, preparedToolCall);
		// ④ beforeToolCall 钩子——权限门就挂在这里（coding-agent 的扩展 tool_call 事件）
		if (config.beforeToolCall) {
			const beforeResult = await config.beforeToolCall(
				{ assistantMessage, toolCall, args: validatedArgs, context: currentContext },
				signal,
			);
			if (signal?.aborted) {
				return { kind: "immediate", result: createErrorToolResult("Operation aborted"), isError: true };
			}
			if (beforeResult?.block) {
				// 钩子说 block：拒绝执行，原因会作为错误结果喂回模型
				const result = createErrorToolResult(beforeResult.reason || "Tool execution was blocked");
				if (beforeResult.terminate === true) {
					result.terminate = true;   // 钩子还可以要求终止整个 run
				}
				return { kind: "immediate", result, isError: true };
			}
		}
		if (signal?.aborted) {
			return { kind: "immediate", result: createErrorToolResult("Operation aborted"), isError: true };
		}
		// 全部通过：返回"已备好"
		return { kind: "prepared", toolCall, tool, args: validatedArgs };
	} catch (error) {
		// 校验器/钩子抛的任何异常也统一转成"立即失败"，绝不向上抛
		return {
			kind: "immediate",
			result: createErrorToolResult(error instanceof Error ? error.message : String(error)),
			isError: true,
		};
	}
}
```

**Java 视角**：这就是一个过滤器链（Filter Chain），但返回的是"通过"或"带原因的拒绝"（`Either<ImmediateFailure, Prepared>`），而不是抛异常。**为什么预检失败不抛异常？** 因为失败要变成喂回模型的错误结果，是正常业务路径，不是系统错误。

---

## 9. 执行 `executePreparedToolCall`（第 677~718 行）

```ts
async function executePreparedToolCall(
	prepared: PreparedToolCall,
	signal: AbortSignal | undefined,
	emit: AgentEventSink,
): Promise<ExecutedToolCallOutcome> {
	const updateEvents: Promise<void>[] = [];   // 收集所有 update 事件的发射 Promise
	let acceptingUpdates = true;                // 竞态防护开关

	try {
		// ★ 调用工具。注意参数顺序：toolCallId, 参数, 取消信号, 进度回调
		const result = await prepared.tool.execute(
			prepared.toolCall.id,
			prepared.args as never,
			signal,                          // 工具内部要自己响应取消（协作式）
			(partialResult) => {
				// 工具通过这个回调上报流式进度（比如 bash 的部分输出）。
				// 这里不 await，只把发射 Promise 攒起来
				if (!acceptingUpdates) return;   // 已定稿后迟到的进度直接丢弃
				updateEvents.push(
					Promise.resolve(
						emit({
							type: "tool_execution_update",
							toolCallId: prepared.toolCall.id,
							toolName: prepared.toolCall.name,
							args: prepared.toolCall.arguments,
							partialResult,
						}),
					),
				);
			},
		);
		acceptingUpdates = false;         // 关闸：之后迟到的 onUpdate 调用被忽略
		await Promise.all(updateEvents);  // 等所有 update 事件发完，保证顺序：update 都在 end 之前
		return { result, isError: false };
	} catch (error) {
		// ★ 核心约定：工具失败 = throw。循环在这里接住，转成错误结果喂回模型。
		// 工具作者永远不要自己把错误写进 content——那会绕过 isError 标记。
		acceptingUpdates = false;
		await Promise.all(updateEvents);
		return {
			result: createErrorToolResult(error instanceof Error ? error.message : String(error)),
			isError: true,
		};
	} finally {
		acceptingUpdates = false;         // 双保险
	}
}
```

---

## 10. 收尾 `finalizeExecutedToolCall` 与辅助函数（第 720~803 行）

```ts
async function finalizeExecutedToolCall(...): Promise<FinalizedToolCallOutcome> {
	let result = executed.result;
	let isError = executed.isError;

	// afterToolCall 钩子：可以按字段改写结果（content/details/usage/terminate/isError）
	// （coding-agent 的扩展 tool_result 事件挂在这里）
	if (config.afterToolCall) {
		try {
			const afterResult = await config.afterToolCall({ ... , context: currentContext }, signal);
			if (afterResult) {
				result = {
					...result,
					// ?? 表示"钩子没给这个字段就保留原值"——字段级覆盖
					content: afterResult.content ?? result.content,
					details: afterResult.details ?? result.details,
					usage: afterResult.usage ?? result.usage,
					terminate: afterResult.terminate ?? result.terminate,
				};
				isError = afterResult.isError ?? isError;
			}
		} catch (error) {
			// 钩子自己抛错也转为错误结果（钩子不可信原则）
			result = createErrorToolResult(error instanceof Error ? error.message : String(error));
			isError = true;
		}
	}

	return { toolCall: prepared.toolCall, result, isError };
}
```

```ts
// 统一构造错误工具结果：错误文本放在 content 里，会喂回给模型
function createErrorToolResult(message: string): AgentToolResult<any> {
	return {
		content: [{ type: "text", text: message }],
		details: {},   // details 是给 UI 的结构化数据，不进模型上下文
	};
}

// 把最终结果包装成 ToolResultMessage（会作为一条消息进入会话历史）
function createToolResultMessage(finalized: FinalizedToolCallOutcome): ToolResultMessage {
	return {
		role: "toolResult",
		toolCallId: finalized.toolCall.id,   // 靠 id 和工具调用配对——provider 靠这个关联
		toolName: finalized.toolCall.name,
		// JS 扩展（无类型）可能返回没有 content 的结果，兜底成空数组，
		// 防止 null 混进会话历史或 provider 请求
		content: finalized.result.content ?? [],
		details: finalized.result.details,
		usage: finalized.result.usage,       // 工具可以上报自己消耗的 token
		// 条件展开：只有真的有 addedToolNames 才加这个字段
		// （工具结果可以声明"加载了新工具"，实现 deferred tools）
		...(finalized.result.addedToolNames?.length ? { addedToolNames: finalized.result.addedToolNames } : {}),
		isError: finalized.isError,
		timestamp: Date.now(),
	};
}

// toolResult 消息也走 message_start/message_end 事件对——
// 在监听者眼里它和用户消息、assistant 消息一视同仁
async function emitToolResultMessage(toolResultMessage: ToolResultMessage, emit: AgentEventSink): Promise<void> {
	await emit({ type: "message_start", message: toolResultMessage });
	await emit({ type: "message_end", message: toolResultMessage });
}
```

**content 和 details 的区别**（面试级考点）：`content` 会发给 LLM（模型能看到）；`details` 只给 UI/日志用（比如 diff 的结构化数据、bash 的退出码），不占模型上下文。

---

## 十个关键设计点（读完自测）

1. **事件是 await 的**：每个事件的监听者处理完循环才继续——顺序保证的根基。
2. **错误是数据不是控制流**：LLM 错误编码为 `stopReason`，工具错误用 throw 但循环接住转结果。
3. **双层循环**：内层 = 工具循环 + steering 注入；外层 = follow-up 续跑。
4. **`length` 截断 → 整批工具判失败**：流式 JSON 修复可能产出"合法但残缺"的参数，宁可不执行。
5. **预检顺序 / 执行并发 / 落盘原序**：三种顺序各司其职。
6. **beforeToolCall 是权限门的挂载点**，afterToolCall 是结果改写点——两个钩子加上 prepare/execute 分离，构成工具执行的三段式。
7. **流式消息从一开始就占据上下文槽位**，delta 原地覆盖，结束时替换为最终版。
8. **发事件用浅拷贝**（`{...partial}`），防止监听者污染权威对象。
9. **每次调 LLM 前都重新解析 API key**——支持会过期的 OAuth token。
10. **取消是协作式的**：signal 层层传递，预检后、闭包内、工具内部各自检查。

**对照源码重走一遍**：从 `runLoop` 第 171 行的 `while (true)` 开始，用手指沿着代码走一遍"用户提问 → LLM 返回两个工具调用 → 并发执行 → 结果入上下文 → 再调 LLM → stop → agent_end"，走通了这个文件你就学会了。
