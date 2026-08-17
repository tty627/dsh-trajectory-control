# Agent Note: 轨迹控制 —— 暂停、分支回滚与原地回滚

Status: proposed

[English](2026-08-17-trajectory-control-and-in-place-rewind.md) | 中文

## 问题

Trajectory 视图是会话事件流的被动回放。用户看着 agent 走错方向，却无法对所见内容采取行动：不能在正看着的那一步截住它，不能回到更早的决策点重新插入对话，也无法对比换一条指令会产生什么结果。现有控制手段都是会话级的——取消整个 turn，或者从头再来。

所要求的能力是把该视图变成控制台：暂停运行中的 agent、回滚到选定点（分支或原地重写会话）、在该点插入新对话、并驱动 harness 从那里继续。目标是通过控制模型看到的每一个中间步骤来控制其输出。

现有运行时的三条性质约束了任何方案：

- **会话日志只可追加**，且 `Model-visible ⟺ logged` 不变量要求任何进入模型请求的内容都必须可从该日志重建。回滚不能截断，也不能是内存级修改。
- **世界不可回滚。** 工具副作用（文件写入、进程、网络）已经发生。回滚对话不会撤销它们。
- **暂停粒度受循环边界限制。** agent 只能停在 turn 或 pre-step 边界，无法冻结在工具执行中途。

本设计的上一版（在本 note 出现之前保存为 `trajectory-control-plugin/DESIGN.md`）得出了正确的功能集，但在三个承重事实上判断有误，且方向相反：它把一个并不存在的核心约束列为项目最高风险，同时完全漏掉了两个真实阻塞项。记录经核实的结论是本 note 的主要价值。

## 提案

以**仓库内 packages** 而非动态 Cordis 插件交付轨迹控制，分五个阶段：一个含两项的前置阶段（T-1），然后是 T0–T3 功能梯队。

### 交付形态：为什么是仓库内 packages

动态 Cordis 插件无法承载此功能，有三条各自独立的原因：

- **新增会话事件。** 持久化读路径会拒绝含有本构建不认识的事件类型的日志，除非该事件标记了 `ignorable`（`packages/session/session-persistence/src/coordinator.ts:1063`）。该词汇表是生成的 `KNOWN_SESSION_EVENT_TYPES`（`packages/core/session/src/known-event-types.ts:19`），从仓库内的 `SessionEventMap` 合并扫描得出；其 JSDoc 明确指出仓库外插件事件按构造方式即不在此列表内。
- **新增 RPC。** `RpcMethodMap`（`packages/host/apiproxy/src/api/rpc-map.ts:24`）是封闭 interface，无声明合并点。
- **Client 渲染改动。** conversation 视图的 replacement 处理位于 `packages/client/ui-conversation` 内部（见下文 T-1 兜底项）。

