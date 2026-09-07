# 精读 04：compaction.ts —— 上下文压缩（长会话不失忆的机制）

> 对应源码：`packages/agent/src/harness/compaction/compaction.ts`（849 行）
> 定位：pi 最有特色的机制之一。**问题**：长会话必然超出模型的上下文窗口，直接丢历史 agent 会"失忆"；**方案**：把旧历史交给 LLM 压缩成结构化摘要，替换掉旧历史，但**原始历史不删除**（存在会话树里，随时可回看/回退）。
> 前置：[01-agent-loop.md](01-agent-loop.md)（压缩的触发点 `shouldStopAfterTurn` / `prepareNextTurn` 在循环里）。

## 全流程图（先建立整体观）

```
                     ① 何时压                     ② 在哪切                    ③ 怎么摘
  ┌──────────────────────────────┐   ┌──────────────────────────┐   ┌─────────────────────────┐
  │ estimateContextTokens()      │   │ findValidCutPoints()     │   │ prepareCompaction()     │
  │  usage 锚点 + 尾部字符估算    │   │  合法切点集合             │   │  把 Entry 切成三段       │
  │ shouldCompact()              │→  │ findCutPoint()           │→  │ compact()               │
  │  tokens > 窗口 - 16384        │   │  保留 20000 token 回退    │   │  LLM 生成结构化摘要      │
  └──────────────────────────────┘   └──────────────────────────┘   └─────────────────────────┘
                                                                                │
                                    ┌───────────────────────────────────────────┘
                                    ▼
                     ④ 结果怎么用
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ CompactResult {summary, retainedTail, ...}                                    │
  │   → 存成会话树里的 compaction Entry                                            │
  │   → 下次 buildSessionContext() 重建上下文时：                                  │
  │       [compaction Entry(摘要+retainedTail), ...之后的新消息]  整体替换旧历史      │
  └──────────────────────────────────────────────────────────────────────────────┘
```

## 文件地图

| 行号 | 函数/类型 | 职责 |
| --- | --- | --- |
| 30~67 | `CompactionDetails` / `extractFileOperations` | 文件操作清单（跨代继承） |
| 89~100 | `CompactResult` | 压缩产出 |
| 102~122 | `completeSimpleWithRetries` | 摘要专用的 LLM 调用（隔离、带重试） |
| 147~162 | `CompactionSettings` | 阈值配置 |
| 164~250 | usage/token 估算系列 | "何时压"的判定材料 |
| 252~311 | `estimateTokens` | 单条消息的字符启发式 |
| 312~422 | 切点系列 | "在哪切" |
| 424~498 | 三个摘要提示词 | "怎么摘"的提示词工程 |
| 500~593 | `generateSummary(WithUsage)` | 调 LLM 生成摘要 |
| 595~687 | `prepareCompaction` | 纯函数：把会话切成三段 |
| 706~794 | `compact` | 编排：生成最终 CompactResult |
| 795~848 | `generateTurnPrefixSummary` | split-turn 的前缀摘要 |

**本文件的错误处理风格**：注意大量 `Result<T, CompactionError>`（`ok(...)` / `err(...)`）——**用值表示错误，不用异常**。Java 视角：vavr 的 `Either<CompactionError, T>` 或 Go 的 `(T, error)` 双返回。压缩是"可预期的业务失败"（LLM 调用失败/被取消），走值语义让调用方必须显式处理。

---

## 1. 文件操作清单（第 29~67 行）——压缩也不能忘掉"改过哪些文件"

