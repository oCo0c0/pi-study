# 精读 05：sdk.ts —— 组合根：产品层如何装配 Agent 运行时

> 对应源码：`packages/coding-agent/src/core/sdk.ts`（411 行）
> 定位：pi CLI 的**组合根（composition root）**——把模型运行时、设置、会话管理、资源加载、内置工具、扩展系统全部装配起来，最后 `new Agent(...)` 并包进 `AgentSession`。**这是连接 agent 包（运行时）和 coding-agent（产品层）的枢纽文件**，也是 `packages/coding-agent/examples/sdk/` 那 13 个示例的入口。
> 前置：[02-agent.md](02-agent.md)（Agent 构造选项）、[03-agent-types.md](03-agent-types.md)（各配置项含义）。

## 一个问题串起全文件

**"用户敲下 `pi` 回车后，第一个 Agent 实例是怎么被造出来的？"** 这个文件就是答案。整条装配线：

```
解析路径（cwd / agentDir）
  → ModelRuntime（provider + 凭证）      ─┐
  → SettingsManager（全局+项目设置合并）   ├─ 基础设施
  → SessionManager（会话 JSONL）          │
  → DefaultResourceLoader（扩展/skills…） ─┘
  → 模型解析三级链（显式 → 会话恢复 → 设置默认）
  → thinkingLevel 解析链（同类优先级）
  → 工具名单解析（allowlist/denylist/默认）
  → new Agent({...})   ← ★ 注入全部策略
  → 恢复历史消息（如有）
  → new AgentSession({...})  ← 包装成产品级会话
```

## 1. 文件头：全局默认注入（第 34~37 行）

```ts
// 为"0.81 之前的老扩展"保留的兜底：那些扩展自己 new Agent 或直接跑
// agentLoop 却没传 streamFn。agent 核心保持 provider 无关、自己不 import
// pi-ai/compat，所以由（也只由）宿主在这里注册默认流函数。
setDefaultStreamFn(streamSimple);
```

这是一个"谁污染谁负责"的边界决策：agent 包坚持不依赖任何 provider 实现，默认流函数由第一个加载的宿主（这里就是 pi CLI）注册。

## 2. 装配选项 `CreateAgentSessionOptions`（第 39~88 行）

全部可选——不传就用默认值（最小示例 `createAgentSession()` 一行就能跑）。按用途分四组：

```ts
export interface CreateAgentSessionOptions {
	// ── 路径组 ──
	cwd?: string;        // 项目目录，默认 process.cwd()（决定 .pi/ 项目资源 discovery）
	agentDir?: string;   // 全局配置目录，默认 ~/.pi/agent

	// ── 模型组 ──
	modelRuntime?: ModelRuntime;   // 自带模型运行时（默认用 agentDir 的 auth.json/models.json）
	model?: Model<any>;            // 显式指定模型
	thinkingLevel?: ThinkingLevel; // 显式指定推理等级
	scopedModels?: Array<{ model: Model<any>; thinkingLevel?: ThinkingLevel }>;
	                               // Ctrl+P 可循环切换的模型圈

	// ── 工具组 ──
	noTools?: "all" | "builtin";  // 无显式 allowlist 时的默认抑制模式
	tools?: string[];             // allowlist：只启用列出的
	excludeTools?: string[];      // denylist：在 allowlist 之后应用
	customTools?: ToolDefinition[]; // 额外注册的自定义工具

	// ── 基础设施组（依赖注入点）──
	resourceLoader?: ResourceLoader;   // 默认 DefaultResourceLoader + reload()
	sessionManager?: SessionManager;   // 默认 SessionManager.create(cwd)
	settingsManager?: SettingsManager; // 默认 SettingsManager.create(cwd, agentDir)
	sessionStartEvent?: SessionStartEvent; // 扩展启动时的会话元数据
}
```

**Java 视角**：这就是一个 Builder 的参数对象，且每个依赖都可以从外面注入（方便测试——examples 里用 `SessionManager.inMemory()` 做无副作用会话）。

