# Agent Note: Trajectory control — pause, branch rollback, and in-place rewind

Status: proposed

English | [中文](2026-08-17-trajectory-control-and-in-place-rewind.zh.md)

## Problem

The Trajectory view is a passive replay of a session's event stream. A user watching an agent take a wrong turn has no way to act on what they see: they cannot stop it at the step they are looking at, cannot re-enter the conversation at an earlier decision point, and cannot compare what a different instruction would have produced. The only available controls are session-wide — cancel the whole turn, or start over.

The requested capability is to turn that view into a console: pause a running agent, roll back to a chosen point (either by branching or by rewriting the session in place), insert new conversation there, and drive the harness onward from that point. The goal is control over the model's output by controlling each intermediate step it sees.

Three properties of the existing runtime constrain any solution:

- **The session log is append-only**, and the `Model-visible ⟺ logged` invariant means anything that reaches a model request must be reconstructable from that log. Rollback cannot truncate, and cannot be an in-memory edit.
- **The world is not rollbackable.** Tool side effects (file writes, processes, network) have already happened. Rolling back the conversation does not undo them.
- **Pause granularity is bounded by loop boundaries.** An agent can be stopped at a turn or pre-step boundary, not frozen mid-tool-execution.

A prior revision of this design (kept as `trajectory-control-plugin/DESIGN.md` before this note existed) reached the right feature set but misjudged three load-bearing facts about the runtime, in both directions: it treated a non-existent core constraint as the project's highest risk, and it missed two real blockers entirely. Recording the verified position is most of this note's value.

## Proposal

Deliver trajectory control as **in-repository packages**, not as a dynamic Cordis plugin, in five stages: a two-item prerequisite stage (T-1), then T0–T3 as feature tiers.

### Delivery form: why in-repository packages

A dynamic Cordis plugin cannot carry this feature, for three independent reasons:

- **New session events.** The persistence read path refuses a log containing an event type this build does not know unless the event is marked `ignorable` (`packages/session/session-persistence/src/coordinator.ts:1063`). That vocabulary is the generated `KNOWN_SESSION_EVENT_TYPES` (`packages/core/session/src/known-event-types.ts:19`), scanned from in-repository `SessionEventMap` merges; its own JSDoc states that out-of-repo plugin events are outside the list by construction.
- **New RPCs.** `RpcMethodMap` (`packages/host/apiproxy/src/api/rpc-map.ts:24`) is a closed interface with no declaration-merge point.
- **Client rendering changes.** The conversation view's replacement handling lives inside `packages/client/ui-conversation` (see the T-1 fallback item below).