```ts
/** 存在 compaction entry 上的文件操作明细 */
export interface CompactionDetails {
	readFiles: string[];      // 被压缩历史里读过的文件
	modifiedFiles: string[];  // 被压缩历史里改过的文件
}

// JSON.stringify 不会抛错的防御包装（循环引用等会抛）
function safeJsonStringify(value: unknown): string {
	try {
		return JSON.stringify(value) ?? "undefined";
	} catch {
		return "[unserializable]";
	}
}

function extractFileOperations(
	messages: AgentMessage[],
	entries: Entry[],
	prevCompactionIndex: number,
): FileOperations {
	const fileOps = createFileOps();
	// ★ 跨代继承：如果之前压缩过，先把【上一次 compaction entry 里存的清单】
	// 搬过来当起点。否则第二次压缩后，第一次压缩前读改的文件就全忘了。
	if (prevCompactionIndex >= 0) {
		const prevCompaction = entries[prevCompactionIndex] as CompactionEntry;
		if (prevCompaction.details) {
			const details = prevCompaction.details as CompactionDetails;
			if (Array.isArray(details.readFiles)) {
				for (const f of details.readFiles) fileOps.read.add(f);
			}
			if (Array.isArray(details.modifiedFiles)) {
				for (const f of details.modifiedFiles) fileOps.edited.add(f);
			}
		}
	}
	// 再叠加本次要压缩的消息里的文件操作（扫描 read/edit/write 工具调用）
	for (const msg of messages) {
		extractFileOpsFromMessage(msg, fileOps);
	}
	return fileOps;
}
```

**为什么要有这个？** 模型最常问的问题之一是"我们改过哪些文件"。压缩把原始工具调用换成了摘要，如果没有这份清单，这个信息就永久丢了。清单跟着 compaction entry **代代相传**。

```ts
// 把 Entry 转回消息（供摘要用）：
// message → 原样；branch_summary / compaction → 转成对应的消息类型（保留已有摘要）
function getMessageFromEntry(entry: Entry): AgentMessage | undefined { ... }

// 变体：compaction entry 返回 undefined——【不再摘要旧的摘要】，
// 旧摘要通过 previousSummary 机制增量更新（见第 5 节）
function getMessageFromEntryForCompaction(entry: Entry): AgentMessage | undefined {
	if (entry.type === "compaction") {
		return undefined;
	}
	return getMessageFromEntry(entry);
}
```

## 2. 摘要专用的 LLM 调用（第 102~122 行）

```ts
export async function completeSimpleWithRetries(
	models: Models, model: Model<Api>, context: Context, options: SimpleStreamOptions,
	retry?: RetryPolicy, callbacks?: RetryCallbacks,
): Promise<AssistantMessage> {
	// 摘要是独立请求，做两个隔离决策：
	// 1. cacheRetention: "none" —— 摘要请求的结果写缓存没有意义（不会复用）
	// 2. sessionId: uuidv7()   —— 换个全新 sessionId，不污染主会话的缓存亲和
	const requestOptions: SimpleStreamOptions = {
		...options,
		cacheRetention: "none",
		sessionId: uuidv7(),
	};
	// retryAssistantCall：pi-ai 提供的"瞬时错误自动重试"包装
	return retryAssistantCall(
		() => models.completeSimple(model, context, requestOptions),
		retry,
		requestOptions.signal,
		callbacks,
	);
}
```

## 3. "何时压"：token 估算（第 147~250 行）