## 3. 基础设施装配（第 173~189 行）

```ts
export async function createAgentSession(options = {}): Promise<CreateAgentSessionResult> {
	const cwd = resolvePath(options.cwd ?? options.sessionManager?.getCwd() ?? process.cwd());
	const agentDir = options.agentDir ? resolvePath(options.agentDir) : getDefaultAgentDir();

	// 显式传了 agentDir 才指定 auth/models 路径（没传则 ModelRuntime 用自己的默认）
	const authPath = options.agentDir ? join(agentDir, "auth.json") : undefined;
	const modelsPath = options.agentDir ? join(agentDir, "models.json") : undefined;
	const modelRuntime = options.modelRuntime ?? (await ModelRuntime.create({ authPath, modelsPath }));

	const settingsManager = options.settingsManager ?? SettingsManager.create(cwd, agentDir);
	const sessionManager = options.sessionManager ?? SessionManager.create(cwd, getDefaultSessionDir(cwd, agentDir));

	if (!resourceLoader) {
		resourceLoader = new DefaultResourceLoader({ cwd, agentDir, settingsManager });
		await resourceLoader.reload();      // 真正去磁盘扫描扩展/skills/模板/主题/上下文文件
		time("resourceLoader.reload");      // 启动性能打点（timings.ts）
	}
	// 注意 ?? 的用法：每个依赖都是"传了就用你的，没传我就造一个"。
	// 这是整个函数的基本节奏——依赖注入 + 默认实现。
```

## 4. 会话恢复检查与模型解析三级链（第 191~227 行）

```ts
	// 从会话文件重建上下文（buildSessionContext 就是"压缩+分支"投影后的消息列表）
	const existingSession = sessionManager.buildSessionContext();
	const hasExistingSession = existingSession.messages.length > 0;
	const hasThinkingEntry = sessionManager.getBranch().some((e) => e.type === "thinking_level_change");

	let model = options.model;
	let modelFallbackMessage: string | undefined;

	// 第一级：显式指定（options.model）——什么都不用做

	// 第二级：会话里保存过的模型（恢复会话场景）
	if (!model && hasExistingSession && existingSession.model) {
		const restoredModel = modelRuntime.getModel(existingSession.model.provider, existingSession.model.modelId);
		// 恢复成功还要确认凭证可用（hasConfiguredAuth）——
		// 否则模型在目录里存在但调不通
		if (restoredModel && modelRuntime.hasConfiguredAuth(restoredModel.provider)) {
			model = restoredModel;
		}
		if (!model) {
			// 恢复失败要告知用户（比如换了台没登录的机器）
			modelFallbackMessage = `Could not restore model ${existingSession.model.provider}/${existingSession.model.modelId}`;
		}
	}

	// 第三级：findInitialModel（设置里的默认 provider/model → 可用 provider 的默认）
	if (!model) {
		const result = await findInitialModel({
			scopedModels: [],
			isContinuing: hasExistingSession,
			defaultProvider: settingsManager.getDefaultProvider(),
			defaultModelId: settingsManager.getDefaultModel(),
			defaultThinkingLevel: settingsManager.getDefaultThinkingLevel(),
			modelThinkingLevels: settingsManager.getAllModelThinkingLevels(),
			modelRuntime,
		});
		model = result.model;
		if (!model) {
			// 一个可用模型都没有 → 提示用户登录/配置（auth-guidance 的引导文案）
			modelFallbackMessage = formatNoModelsAvailableMessage();
		} else if (modelFallbackMessage) {
			modelFallbackMessage += `. Using ${model.provider}/${model.id}`;  // 拼成完整告知
		}
	}
```

**三级链 + 全程兜底文案**——任何一级失败都不抛异常，最终一定给出一个可用的模型或者一条清晰的引导消息。产品级代码对"启动路径"的容错要求远高于库代码。

## 5. thinkingLevel 解析链（第 229~254 行）