The [session-log version mechanism note](https://github.com/deepseek-ai/deepseek-harness/blob/main/.agents/notes/implemented/architecture/2026-08-10-session-log-version-mechanism.md) already records and accepts this: out-of-repo plugin events refuse resume under first-party readers, and the refusal is loud rather than silent. A dynamic plugin remains useful for prototyping the T0 subset that needs no new event and no new RPC.

### T-1: two prerequisites, each an independent change

**1. A writer surface for `ignorable` on `Session.append`.** The field exists on the event envelope and is honored by seed validation, both persistence backends, and the BFF wire schema, but before this change no writer could set it: `append`'s only optional parameter was `SurfaceIntent`, and the envelope was constructed without the field. Every new event type was therefore required-on-read, and an older build refused the whole session on encountering one.

The version-mechanism note anticipates this exact change: *"writers do not yet set `ignorable` (no producer needs it), so `Session.append` gains that surface with its first user."* Trajectory control is that first user.

**This item has shipped.** `SurfaceIntent` was the wrong carrier — it is reachable only for the three `SurfaceEventType` variants, because `append` makes its options parameter conditional on the event type, so a non-surface event like `trajectory/annotation` could never pass a value through it. The options parameter now selects one of two disjoint types by event class: a `SurfaceEventType` requires `SurfaceIntent` as before, and every other type accepts an optional `LogIntent` carrying `ignorable`.

Disjoint types rather than one merged object make two mistakes unrepresentable instead of merely discouraged: the compiler rejects a surface placement on a log-only event, and rejects an `ignorable` marker on a message-producing one. The second is the load-bearing half — a surface event is always a recognized, model-visible type, so skipping one could never be safe, and a shared options object would have left that expressible.

`SESSION_FORMAT_VERSION` stays at `0`: the field was already part of the envelope, and this change only supplies the missing producer.

**2. A fallback renderer for unrecognized surface replacements.** The conversation view identifies replacements by hardcoded plugin name:

```text
// packages/client/ui-conversation/src/client/conversation-nodes/message.ts:25-28
if (event.type !== 'user/message' || !isReplacementSurfaceEvent(event)) return false
const source = event.data.source
return source.kind === 'plugin' && source.plugin === 'compact'
```

`command.ts:83` applies the same `plugin === COMPACT_PLUGIN` test, and the generic fallback matches only `isAppendSurfaceEvent` (`conversation-nodes/fallback.ts:19`). A `user/message` replacement from any other producer is therefore claimed by no definition and caught by no fallback: **it renders as nothing.** In-place rewind produces exactly that event shape.

**This item has shipped**, and it landed as an extension of the existing fallback rather than a second one: `ConversationEventRegistry.registerFallback` admits exactly one fallback and throws on a duplicate, and the assembler consults it only when no ordinary Definition claimed the target (`packages/client/runtime/src/client/conversation/event-registry.ts:34-50`, `sessions/conversation-assembler.ts:376-383`). So `unknownFallbackDefinition` now matches both surface origins, and `UnknownSurfaceNode` carries an optional `replaced` list of shadowed seqs, taken from `sourceEventSeqs` — a superset of the shadowed nodes, since core requires that field to include every shadowed node while a producer may cite further sources.

The node reports shadowed history rather than omitting content, because `isAppendSurfaceEvent` exists precisely so a landed replacement does not erase a transcript the user already read (`packages/core/session/src/surface.ts:44-47`). The Trajectory view needs no counterpart: its message Definition matches `user/message` regardless of surface origin, so a replacement was never dropped there.

Append-origin fallback remains unreachable through an assembled window — all three `SurfaceEventType` members have owning Definitions — so that arm is covered at the Definition level and stays a degradation path for a core-side widening of the type set.

### In-place rewind needs no surface-mechanism change

The prior revision claimed the replace range is constrained to lie "between two existing nodes" and that core must be extended to allow shadowing "from the anchor to the end of the log," rating this the project's highest risk. **That constraint does not exist.** Validation requires only that `start` and `end` are both present in the *current* surface and ordered by position (`packages/core/session/src/surface.ts:246-266`). Because the replacing event is appended at the tail, the last surface node at validation time is the tail, so "anchor through tail" is expressible today. Two in-tree precedents do it: `packages/host/apiproxy/tests/api-proxy-models.spec.ts:221` uses `{ op: 'replace', start: 0, end: events.length - 1 }`, and `packages/compaction/compaction/tests/tool-pairing.spec.ts:171` uses `end: nodes.at(-1)!`.

Following the prior revision would have been actively harmful: `isReplaceOp` requires exactly three keys (`surface.ts:175`), so any added field is rejected at append — and `SurfaceOp` variants are named in the `SESSION_FORMAT_VERSION` bump rule (`types.ts:41-52`), so the change would have cost a version bump to obtain a capability that already exists.

The four real constraints, none of which the prior revision identified:

1. `end` must be a **surface** node, not the last log event. A tail of `turn/end`, `step/end`, or a chunk fails with `end seq N not found in surface`; the caller must resolve to the last surface node.
2. `sourceEventSeqs` must enumerate **every** shadowed node (`surface.ts:239`), each strictly earlier than the event (`surface.ts:235`). This makes the replacement `user/message` the carrier of shadowing information, not the `trajectory/rewind` payload.
3. The range cannot be expressed as a `tool/result`: those replacements are restricted to one node, must target a `tool/result`, and may change only content (`surface.ts:287-318`). Use `user/message`.
4. Ordering is positional on the surface, not by seq value.

Because real replace semantics are available, the prior revision's "invisible fork" transition is dropped: it would leave misleading branch lineage for no benefit.

### Event vocabulary

One required event and two ignorable ones. `trajectory/branch` and `trajectory/explore` from the prior revision are dropped — lineage is reconstructable from `header.parentSession`, and exploration outcomes fit in an annotation.

- `trajectory/rewind` (**required**) — `{ anchorSeq, replacementSeq }`. Marks that model context re-derives from the anchor, distinguishing a rollback from compaction's structurally identical replacement. An older build refuses to load a session containing it, which is the intended behavior: reconstructing a shadowed history as an ordinary log would yield wrong model context.
- `trajectory/annotation` (**ignorable**) — anchored note; `visibleToModel` injects it at `agent/pre-step`.
- `trajectory/pause` (**ignorable**) — pause audit record.

Adding ordinary event types does not bump `SESSION_FORMAT_VERSION`; the `ignorable` guard covers vocabulary growth.

### Reuse rather than rebuild

`ctx.sessionQuery` and existing projections already provide read models the prior revision planned to build: `traceSession()` for lineage (ancestors plus recursive descendants, with a `complete: false` marker and cycle detection), `traceEvent()` for `replacementChain`, `listEvents()` for `current | shadowed | log-only` classification, `session-stats` for turn/step counts and phase wall times, and `session-log-export` for ZIP/JSONL export. Trajectory control consumes these; it does not define competing services, which also avoids public service methods with a single internal caller.

Two real gaps in that reuse path, each worth an independent fix: `parent_session` carries no SQL index (`packages/session-query/session-query-sqlite/src/schema.ts` declares only `id TEXT PRIMARY KEY`), and `listChildren` additionally requires `origin === 'subagent'` (`packages/subagent/subagent/src/list-children.ts:141`), which excludes ordinary fork children a lineage panel must show.

### Corrections to the runtime model

The prior revision's flows would not compile or would misbehave against these verified facts:

- **No `paused` status.** `AgentStatus = 'idle' | 'running'` (`packages/core/agent/src/runtime-types.ts:50`). Pause is `idle` plus a plugin-owned marker.
- **`AgentCancelCause` is a closed union** — `user | parent | hook | disposed` (`packages/core/session/src/types.ts:143`). A `{ kind: 'user-pause' }` cause is invalid; `{ kind: 'hook', reason: 'trajectory-pause' }` carries the audit reason.
- **`agent.steering()` does not exist.** Steering is `InboxTarget = 'next-step'` (`packages/core/agent/src/types.ts:10`) passed to `agent.send`.
- **`session.fork` already builds a live child agent** and inherits the preset (`packages/host/apiproxy/src/api-proxy.ts:2420`), so a separate resume step is unnecessary. Boundary snapping to the next `turn/end` lives in that RPC (`:2382`); `SessionStore.fork` does not snap and rejects a prefix ending inside an open turn with `OPEN_TURN` (`packages/core/session/src/index.ts:1128`). The rule is "must not end inside an open turn," not "must land on a `turn/end`": a log-only event after a closed turn is a valid boundary.
- **Preset resolution reads the log, not the header** (`api-proxy.ts:1258`), because a session that switched composition while blank ran its turns under the newer one while the header is written once at creation.
- **`turn/end.reason` is a structured value**, and `blocked` and `max-tokens` belong in automatic-pause rules alongside `error`. `agent/turn-stopping` is `@mode serial` and cannot veto stopping.
- **Tool arguments cannot be rewritten.** `ToolExecutionInput.arguments` is `readonly`, and `tools/execute` states that call identity remains immutable (`packages/core/tools/src/index.ts:154`). Editing instructions at an approval prompt is feasible through pre-step rewriting; editing tool arguments is not, and the correct alternative is a `tools/pre-execute` block with corrective feedback so the model reissues the call and the log stays truthful.

Pre-step "park" — holding a step while a human decides — is feasible: `agent/pre-step` is a waterfall with async listeners and a signal (`runtime-types.ts:231`), `userQuestions.ask()` accepts a signal and awaits a human, and the tool-call timeout policy is scoped to `tools/execute` and cannot kill a parked step. `plan-mode` and `tool-cordis` already do asynchronous work and message injection at that boundary.

### Staging

| Stage | Content |
|---|---|
| **T-1** | `ignorable` writer surface (**shipped**); replacement fallback renderer (**shipped**) |
| **T0** | Live pause/resume; trajectory row actions reusing the existing `forkAt`; steer/queue insertion form; pause banner |
| **T1** | Annotations; lineage panel over `traceSession`; in-place rewind; instruction-only approval editing |
| **T2** | Breakpoint park; automatic error pause; side-effect inventory; trajectory diff; branch exploration |
| **T3** | Export/import/replay over `session-log-export`; stats over `session-stats`; subagent inlining; goal coupling |

T0 is smaller than the prior revision assumed: `forkAt(seq)` already ships and is already mounted as a turn-tail branch action (`packages/client/ui-conversation/src/client/apply.ts:417`, `chat/TurnTailNodeView.tsx:45`).

Exploration limits are validated `Config` fields changeable from `cordis.yml`, not `DEFAULT_*` constants, per the no-hardcoded-tunables rule; exploration defaults to disabled because it multiplies cost.

## Alternatives considered

**A dynamic Cordis plugin.** Rejected as the delivery form: new session events would be refused on the next load, `RpcMethodMap` admits no merge, and the conversation-view fix is inside an existing package. Retained only as a prototyping path for the T0 subset that adds no event and no RPC.

**One merged options object carrying `surfaceOp`, `sourceEventSeqs`, and `ignorable`.** Rejected for the `append` writer surface: it would let a caller mark a message-producing event ignorable, and a surface event is always a recognized model-visible type, so a reader skipping one would silently drop model context. Two disjoint types make that unrepresentable instead of merely discouraged, and they also reject a surface placement on a log-only event. A fourth positional parameter was the other candidate; it keeps the same guarantee but leaves `append` with two options parameters whose valid combinations are implicit.

**A new `SurfaceOp` variant for anchor-to-tail shadowing** (the prior revision's plan). Rejected because the capability already exists, the three-key `isReplaceOp` check rejects added fields, and `SurfaceOp` changes fall under the version-bump rule — paying a durable-format cost for nothing.

**"Invisible fork" as an in-place-rollback stand-in** — fork internally and swap the visible session card. Rejected once real replace semantics were confirmed available: it leaves branch lineage that misrepresents what the user did, and it was only ever justified by the surface constraint that turned out not to exist.

**Truncating the log at the rollback point.** Rejected: the log is append-only and audit-complete, and truncation would destroy the record of what the agent actually did — the evidence a rollback console exists to inspect.

**Rewriting tool arguments at an approval prompt.** Rejected as impossible rather than undesirable: `arguments` is `readonly` and call identity is immutable by contract. Blocking with corrective feedback so the model reissues the call keeps `tool/call` and execution consistent.

**Dedicated services for lineage, statistics, and export.** Rejected in favor of consuming `ctx.sessionQuery`, `session-stats`, and `session-log-export`, which already implement them; duplicating would add public methods with one internal caller.

**Marking `trajectory/rewind` ignorable** so older builds keep loading rewound sessions. Rejected: an unrecognized rewind marker would let an older reader rebuild shadowed history as ordinary context, producing a wrong model request. Loud refusal is correct.

## Acceptance criteria

- `Session.append` can mark an event `ignorable`, and an older build skips such an event while still refusing an unknown required one.
- A `user/message` surface replacement whose source is not the compaction plugin renders a shadowed-history marker in the chat view instead of nothing.
- Pausing a running agent leaves it `idle` with queued work intact, records one `trajectory/pause`, and resumes without fabricating an empty instruction.
- A trajectory row action forks at that row's boundary, inserts the supplied instruction, and continues in the child session; the parent trajectory remains intact.
- An in-place rewind appends a replacement `user/message` plus `trajectory/rewind`, derives model context as anchor prefix plus insertion, and leaves shadowed events readable as `shadowed` through `sessionQuery.listEvents`.
- Every new registration is disposable: disposing the fiber removes it.
- Coverage matches the surfaces per repository testing policy: unit tests for boundary and surface resolution, a non-unit real-composition test booting a test `cordis.yml` through the Loader, keyless snapshots for the fork-and-steer and rewind transcripts, and a GIF recorded from the PR's own server for each user-visible GUI change.

## Risks

**The `ignorable` writer surface touches `packages/core/session`,** which every append path depends on. It is additive: the surface-event arm of the options parameter is unchanged, a log-only event still accepts no options at all, and no existing call site moved. The durable part is the two-type split, which is why it landed as its own change rather than inside a feature PR.

**In-place rewind is irreversible from the user's perspective** even though the log retains everything: after a rewind, model context no longer contains the shadowed turns. The shadowed events stay auditable, which is what makes this acceptable.

**Rollback does not undo world state.** Files written, processes started, and network calls made before the rollback point remain. Fork and rewind confirmation must show the side-effect inventory for the range and state plainly that these are not reverted; exploration repeats the warning per variant. Sandbox checkpoint/restore would change this, and is out of scope.

**Concurrent rewinds contend for the surface tail.** A second rewind computed against a stale tail fails with `end seq N not found in surface`. Serialize per session and recompute on failure.

**`sourceEventSeqs` grows with rollback distance** — rolling back dozens of turns enumerates hundreds of seqs. Correct but bulky; the UI must not render the array.

**Branch count grows without bound.** Lineage folding and archiving (hide, never delete) keep the panel usable.

**Exploration multiplies model cost.** Configurable variant, round, and token ceilings plus a default-off switch bound it; the stats panel reports consumption per exploration.