```ts
/** 压缩阈值与保留设置 */
export interface CompactionSettings {
	enabled: boolean;        // 是否启用自动压缩
	reserveTokens: number;   // 为摘要的提示词+输出预留的 token（判断触发用）
	keepRecentTokens: number; // 压缩后保留的近期上下文 token（切点用）
}

/** 默认值：保留 16384 触发余量 + 20000 近期保留 */
export const DEFAULT_COMPACTION_SETTINGS: CompactionSettings = {
	enabled: true,
	reserveTokens: 16384,
	keepRecentTokens: 20000,
};

// 从 provider 上报的 usage 算"这次请求总共吃了多少上下文 token"
export function calculateContextTokens(usage: Usage): number {
	return usage.totalTokens || usage.input + usage.output + usage.cacheRead + usage.cacheWrite;
}

// 判定一条 assistant 消息的 usage 是否可信：
// stopReason 是 aborted/error 的消息 usage 不可信（请求没正常完成）
function getAssistantUsage(msg: AgentMessage): Usage | undefined { ... }

/** 估算结果的结构 */
export interface ContextUsageEstimate {
	tokens: number;          // 总估算
	usageTokens: number;     // 最近一条可信 usage 报告的 token
	trailingTokens: number;  // usage 之后的消息的估算
	lastUsageIndex: number | null; // usage 来自哪条消息
}

/** ★ 核心算法：锚点 + 尾部估算 */
export function estimateContextTokens(messages: AgentMessage[]): ContextUsageEstimate {
	const usageInfo = getLastAssistantUsageInfo(messages); // 从尾往头找第一条可信 usage

	if (!usageInfo) {
		// 完全没有锚点：全部用字符启发式估算
		let estimated = 0;
		for (const message of messages) estimated += estimateTokens(message);
		return { tokens: estimated, usageTokens: 0, trailingTokens: estimated, lastUsageIndex: null };
	}

	// 有锚点：锚点之前的消息【不估】——usage 已经涵盖了它们。
	// 只估算锚点之后的"尾巴"。这样误差不会累积。
	const usageTokens = calculateContextTokens(usageInfo.usage);
	let trailingTokens = 0;
	for (let i = usageInfo.index + 1; i < messages.length; i++) {
		trailingTokens += estimateTokens(messages[i]);
	}
	return { tokens: usageTokens + trailingTokens, usageTokens, trailingTokens, lastUsageIndex: usageInfo.index };
}

/** 触发判定 */
export function shouldCompact(contextTokens: number, contextWindow: number, settings: CompactionSettings): boolean {
	if (!settings.enabled) return false;
	return contextTokens > contextWindow - settings.reserveTokens;
	// 例：窗口 200000、reserve 16384 → 超过 183616 就该压了。
	// 留余量是因为下一次请求还要装下新回复（以及万一本次就该压缩时的摘要提示词）。
}
```

**"锚点 + 尾部"为什么聪明**：provider 的 usage 是**精确值**（它知道真实 token 数），但只覆盖到那条消息为止；之后的消息只能估。把精确值当锚点、只估尾部，误差被限制在最近几条消息内，不会随会话变长而累积。

## 4. 单条消息的字符启发式（第 252~311 行）

```ts
const ESTIMATED_IMAGE_CHARS = 4800;   // 一张图片按 4800 字符估

export function estimateTokens(message: AgentMessage): number {
	let chars = 0;
	switch (message.role) {
		case "user":
			chars = estimateTextAndImageContentChars(message.content);
			return Math.ceil(chars / 4);          // 4 字符 ≈ 1 token（英文经验值）
		case "assistant":
			// 文本 + thinking + 工具调用（name + JSON.stringify(arguments)）都算
			for (const block of assistant.content) {
				if (block.type === "text") chars += block.text.length;
				else if (block.type === "thinking") chars += block.thinking.length;
				else if (block.type === "toolCall") chars += block.name.length + safeJsonStringify(block.arguments).length;
			}
			return Math.ceil(chars / 4);
		case "custom":
		case "toolResult":
			chars = estimateTextAndImageContentChars(message.content);
			return Math.ceil(chars / 4);
		case "bashExecution":
			chars = message.command.length + message.output.length;
			return Math.ceil(chars / 4);
		case "branchSummary":
		case "compactionSummary":
			chars = message.summary.length;
			return Math.ceil(chars / 4);
	}
	return 0;
}
```

4 字符/token 对英文偏保守、对中文偏乐观（中文约 1.5~2 字符/token），所以中文会话下实际占用可能比估算高——这也是为什么触发判定要留 reserveTokens 余量。

## 5. "在哪切"：切点选择（第 312~422 行）