```ts
	let thinkingLevel = options.thinkingLevel;

	// ① 会话里记录过的（且会话有 thinking_level_change entry）
	if (thinkingLevel === undefined && hasExistingSession) {
		thinkingLevel = hasThinkingEntry
			? existingSession.thinkingLevel
			: settingsManager.getDefaultThinkingLevel() ?? DEFAULT_THINKING_LEVEL;
	}
	// ② 按模型的独立覆盖（settings 里可为某个具体模型配默认等级）
	if (thinkingLevel === undefined && model) {
		const perModel = settingsManager.getModelThinkingLevel(model.provider, model.id);
		if (perModel) thinkingLevel = perModel;
	}
	// ③ 全局默认
	if (thinkingLevel === undefined) {
		thinkingLevel = settingsManager.getDefaultThinkingLevel() ?? DEFAULT_THINKING_LEVEL;
	}
	// ④ 最后 clamp 到模型实际支持的范围（比如模型只到 high，请求 max 会被压回 high）
	if (!model) {
		thinkingLevel = "off";
	} else {
		thinkingLevel = clampThinkingLevel(model, thinkingLevel) as ThinkingLevel;
	}
```

## 6. 工具名单解析（第 256~263 行）

```ts
	const defaultActiveToolNames: ToolName[] = ["read", "bash", "edit", "write"]; // 默认四件套
	const configuredDefaultToolNames = settingsManager.getDefaultTools();          // 设置里配的默认
	const allowedToolNames = options.tools ?? (options.noTools === "all" ? [] : undefined);
	const excludedToolNames = options.excludeTools;
	const excludedToolNameSet = excludedToolNames ? new Set(excludedToolNames) : undefined;
	// 解析顺序：显式 allowlist > noTools 抑制 > 设置默认 > 内置默认，
	// 最后再过一遍 denylist 过滤
	const initialActiveToolNames = (
		options.tools ?? (options.noTools ? [] : (configuredDefaultToolNames ?? defaultActiveToolNames))
	).filter((name) => !excludedToolNameSet?.has(name));
```

## 7. `convertToLlm` 的图片过滤包装（第 267~302 行）

```ts
	// 包装 messages.ts 的 convertToLlm：如果开启了 blockImages 设置就过滤图片。
	// 注释写明这是 defense-in-depth（纵深防御）——上游已经尽量不放图片进来了，
	// 这里在"发往 LLM 的最后一关"再拦一次。
	const convertToLlmWithBlockImages = (messages: AgentMessage[]): Message[] => {
		const converted = convertToLlm(messages);
		// ★ 每次调用都【动态】读设置——会话中途改设置立即生效，
		//   而不是启动时读一次就固化
		if (!settingsManager.getBlockImages()) return converted;

		return converted.map((msg) => {
			if (msg.role === "user" || msg.role === "toolResult") {
				const content = msg.content;
				if (Array.isArray(content)) {
					const hasImages = content.some((c) => c.type === "image");
					if (hasImages) {
						// 图片 → 文本占位 "Image reading is disabled."
						// 连续的占位文本再去重（连续两张图只留一条提示）
						const filteredContent = content
							.map((c) => c.type === "image" ? { type: "text", text: "Image reading is disabled." } : c)
							.filter(/* 相邻重复占位去重 */);
						return { ...msg, content: filteredContent };
					}
				}
			}
			return msg;
		});
	};
```

## 8. 扩展运行时的"前向引用"（第 304 行）——解决鸡生蛋问题

```ts
	const extensionRunnerRef: { current?: ExtensionRunner } = {};
```

**装配顺序的矛盾**：`new Agent(...)` 需要在 streamFn 里调用扩展事件（`before_provider_headers` 等），但 `ExtensionRunner` 是在 `AgentSession` 构造时才创建并绑定扩展的——**Agent 建立时扩展系统还不存在**。

解法就是这一个可变引用盒子：Agent 的闭包捕获 `ref` 对象（不是值），之后 AgentSession 把 runner 塞进 `ref.current`。闭包里每次用时判空：

```ts
	const runner = extensionRunnerRef.current;   // 可能为 undefined（扩展还没加载）
	if (!runner?.hasHandlers("before_provider_request")) return payload;
```