[会话日志版本机制 note](https://github.com/deepseek-ai/deepseek-harness/blob/main/.agents/notes/implemented/architecture/2026-08-10-session-log-version-mechanism.md) 已记录并接受这一现状：仓库外插件事件在第一方读取器下会拒绝 resume，且该拒绝是显式而非静默的。动态插件对于原型验证 T0 中无需新事件、无需新 RPC 的那部分仍然有用。

### T-1：两项前置，各为独立改动

**1. `Session.append` 的 `ignorable` 写入面。** 该字段存在于事件 envelope，并已被 seed 校验、两个持久化后端与 BFF wire schema 认可，但在此改动之前没有任何写入方能设置它：`append` 唯一的可选参数是 `SurfaceIntent`，构造 envelope 时不写该字段。因此任何新事件类型都是 required-on-read，旧构建遇到其一即拒绝整条会话。

版本机制 note 恰好预留了这一改动：*"writers do not yet set `ignorable` (no producer needs it), so `Session.append` gains that surface with its first user."* 轨迹控制就是那个 first user。

**本项已交付。** `SurfaceIntent` 是错误的载体——它仅对三种 `SurfaceEventType` 变体可达，因为 `append` 使其 options 参数以事件类型为条件，故 `trajectory/annotation` 这类非 surface 事件永远无法通过它传值。现在该 options 参数按事件类别在两个互斥类型之间取一：`SurfaceEventType` 一如既往要求 `SurfaceIntent`，其余类型接受可选的、承载 `ignorable` 的 `LogIntent`。

采用互斥类型而非单一合并对象，使两类错误不可表达而非仅仅不被鼓励：编译器拒绝仅日志事件声明 surface 位置，也拒绝产生消息的事件标记 `ignorable`。后者是承重的一半——surface 事件永远是被识别的、模型可见的类型，跳过它不可能安全，而共享 options 对象会让这种写法仍然可表达。

`SESSION_FORMAT_VERSION` 保持为 `0`：该字段已属 envelope 的一部分，本改动只是补上缺失的生产者。

**2. 未识别 surface replacement 的兜底渲染器。** conversation 视图靠硬编码插件名识别 replacement：

```text
// packages/client/ui-conversation/src/client/conversation-nodes/message.ts:25-28
if (event.type !== 'user/message' || !isReplacementSurfaceEvent(event)) return false
const source = event.data.source
return source.kind === 'plugin' && source.plugin === 'compact'
```

`command.ts:83` 使用同样的 `plugin === COMPACT_PLUGIN` 判断，而通用 fallback 只匹配 `isAppendSurfaceEvent`（`conversation-nodes/fallback.ts:19`）。因此来自任何其他生产者的 `user/message` replacement 既无 definition 认领、也无 fallback 兜住：**它渲染为空白。** 原地回滚产生的正是这种事件形状。

**本项已交付**，且落地方式是扩展既有兜底而非新增第二个：`ConversationEventRegistry.registerFallback` 只接受一个兜底，重复注册会抛错；assembler 仅在没有任何普通 Definition 认领该 target 时才咨询它（`packages/client/runtime/src/client/conversation/event-registry.ts:34-50`、`sessions/conversation-assembler.ts:376-383`）。因此 `unknownFallbackDefinition` 现在同时匹配两种 surface 来源，`UnknownSurfaceNode` 携带可选的 `replaced` 被遮蔽 seq 列表，取自 `sourceEventSeqs`——它是被遮蔽节点的超集，因为核心要求该字段包含每一个被遮蔽节点，而生产者可以额外引用其他来源。

该节点报告历史已被遮蔽，而非省略内容，因为 `isAppendSurfaceEvent` 存在的理由正是避免已落地的 replacement 抹掉用户已读过的转录（`packages/core/session/src/surface.ts:44-47`）。Trajectory 视图无需对应改动：其 message Definition 不分 surface 来源地匹配 `user/message`，故 replacement 在那里从未被丢弃。

append 来源的兜底在装配窗口中仍不可达——三种 `SurfaceEventType` 成员都有各自的 Definition——故该分支在 Definition 层面覆盖，并保留为核心侧扩大该类型集时的降级路径。

### 原地回滚无需改动 surface 机制

上一版称 replace 范围被约束为处于"两个已存在节点之间"，需要扩展核心以允许"从锚点到日志尾部"的遮蔽，并将其评为项目最高风险。**该约束并不存在。** 校验只要求 `start` 与 `end` 都存在于**当前** surface 中且按位置有序（`packages/core/session/src/surface.ts:246-266`）。由于替换事件追加在尾部，校验时的最后一个 surface 节点即是尾部，故"锚点到尾部"今天就能表达。仓库内有两处先例：`packages/host/apiproxy/tests/api-proxy-models.spec.ts:221` 使用 `{ op: 'replace', start: 0, end: events.length - 1 }`，`packages/compaction/compaction/tests/tool-pairing.spec.ts:171` 使用 `end: nodes.at(-1)!`。

照上一版执行会主动造成损害：`isReplaceOp` 要求恰好三个键（`surface.ts:175`），任何新增字段都会在 append 时被拒；而 `SurfaceOp` 变体被 `SESSION_FORMAT_VERSION` 的 bump 规则明文列举（`types.ts:41-52`），该改动将为一项已具备的能力付出版本 bump 的代价。

四条真实约束，上一版一条都未识别：

1. `end` 必须是 **surface** 节点，而非最后一个日志事件。尾部若是 `turn/end`、`step/end` 或 chunk，会以 `end seq N not found in surface` 失败；调用方必须自行解析到最后一个 surface 节点。
2. `sourceEventSeqs` 必须列举**每一个**被遮蔽节点（`surface.ts:239`），且每个都严格早于本事件（`surface.ts:235`）。这使得遮蔽信息的载体是那条 replacement `user/message`，而非 `trajectory/rewind` 的 payload。
3. 该范围不能表达为 `tool/result`：此类 replacement 被限制为一个节点、必须指向 `tool/result`、且只能改 content（`surface.ts:287-318`）。用 `user/message`。
4. 顺序按 surface 位置判定，而非 seq 数值。

由于真 replace 语义可用，上一版的"隐形 fork"过渡方案被舍弃：它会留下误导性的分支血缘而无任何收益。

### 事件词汇

一个 required 事件与两个 ignorable 事件。上一版的 `trajectory/branch` 与 `trajectory/explore` 被删除——血缘可由 `header.parentSession` 重建，探索结果可并入标注。

- `trajectory/rewind`（**required**）—— `{ anchorSeq, replacementSeq }`。标记模型上下文自锚点重新演绎，用以区别于 compaction 结构上完全相同的 replacement。旧构建会拒绝加载含此事件的会话，这是预期行为：把被遮蔽的历史当作普通日志重建会得到错误的模型上下文。
- `trajectory/annotation`（**ignorable**）—— 锚定的笔记；`visibleToModel` 时在 `agent/pre-step` 注入。
- `trajectory/pause`（**ignorable**）—— 暂停审计记录。

新增普通事件类型不 bump `SESSION_FORMAT_VERSION`；`ignorable` 守卫覆盖词汇增长。

### 复用而非重建

`ctx.sessionQuery` 与既有 projection 已提供上一版计划自建的读模型：`traceSession()` 提供血缘（祖先加递归后代，带 `complete: false` 标记与环检测）、`traceEvent()` 提供 `replacementChain`、`listEvents()` 提供 `current | shadowed | log-only` 分类、`session-stats` 提供 turn/step 计数与各阶段墙钟时间、`session-log-export` 提供 ZIP/JSONL 导出。轨迹控制消费这些能力，不定义竞争性服务，这同时避免了只有单一内部调用者的公开服务方法。

该复用路径上有两处真实缺口，各值得独立修复：`parent_session` 无 SQL 索引（`packages/session-query/session-query-sqlite/src/schema.ts` 只声明了 `id TEXT PRIMARY KEY`），以及 `listChildren` 额外要求 `origin === 'subagent'`（`packages/subagent/subagent/src/list-children.ts:141`），从而排除了血缘面板必须展示的普通 fork 子会话。

### 对运行时模型的纠正

上一版的流程在以下经核实的事实面前无法编译或行为错误：

- **没有 `paused` 状态。** `AgentStatus = 'idle' | 'running'`（`packages/core/agent/src/runtime-types.ts:50`）。暂停是 `idle` 加一个插件自有标记。
- **`AgentCancelCause` 是封闭联合** —— `user | parent | hook | disposed`（`packages/core/session/src/types.ts:143`）。`{ kind: 'user-pause' }` 非法；`{ kind: 'hook', reason: 'trajectory-pause' }` 可携带审计原因。
- **`agent.steering()` 不存在。** steering 是传给 `agent.send` 的 `InboxTarget = 'next-step'`（`packages/core/agent/src/types.ts:10`）。
- **`session.fork` 已建好带 agent 的子会话**并继承 preset（`packages/host/apiproxy/src/api-proxy.ts:2420`），故无需单独的 resume 步骤。向后吸附到下一个 `turn/end` 的逻辑位于该 RPC（`:2382`）；`SessionStore.fork` 不吸附，且对结束于开放 turn 内的前缀以 `OPEN_TURN` 拒绝（`packages/core/session/src/index.ts:1128`）。规则是"不得结束于开放 turn 之内"，而非"必须落在 `turn/end` 上"：已闭合 turn 之后的 log-only 事件是合法边界。
- **preset 解析读日志而非 header**（`api-proxy.ts:1258`），因为在空白期切换过 composition 的会话，其 turn 是在新 composition 下跑的，而 header 只在创建时写一次。
- **`turn/end.reason` 是结构化值**，且 `blocked` 与 `max-tokens` 应与 `error` 一同纳入自动暂停规则。`agent/turn-stopping` 是 `@mode serial`，不能否决终止。
- **工具参数不可改写。** `ToolExecutionInput.arguments` 是 `readonly`，且 `tools/execute` 声明调用身份不可变（`packages/core/tools/src/index.ts:154`）。在审批处编辑指令可经 pre-step 重写实现；编辑工具参数则不可行，正确替代是以 `tools/pre-execute` 返回 block 加纠正性反馈，让模型重发调用，从而保持日志真实。

pre-step "park"——在人类决定期间挂住一步——是可行的：`agent/pre-step` 是支持 async listener 且带 signal 的 waterfall（`runtime-types.ts:231`），`userQuestions.ask()` 接受 signal 并等待人类，而工具调用超时策略仅作用于 `tools/execute`，无法杀死被 park 的步骤。`plan-mode` 与 `tool-cordis` 已在该边界上执行异步工作并注入消息。

### 分期

| 阶段 | 内容 |
|---|---|
| **T-1** | `ignorable` 写入面（**已交付**）；replacement 兜底渲染器（**已交付**） |
| **T0** | 实时暂停/恢复；复用既有 `forkAt` 的轨迹行操作；steer/queue 插入表单；暂停横幅 |
| **T1** | 标注；基于 `traceSession` 的血缘面板；原地回滚；仅限指令的审批编辑 |
| **T2** | 断点 park；错误自动暂停；副作用清单；轨迹 diff；分支探索 |
| **T3** | 基于 `session-log-export` 的导出/导入/重放；基于 `session-stats` 的统计；子 agent 内联；goal 联动 |

T0 比上一版设想的更小：`forkAt(seq)` 已经交付，且已挂载为 turn 尾部的分支操作（`packages/client/ui-conversation/src/client/apply.ts:417`、`chat/TurnTailNodeView.tsx:45`）。

探索上限按"插件内不得硬编码可调项"规则落为可从 `cordis.yml` 修改的、经校验的 `Config` 字段，而非 `DEFAULT_*` 常量；探索默认关闭，因为它会成倍放大成本。

## Alternatives considered

**动态 Cordis 插件。** 作为交付形态被否决：新增会话事件会在下次加载时被拒绝，`RpcMethodMap` 不接受合并，且 conversation 视图的修复位于既有 package 内部。仅保留为 T0 中不增事件、不增 RPC 那部分的原型验证路径。

**用一个合并的 options 对象同时承载 `surfaceOp`、`sourceEventSeqs` 与 `ignorable`。** 在 `append` 写入面上被否决：它会允许调用方把产生消息的事件标记为 ignorable，而 surface 事件永远是被识别的、模型可见的类型，读取方跳过它会静默丢失模型上下文。两个互斥类型使该写法不可表达，而非仅仅不被鼓励，同时也拒绝仅日志事件声明 surface 位置。另一个候选是增加第四个位置参数；它能保持同样的保证，但会让 `append` 有两个 options 参数，其合法组合只能靠隐式约定。

**为锚点到尾部的遮蔽新增 `SurfaceOp` 变体**（上一版的计划）。被否决：该能力已存在，三键的 `isReplaceOp` 校验会拒绝新增字段，且 `SurfaceOp` 改动落在版本 bump 规则之内——为无所得付出持久化格式代价。

**以"隐形 fork"替代原地回滚** —— 内部 fork 并切换可见的会话卡片。在确认真 replace 语义可用后被否决：它留下的分支血缘会误述用户的实际操作，而其唯一理由是那个后来证明并不存在的 surface 约束。

**在回滚点截断日志。** 被否决：日志只可追加且审计完整，截断会摧毁 agent 实际行为的记录——正是回滚控制台要用来审查的证据。

**在审批处改写工具参数。** 被否决的理由是不可实现而非不可取：`arguments` 为 `readonly` 且调用身份按契约不可变。以纠正性反馈 block 让模型重发调用，可保持 `tool/call` 与实际执行一致。

**为血缘、统计、导出建立专用服务。** 被否决，改为消费已实现这些能力的 `ctx.sessionQuery`、`session-stats` 与 `session-log-export`；重复实现会增加只有单一内部调用者的公开方法。

**把 `trajectory/rewind` 标为 ignorable** 以便旧构建仍能加载回滚过的会话。被否决：无法识别的 rewind 标记会让旧读取器把被遮蔽的历史重建为普通上下文，产生错误的模型请求。显式拒绝才是正确行为。

## Acceptance criteria

- `Session.append` 能将事件标记为 `ignorable`，旧构建跳过此类事件，同时仍拒绝未知的 required 事件。
- 来源非 compaction 插件的 `user/message` surface replacement 在对话视图中渲染出历史被遮蔽的标记，而非空白。
- 暂停运行中的 agent 使其变为 `idle` 且排队工作完好，记录一条 `trajectory/pause`，恢复时不伪造空指令。
- 轨迹行操作在该行边界处 fork、插入所给指令、并在子会话中继续；父轨迹保持完整。
- 原地回滚追加一条 replacement `user/message` 加 `trajectory/rewind`，模型上下文推导为锚点前缀加插入内容，被遮蔽事件经 `sessionQuery.listEvents` 仍可读为 `shadowed`。
- 每项新注册均可释放：dispose fiber 后注册被移除。
- 覆盖率与所涉面匹配，遵循仓库测试策略：边界与 surface 解析的单元测试、通过 Loader 启动测试用 `cordis.yml` 的非单元真实 composition 测试、fork-and-steer 与 rewind 转录的 keyless 快照，以及每项用户可见 GUI 变更从该 PR 自身服务录制的 GIF。

## Risks

**`ignorable` 写入面触及 `packages/core/session`**，每条 append 路径都依赖它。该改动是追加式的：options 参数中 surface 事件的那一支未变，仅日志事件仍可完全不传 options，且没有任何现有调用点移动。具有长期性的部分是两个类型的拆分，这也是它作为独立改动落地、而非夹在功能 PR 内的原因。

**从用户视角看原地回滚不可逆**，尽管日志保留了一切：回滚后模型上下文不再包含被遮蔽的 turn。被遮蔽事件仍可审计，这正是该方案可接受的原因。

**回滚不撤销世界状态。** 回滚点之前写入的文件、启动的进程、发出的网络调用依然存在。fork 与 rewind 的确认必须展示该区间的副作用清单，并明示这些不会被撤销；探索对每条变体重复该警告。沙箱 checkpoint/restore 会改变这一点，但不在范围内。

**并发回滚争用 surface 尾部。** 基于过期尾部计算的第二次回滚会以 `end seq N not found in surface` 失败。按会话串行化，并在失败时重算。

**`sourceEventSeqs` 随回滚距离增长** —— 回滚数十个 turn 会列举数百个 seq。正确但体积大；UI 不得渲染该数组。

**分支数量无界增长。** 血缘折叠与归档（隐藏而非删除）可保持面板可用。

**探索会成倍放大模型成本。** 可配置的变体数、轮数与 token 上限加上默认关闭的开关对其加以约束；统计面板按次报告消耗。