```ts
/** 找出 [startIndex, endIndex) 内所有【合法切点】的下标 */
function findValidCutPoints(entries: Entry[], startIndex: number, endIndex: number): number[] {
	const cutPoints: number[] = [];
	for (let i = startIndex; i < endIndex; i++) {
		const entry = entries[i];
		switch (entry.type) {
			case "message": {
				switch (entry.message.role) {
					// 这些角色的消息【前面】可以切：
					// user（turn 起点）、assistant、custom、branchSummary、compactionSummary、bashExecution
					case "user":
					case "assistant":
					case "custom":
					case "bashExecution":
					case "branchSummary":
					case "compactionSummary":
						cutPoints.push(i);
						break;
					case "toolResult":
						break;   // ★ toolResult 前面【不能切】！
				}
				break;
			}
			// 纯元数据 entry（模型/思考级别变更等）不是切点
			case "model_change": /* ... */ break;
		}
	}
	return cutPoints;
}
```

**为什么 toolResult 前不能切**：上下文里出现"孤儿 toolResult"（前面没有对应的 assistant 工具调用）会被 LLM provider 直接拒绝请求。所以切点只能落在"回合边界"或至少"assistant 消息边界"上。

```ts
/** 从 entryIndex 往前找包含它的 turn 的起点（用户可见的起点消息） */
export function findTurnStartIndex(entries: Entry[], entryIndex: number, startIndex: number): number {
	for (let i = entryIndex; i >= startIndex; i--) {
		const entry = entries[i];
		if (entry.type === "branch_summary") return i;
		if (entry.type === "message") {
			const role = entry.message.role;
			if (role === "user" || role === "bashExecution") return i;  // 用户说话/敲命令 = turn 起点
		}
	}
	return -1;
}

/** 切点结果 */
export interface CutPointResult {
	firstKeptEntryIndex: number; // 压缩后保留的第一个 entry 下标
	turnStartIndex: number;      // 切点切在 turn 中间时，那个 turn 的起点（否则 -1）
	isSplitTurn: boolean;        // 是否切在了一个 turn 中间
}

/** ★ 核心算法：从尾部往回累计 token，找到保留预算的切点 */
export function findCutPoint(
	entries: Entry[], startIndex: number, endIndex: number, keepRecentTokens: number,
): CutPointResult {
	const cutPoints = findValidCutPoints(entries, startIndex, endIndex);
	if (cutPoints.length === 0) {
		return { firstKeptEntryIndex: startIndex, turnStartIndex: -1, isSplitTurn: false };
	}
	let accumulatedTokens = 0;
	let cutIndex = cutPoints[0];   // 兜底：至少从第一个合法切点切

	// 从【最新】往【最旧】走，累计每条消息的估算 token
	for (let i = endIndex - 1; i >= startIndex; i--) {
		const entry = entries[i];
		if (entry.type !== "message") continue;
		accumulatedTokens += estimateTokens(entry.message as AgentMessage);
		// 一旦累计到保留预算：在合法切点里找【第一个 ≥ 当前位置】的
		// （即：从尾部数够了 20000 token，切点就在这个位置附近往前对齐到合法边界）
		if (accumulatedTokens >= keepRecentTokens) {
			for (let c = 0; c < cutPoints.length; c++) {
				if (cutPoints[c] >= i) { cutIndex = cutPoints[c]; break; }
			}
			break;
		}
	}
	// 再往前对齐：如果切点前面是纯元数据 entry（model_change 等），
	// 继续前移直到前面是 message 或 compaction（切点尽量贴近实际消息边界）
	while (cutIndex > startIndex) { /* ... cutIndex-- ... */ }

	const cutEntry = entries[cutIndex];
	const isUserMessage = cutEntry.type === "message" && cutEntry.message.role === "user";
	// 切在 user 消息上 = 完美的 turn 边界，没有 split；
	// 否则（比如切在 assistant 上）→ 尝试找它所在 turn 的起点，标记为 split turn
	const turnStartIndex = isUserMessage ? -1 : findTurnStartIndex(entries, cutIndex, startIndex);
	return {
		firstKeptEntryIndex: cutIndex,
		turnStartIndex,
		isSplitTurn: !isUserMessage && turnStartIndex !== -1,
	};
}
```