Java 视角：`AtomicReference<ExtensionRunner>` 或 `Supplier<Optional<ExtensionRunner>>`，但因为单线程 + 事件循环，普通对象就够了。

## 9. `new Agent({...})`（第 306~372 行）——全部注入点逐个看

```ts
	agent = new Agent({
		initialState: {
			systemPrompt: "",          // 先空着！系统提示词由 AgentSession 组装后注入
			                           // （buildSystemPrompt 要知道最终的工具列表/skills）
			model,
			thinkingLevel,
			tools: [],                 // 工具也由 AgentSession 注册（含扩展工具）
		},
		convertToLlm: convertToLlmWithBlockImages,   // 上面第 7 节的包装版

		// ★ streamFn 是最厚的包装层（洋葱式）：
		streamFn: async (model, context, options) => {
			// 每次调用都动态读设置（热更新友好）
			const providerRetrySettings = settingsManager.getProviderRetrySettings();
			const httpIdleTimeoutMs = settingsManager.getHttpIdleTimeoutMs();
			// SDK 把 timeout=0 当"0 毫秒超时"而不是"无超时"——
			// 所以"禁用超时"要用 int32 最大值表达
			const effectiveTimeoutMs = httpIdleTimeoutMs === 0 ? 2147483647 : httpIdleTimeoutMs;
			const timeoutMs = options?.timeoutMs ?? providerRetrySettings.timeoutMs ?? effectiveTimeoutMs;

			const headerRunner = extensionRunnerRef.current;
			return modelRuntime.streamSimple(model, context, {
				...options,
				timeoutMs,
				maxRetries: options?.maxRetries ?? providerRetrySettings.maxRetries,
				maxRetryDelayMs: options?.maxRetryDelayMs ?? providerRetrySettings.maxRetryDelayMs,
				// 请求头改写链：先合并 attribution 头（归因标识），
				// 再给扩展一个 before_provider_headers 事件的机会
				transformHeaders: async (requestHeaders) => {
					const headers = mergeProviderAttributionHeaders(model, settingsManager, options?.sessionId, requestHeaders);
					return headerRunner?.hasHandlers("before_provider_headers")
						? headerRunner.emitBeforeProviderHeaders(headers ?? {})
						: (headers ?? {});
				},
			});
		},

		// 请求载荷钩子：扩展的 before_provider_request 事件（改写请求体）
		onPayload: async (payload, _model) => {
			const runner = extensionRunnerRef.current;
			if (!runner?.hasHandlers("before_provider_request")) return payload;
			return runner.emitBeforeProviderRequest(payload);
		},
		// 响应钩子：扩展的 after_provider_response 事件（只观察）
		onResponse: async (response, _model) => {
			const runner = extensionRunnerRef.current;
			if (!runner?.hasHandlers("after_provider_response")) return;
			await runner.emit({ type: "after_provider_response", status: response.status, headers: response.headers });
		},

		sessionId: sessionManager.getSessionId(),   // 给 provider 做 prompt 缓存亲和
		// 每次 LLM 调用前的上下文变换 → 扩展的 context 事件
		transformContext: async (messages) => {
			const runner = extensionRunnerRef.current;
			if (!runner) return messages;
			return runner.emitContext(messages);
		},
		steeringMode: settingsManager.getSteeringMode(),     // "one-at-a-time" | "all"
		followUpMode: settingsManager.getFollowUpMode(),
		transport: settingsManager.getTransport(),           // sse | websocket | auto
		thinkingBudgets: settingsManager.getThinkingBudgets(),
		maxRetryDelayMs: settingsManager.getProviderRetrySettings().maxRetryDelayMs,
	});
```

**包装模式总结**（这个 streamFn 值得抄）：

| 层 | 做什么 | 为什么在这里 |
| --- | --- | --- |
| 超时/重试 | 动态读设置并应用 | 支持运行中改配置 |
| transformHeaders | 归因头 + 扩展事件 | 扩展可以按请求改头（比如注入代理凭证） |
| onPayload / onResponse | 扩展观察/改写请求体 | 提供商请求级的扩展点 |

