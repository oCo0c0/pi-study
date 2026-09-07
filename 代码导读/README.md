# 代码导读（中文注释版）

## 这是什么、为什么这样做

这里是 pi 核心源码的中文学习材料，分两层配合使用：

1. **源码内注释（对照代码看时用）**：以下五个核心源文件已在关键位置（文件头 + 每个重要函数 + 最微妙的逻辑点）加入了中文点睛注释，原有英文注释全部保留、一行未删。文件头注释里有该文件的结构地图，并链接到对应的导读文档。
2. **导读文档（系统学习/写博客时用）**：本目录的逐段深度讲解，覆盖源码注释放不下的推理过程、Java 视角对照和自测题。

五个加了源码注释的文件（均为纯注释插入，0 行代码改动）：

| 源文件 | 注释处数 | 配套导读 |
| --- | --- | --- |
| `packages/agent/src/agent-loop.ts` | 13 | [01-agent-loop.md](01-agent-loop.md) |
| `packages/agent/src/agent.ts` | 11 | [02-agent.md](02-agent.md) |
| `packages/agent/src/types.ts` | 9 | [03-agent-types.md](03-agent-types.md) |
| `packages/agent/src/harness/compaction/compaction.ts` | 12 | [04-compaction.md](04-compaction.md) |
| `packages/coding-agent/src/core/sdk.ts` | 9 | [05-sdk.md](05-sdk.md) |

注意：这些注释是本地未提交的改动（`git diff --stat` 可验证只有 insertions）。以后 `git pull` 更新如果冲突，直接 `git checkout -- <文件>` 丢弃本地注释即可——深度讲解都在本目录文档里，不受影响；需要时也可以让我重新加上。

## 给 Java 开发者的 TS 概念速查表

先说结论：这个仓库的 TypeScript 写法偏"Java 风"——重接口、重类型、类 + 组合、几乎没有花哨的函数式技巧。你读 `Agent` 类的感觉会接近读一个设计良好的 Java 类。下面这些概念对照能帮你扫清 90% 的阅读障碍：

| TypeScript/JS 概念 | 在 pi 里的样子 | Java 对应 / 关键差异 |
| --- | --- | --- |
| `Promise<T>` / `async-await` | 所有 IO 操作（LLM 调用、文件、shell） | `CompletableFuture<T>`。**关键差异**：JS 是单线程事件循环，`await` 是唯一的让出点，内存状态不会被并发抢占，所以整个仓库**没有一把锁** |
| `for await...of` | 遍历 LLM 事件流 | 类似 Reactor 的 `toIterable()`，拉取式消费流 |
| `EventStream`（push/end/result） | ai 包的事件流容器，后台任务往里 push，消费者迭代 | Reactor 的 `Sinks.Many`：生产者 `tryEmitNext`，订阅者收到 |
| `AbortController` / `AbortSignal` | 贯穿所有操作的取消信号 | `Thread.interrupt()` + `Future.cancel(true)`，但**纯协作式**：signal 只是个标志位，代码要自己检查 |
| 判别联合 `{ type: "xxx", ... }` | `AgentEvent`、消息类型、工具调用状态 | **Java 21 的 `sealed interface` + `record` + `switch` 模式匹配**，几乎一一对应 |
| interface 声明合并（declaration merging） | `CustomAgentMessages` 让应用注册自定义消息类型 | 无直接对应；编译期的"接口内容合并"，类似运行时插件注册的静态版 |
| TypeBox schema | 工具参数定义（`parameters` 字段） | JSON Schema + Bean Validation。但注意：schema 是普通 JS 对象，**会原样发给 LLM**，不只是本地校验用 |
| getter/setter 属性 | `MutableAgentState` 的 `tools`/`messages` | Bean 的 getter/setter，这里还做了防御性拷贝（set 时 slice 顶层数组） |
| 对象展开 `{ ...obj, x: 1 }` | "不可变风格"更新（如 `createLoopConfig`） | 相当于一行写完 `new Builder().from(obj).set(x,1).build()` |
| `??`（空值合并）与 `?.`（可选链） | `config.getSteeringMessages?.() \|\| []` | Optional 链的语法糖。注意 JS 里 `undefined` 和 `null` 是两个值，`??` 只对这两个生效，对 `0`/`""`/`false` 不生效 |
| ESM import / package.json exports | `import { Agent } from "@earendil-works/pi-agent-core"` | Maven 模块。import 路径里带 `.ts` 扩展名是 Node 原生跑 TS 的新特性 |
| `satisfies` / `Omit<>` / `Partial<>` | `MutableAgentState = Omit<AgentState,...> & {...}` | 纯编译期类型工具，运行时零开销。`Omit` ≈ "从接口里减掉几个字段"，`Partial` ≈ "所有字段变可空" |
| `void expr`（故意不 await） | `void runAgentLoop(...)` | 相当于 `CompletableFuture.runAsync(...)` 不 join——故意后台跑 |

**读 TS 代码的三个提醒：**

1. 所有类型标注只影响编译期，运行时就是普通 JS——看到一个复杂类型不要慌，运行时行为看代码就行。
2. `async` 函数**总是**返回 Promise；调用时不加 `await` 通常是"故意不等待"（本仓库会用 `void` 标明意图）。
3. 错误处理两条铁律在这个仓库里处处体现：LLM 流的错误**不抛出**而是编码进消息（`stopReason: "error"`）；工具的错误**必须抛出**（循环负责接住转成错误结果）。

## 已覆盖文件清单（全部完成）

| 文档 | 对应源码 | 主题 |
| --- | --- | --- |
| [01-agent-loop.md](01-agent-loop.md) | `packages/agent/src/agent-loop.ts`（803 行） | agent 循环的心脏：双层循环、工具三段式执行、并发与三种顺序 |
| [02-agent.md](02-agent.md) | `packages/agent/src/agent.ts`（592 行） | 有状态的 Agent 类：事件屏障、双数组设计、steering/followUp 队列 |
| [03-agent-types.md](03-agent-types.md) | `packages/agent/src/types.ts`（445 行） | 运行时的全部契约：StreamFn、AgentLoopConfig、AgentTool、AgentEvent |
| [04-compaction.md](04-compaction.md) | `packages/agent/src/harness/compaction/compaction.ts`（849 行） | 上下文压缩：token 锚点估算、切点选择、结构化摘要、增量更新 |
| [05-sdk.md](05-sdk.md) | `packages/coding-agent/src/core/sdk.ts`（411 行） | 组合根：模型三级解析链、streamFn 洋葱包装、扩展前向引用、AgentSession 装配 |

## 建议的读法

1. 先读 [学习路线.md](../学习路线.md) 第三节"一次提问的完整生命周期"，建立全局图。
2. 按顺序读 01 → 02 → 03（agent 运行时三部曲），每读完一段就去源码里找到对应行号看一遍原版。
3. 04（压缩）和 05（装配）可以按兴趣穿插：04 属于 agent 包 harness 层，05 进入 coding-agent 产品层。
4. 读完后动手：跑 `packages/coding-agent/examples/sdk/01-minimal`，然后自己仿写一个最小 agent（见学习路线第十五节）。