**split turn 是什么场景**：一个超长的 turn（用户问了一句 → agent 连跑 30 个工具）。从尾部保留 20000 token 可能把切点落在 turn 中间——比如第 18 个工具调用之后。这时的处理不是硬切（会产生孤儿 toolResult），而是把这个 turn 的**前缀单独摘要**（见第 8 节的 `TURN_PREFIX_SUMMARIZATION_PROMPT`），保留后缀。

## 6. 提示词工程（第 424~498 行）——三段式

```ts
// 系统提示词：给摘要 LLM 的身份定位，防止它"顺着对话继续回答"
export const SUMMARIZATION_SYSTEM_PROMPT = `You are a context summarization assistant. ...
Do NOT continue the conversation. Do NOT respond to any questions in the conversation.
ONLY output the structured summary.`;
// ↑ "不要继续对话、不要回答对话里的问题、只输出结构化摘要"——
//   把长历史原样塞给 LLM 时，它很容易"接话"，这个禁令是必要的。

// 首次摘要提示词：固定六段格式
const SUMMARIZATION_PROMPT = `The messages above are a conversation to summarize.
Create a structured context checkpoint summary that another LLM will use to continue the work.
//                                              ^^^^^^^^^^^^^^^^^^^^ 关键定位：
//                                              摘要的读者是【另一个 LLM】，是"接力棒"不是"会议纪要"
Use this EXACT format:
## Goal                    ← 用户想干什么
## Constraints & Preferences ← 约束和偏好
## Progress                ← Done / In Progress / Blocked 三栏
## Key Decisions           ← 决策 + 理由
## Next Steps              ← 有序的下一步
## Critical Context        ← 继续工作所需的数据/示例/引用
Keep each section concise. Preserve exact file paths, function names, and error messages.`
// ↑ 最后一行是硬要求：精确保留路径/函数名/错误信息——
//   摘要丢了精确字符串，后续工作就会在细节上翻车

// 增量更新提示词：已有旧摘要时用这个
const UPDATE_SUMMARIZATION_PROMPT = `... RULES:
- PRESERVE all existing information from the previous summary
- ADD new progress, decisions, and context from the new messages
- UPDATE the Progress section: move items from "In Progress" to "Done" when completed
- UPDATE "Next Steps" based on what was accomplished
- PRESERVE exact file paths, function names, and error messages
- If something is no longer relevant, you may remove it ...`
// ↑ 增量模式不是"重新摘要一切"，而是"在旧摘要上打补丁"——省 token 且稳定。
//   新旧对话通过 <previous-summary> 标签传入（见下一节）。
```

## 7. 生成摘要（第 500~593 行）

```ts
export async function generateSummaryWithUsage(
	currentMessages: AgentMessage[],   // 要摘要的消息
	models: Models, model: Model<Api>, // 用哪个 LLM（通常是当前主模型）
	reserveTokens: number,
	signal?: AbortSignal,
	customInstructions?: string,       // 调用方附加的侧重指令（拼接 "Additional focus: ..."）
	previousSummary?: string,          // 有 → 走增量提示词；无 → 走首次提示词
	thinkingLevel?: ThinkingLevel,
	retry?, callbacks?,
): Promise<Result<{ text: string; usage: Usage }, CompactionError>> {
	// 摘要输出上限 = reserveTokens 的 80%（给主对话留 20%）
	const maxTokens = Math.min(Math.floor(0.8 * reserveTokens), model.maxTokens > 0 ? model.maxTokens : Infinity);

	let basePrompt = previousSummary ? UPDATE_SUMMARIZATION_PROMPT : SUMMARIZATION_PROMPT;
	if (customInstructions) basePrompt = `${basePrompt}\n\nAdditional focus: ${customInstructions}`;

	// 组装提示词：对话全文包 <conversation> 标签；旧摘要包 <previous-summary> 标签
	const llmMessages = convertToLlm(currentMessages);
	const conversationText = serializeConversation(llmMessages);
	let promptText = `<conversation>\n${conversationText}\n</conversation>\n\n`;
	if (previousSummary) promptText += `<previous-summary>\n${previousSummary}\n</previous-summary>\n\n`;
	promptText += basePrompt;

	// 一条 user 消息 + 摘要系统提示词 → 完整补全（completeSimple，不是流式）
	const response = await completeSimpleWithRetries(models, model,
		{ systemPrompt: SUMMARIZATION_SYSTEM_PROMPT, messages: [{ role: "user", content: [...], timestamp: Date.now() }] },
		completionOptions, retry, callbacks);

	// 错误编码进 Result（不是异常！）
	if (response.stopReason === "aborted") return err(new CompactionError("aborted", ...));
	if (response.stopReason === "error") return err(new CompactionError("summarization_failed", ...));

	return ok({ text: contentText(response.content), usage: response.usage });
}
```