## 10. 历史恢复与最终包装（第 374~410 行）

```ts
	// 恢复历史：把重建的上下文消息直接放进 agent 状态
	if (hasExistingSession) {
		agent.state.messages = existingSession.messages;
		if (!hasThinkingEntry) {
			// 老会话没记录过 thinking level → 补一条 entry（格式向前兼容）
			sessionManager.appendThinkingLevelChange(thinkingLevel);
		}
	} else {
		// 新会话：把初始模型和 thinking level 记进会话文件，
		// 这样 --continue 恢复时能还原
		if (model) sessionManager.appendModelChange(model.provider, model.id);
		sessionManager.appendThinkingLevelChange(thinkingLevel);
	}

	// 最终包装：AgentSession 是产品级的会话门面
	// （steering/followUp 队列、自动压缩、系统提示词重建、工具注册表、扩展绑定都在它里面）
	const session = new AgentSession({
		agent,
		sessionManager,
		settingsManager,
		cwd,
		scopedModels: options.scopedModels,
		resourceLoader,
		customTools: options.customTools,
		modelRuntime,
		initialActiveToolNames,
		allowedToolNames,
		excludedToolNames,
		extensionRunnerRef,       // ★ ref 盒子传进去——AgentSession 创建 runner 后填 current
		sessionStartEvent: options.sessionStartEvent,
	});
	const extensionsResult = resourceLoader.getExtensions();

	return { session, extensionsResult, modelFallbackMessage };
```

## 分层回顾：三个类的分工

| 层 | 类 | 职责 | 所在包 |
| --- | --- | --- | --- |
| 循环 | `agentLoop` / `Agent` | 工具循环、事件流、队列、abort | agent |
| 装配 | `createAgentSession`（本文件） | 把一切依赖注入 Agent | coding-agent |
| 门面 | `AgentSession` | 会话编排：压缩触发、系统提示词重建、工具注册、扩展绑定 | coding-agent |

**systemPrompt 和 tools 为什么不在本文件注入、而留给 AgentSession？** 因为系统提示词的内容（工具列表、skills 列表）和工具注册表（内置 + 扩展工具）都依赖扩展系统的加载结果，而扩展绑定发生在 AgentSession——所以"谁拥有最终信息，谁负责填"。

## 自测

1. 模型解析的三级优先链是什么？恢复失败时用户体验如何保证？（fallback 文案一路拼接）
2. `extensionRunnerRef` 解决什么装配矛盾？为什么可以用普通对象而不用并发安全结构？（单线程事件循环）
3. streamFn 包装里为什么要"每次调用动态读设置"？举一个具体的用户场景。（用户在会话中改了重试设置）
4. `timeoutMs = 0` 为什么被翻译成 `2147483647`？（SDK 把 0 当 0ms 立即超时，不是"无超时"）
5. `agent.state.messages = existingSession.messages` 直接赋值安全吗？（setter 会拷贝顶层数组——见 02）
6. `sessionId` 传给 Agent 的作用是什么？（provider 的 prompt 缓存亲和：同一会话的请求容易命中缓存）

## 系列完结：接下来去哪

五篇导读覆盖了 agent 运行时的核心路径。建议的后续方向（配合 [学习路线.md](../学习路线.md)）：

1. **跑示例**：`packages/coding-agent/examples/sdk/01-minimal` → `13-session-runtime`，每个示例对应本文件的一组选项。
2. **读 AgentSession**：`packages/coding-agent/src/core/agent-session.ts`（3516 行）——本文件装配出的门面的完整实现，重点看 `_rebuildSystemPrompt()` 和自动压缩三触发。
3. **读工具实现**：`packages/coding-agent/src/core/tools/read.ts`（最简单的范本）→ `bash.ts`（进程管理 + 流式输出 + 杀进程树）。
4. **写一个扩展**：照着 `examples/extensions/` 里的例子写个 hello world，把 03 里学的事件钩子用起来。