## 8. 准备与编排：`prepareCompaction` / `compact`（第 595~794 行）

```ts
/** 压缩准备的结果：会话被切成三段 */
export interface CompactionPreparation {
	messagesToSummarize: AgentMessage[]; // ① 要摘要的旧历史
	turnPrefixMessages: AgentMessage[];  // ② split-turn 时的 turn 前缀（单独摘要）
	retainedTail: AgentMessage[];        // ③ 保留的近期消息（原样存进 compaction entry）
	isSplitTurn: boolean;
	tokensBefore: number;                // 压缩前的估算 token
	previousSummary?: string;            // 上次的摘要（增量模式）
	fileOps: FileOperations;             // 文件操作清单
	settings: CompactionSettings;
}

/** 纯函数：把会话 entry 切成三段，不调 LLM */
export function prepareCompaction(pathEntries: Entry[], settings: CompactionSettings
): Result<CompactionPreparation | undefined, CompactionError> {
	// 空会话 / 最后一个 entry 已经是 compaction（刚压完又触发）→ 不适用
	if (pathEntries.length === 0 || pathEntries.at(-1).type === "compaction") return ok(undefined);

	// 找上一次 compaction
	let prevCompactionIndex = -1; /* 从尾往头找 type === "compaction" */

	let previousSummary: string | undefined;
	let compactableEntries = pathEntries;
	if (prevCompactionIndex >= 0) {
		const prevCompaction = pathEntries[prevCompactionIndex] as CompactionEntry;
		previousSummary = prevCompaction.summary;
		// ★★ 本文件最巧的一段：把上次 compaction 的 retainedTail（消息数组）
		//    还原成【虚拟 entry】（type: "message"，id 用 "xxx:retained:N" 拼出来），
		//    拼在上次 compaction 之后、新消息之前。
		//    这样第二次压缩时，上次保留的尾巴也能被正常切分/摘要，
		//    整个算法不需要为"多次压缩"写特例。
		const virtualRetainedEntries: Entry[] = prevCompaction.retainedTail.map((message, index) => ({
			type: "message",
			id: `${prevCompaction.id}:retained:${index}`,
			parentId: index === 0 ? prevCompaction.id : `${prevCompaction.id}:retained:${index - 1}`,
			seq: prevCompaction.seq,
			timestamp: message.timestamp,
			message,
		}));
		compactableEntries = [...virtualRetainedEntries, ...pathEntries.slice(prevCompactionIndex + 1)];
	}

	// 切点 → 三段切分
	const cutPoint = findCutPoint(compactableEntries, 0, compactableEntries.length, settings.keepRecentTokens);
	const historyEnd = cutPoint.isSplitTurn ? cutPoint.turnStartIndex : cutPoint.firstKeptEntryIndex;
	// ① [0, historyEnd) → messagesToSummarize
	// ② [turnStartIndex, firstKeptEntryIndex) → turnPrefixMessages（split 时）
	// ③ [firstKeptEntryIndex, end) → retainedTail
	// （每段都走 getMessageFromEntryForCompaction——旧 compaction entry 被跳过）

	// 文件操作清单 = 上代清单 + 本次被摘要消息的操作
	const fileOps = extractFileOperations(messagesToSummarize, pathEntries, prevCompactionIndex);
	return ok({ messagesToSummarize, turnPrefixMessages, retainedTail, ... });
}
```

```ts
/** 编排：生成最终 CompactResult */
export async function compact(preparation, models, model, customInstructions?, signal?, ...)
: Promise<Result<CompactResult, CompactionError>> {
	const { messagesToSummarize, turnPrefixMessages, retainedTail, isSplitTurn, previousSummary, fileOps, settings } = preparation;

	let summary: string;
	let summaryUsage: Usage;

	if (isSplitTurn && turnPrefixMessages.length > 0) {
		// split-turn：两次 LLM 调用
		// 第一次：正常摘要旧历史（有 previousSummary 走增量）
		const historyResult = await generateSummaryWithUsage(messagesToSummarize, ...);
		if (!historyResult.ok) return err(historyResult.error);
		// 第二次：单独摘要 turn 前缀（用 TURN_PREFIX 提示词，输出上限 0.5×reserve）
		const turnPrefixResult = await generateTurnPrefixSummary(turnPrefixMessages, ...);
		if (!turnPrefixResult.ok) return err(turnPrefixResult.error);
		// 两个摘要拼接：历史摘要在前，turn 上下文在后（用分隔线 + 标题区分）
		summary = `${historyText}\n\n---\n\n**Turn Context (split turn):**\n\n${turnPrefixResult.value.text}`;
		summaryUsage = combineUsage(historyUsage, turnPrefixUsage);   // 两段 usage 相加
	} else {
		// 正常路径：一次 LLM 调用
		const summaryResult = await generateSummaryWithUsage(messagesToSummarize, ...);
		if (!summaryResult.ok) return err(summaryResult.error);
		summary = summaryResult.value.text;
		summaryUsage = summaryResult.value.usage;
	}

	// 最后把文件清单格式化后追加进摘要（模型问"改过哪些文件"时直接可答）
	const { readFiles, modifiedFiles } = computeFileLists(fileOps);
	summary += formatFileOperations(readFiles, modifiedFiles);

	return ok({
		summary,
		tokensBefore,
		usage: summaryUsage,     // 摘要调用自己的花费（记账用）
		retainedTail,            // ★ 保留尾巴跟着 entry 走，重建上下文时原样恢复
		details: { readFiles, modifiedFiles } as CompactionDetails,
	});
}
```

split-turn 前缀的提示词（第 689~702 行）要求输出三段：`Original Request`（这个 turn 用户要什么）/ `Early Progress`（前缀里做了什么）/ `Context for Suffix`（理解保留下 suffix 需要什么）——定位非常精确：**它是给"接下来的保留部分"当引子的，不是完整纪要**。

## 与 coding-agent 的关系

coding-agent 的 `core/compaction/compaction.ts`（1012 行）是同一套思路的产品级实现，多了三种触发路径（manual / threshold / overflow 溢出恢复）和扩展接管点（`session_before_compact` 事件）。学完本文件再去看那份，基本是"复习 + 变体"。

## 自测

1. 触发压缩的条件是什么？为什么是 `窗口 - reserveTokens` 而不是 `窗口`？（要给下一轮回复留空间）
2. 为什么 toolResult 前面不能当切点？（孤儿 toolResult 会被 provider 拒绝）
3. split turn 是怎么产生的？分几步处理？（预算切点落在 turn 中间 → 前缀单独摘要 + 后缀保留）
4. `estimateContextTokens` 为什么"只估尾部"？锚点是什么？（provider 的真实 usage；防止误差累积）
5. 第二次压缩时，上一次的 retainedTail 和旧摘要分别走什么路径？（虚拟 entry 重新参与切分 / previousSummary 增量更新）
6. 文件清单为什么要跨代继承？存放在哪里？（CompactionDetails，跟着 compaction entry 代代相传）
