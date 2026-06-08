# Study Guide: `packages/agent` (`@earendil-works/pi-agent-core`)

> The agent runtime: tool calling + conversation state management, built on top of
> `@earendil-works/pi-ai`. Version 0.79.0. Node >= 22.19.0. All file paths below are
> absolute under `/home/user/pi/packages/agent`.

This document teaches the package from three layers, bottom to top:

1. **Low-level loop** (`src/agent-loop.ts`) — pure function that runs turns over an
   `AgentContext` and emits `AgentEvent`s. No persistence, no internal state.
2. **`Agent` class** (`src/agent.ts`) — stateful, in-memory wrapper that owns a transcript,
   queues (steering/follow-up), abort, and event subscription.
3. **`AgentHarness`** (`src/harness/agent-harness.ts`) — the durable orchestration layer:
   persistent sessions (a branching tree), compaction, tree navigation/branch summaries,
   typed hooks, an execution-environment abstraction, skills, and prompt templates.

The harness does **not** use the `Agent` class. It calls `runAgentLoop()` directly
(`src/harness/agent-harness.ts:587`) and reimplements its own lifecycle. This was a deliberate
decision (see `docs/agent-harness.md` "Remove `Agent` dependency", marked Done).

---

## 1. Purpose & public API

### How it sits on `packages/ai`

`pi-ai` provides the provider/transport layer: `streamSimple` (streaming chat),
`completeSimple` (one-shot completion), `Model<T>`, `Message` (`user` | `assistant` |
`toolResult`), `AssistantMessageEvent` (streaming deltas), `EventStream`, `Tool`,
`validateToolArguments`, `Transport`, `Usage`. `pi-agent-core` adds the **loop, state, tools
beyond the bare `Tool` shape, and durability** on top of those primitives.

The package has two entry points (`package.json:8-18`):

- `.` → `src/index.ts` — browser-safe, exports everything except Node-only modules.
- `./node` → `src/node.ts` — re-exports `index.ts` **plus** `NodeExecutionEnv`
  (`src/node.ts:1-2`). Node-only code is quarantined to `src/harness/env/nodejs.ts` so the
  root import never pulls `node:*` modules.

`src/index.ts` re-exports: the `Agent` class + loop fns (`agent.ts`, `agent-loop.ts`), the
harness (`agent-harness.ts`, `types.ts`), compaction + branch summarization, session
repos/storage/`Session`, `uuidv7`, skills, system-prompt formatting, prompt templates,
shell-output + truncate utils, the `streamProxy` proxy stream fn, and all core types.

### The key public surfaces

- **`Agent`** (`src/agent.ts:166`) — `prompt()`, `continue()`, `steer()`, `followUp()`,
  `abort()`, `waitForIdle()`, `reset()`, `subscribe()`, `state` getter.
- **`agentLoop` / `agentLoopContinue`** (`src/agent-loop.ts:31,64`) — return an
  `EventStream<AgentEvent, AgentMessage[]>`; thin wrappers over the `run*` variants.
- **`runAgentLoop` / `runAgentLoopContinue`** (`src/agent-loop.ts:95,120`) — the async
  generators-with-callback form used by both `Agent` and `AgentHarness`.
- **`AgentHarness`** (`src/harness/agent-harness.ts:174`) — the durable, session-backed
  orchestrator.

---

## 2. The harness loop: how a turn runs

This section describes the **core loop** in `src/agent-loop.ts` (shared by `Agent` and
`AgentHarness`). The vocabulary:

- **turn** = one assistant response + the tool calls it requested + their results.
- **run** = everything between `agent_start` and `agent_end` for a single `prompt()`/`continue()`.

### Entry: prompt vs continue

`runAgentLoop(prompts, context, config, emit, signal, streamFn)` (`:95`):
- appends `prompts` to `context.messages`, emits `agent_start`, `turn_start`, then
  `message_start`/`message_end` for each prompt message, then enters `runLoop`.

`runAgentLoopContinue(context, ...)` (`:120`) resumes with **no new message** (used for
retries / draining queues when the last message is `assistant`). It throws if the transcript
is empty or the last message is `assistant` (`:127-133`) — the provider would reject a request
that ends on an assistant message.

### `runLoop` (`src/agent-loop.ts:155`) — the engine

Two nested loops:

```
outer: while (true)                 // restarts when follow-up messages arrive
  inner: while (hasMoreToolCalls || pendingMessages.length > 0)
    - emit turn_start (except the very first turn)
    - inject pendingMessages (steering/follow-up) as user messages
    - message = streamAssistantResponse(...)        // one LLM call
    - if stopReason error/aborted -> turn_end + agent_end, RETURN
    - toolCalls = message.content where type==="toolCall"
    - if toolCalls: executeToolCalls(...) -> toolResults; hasMoreToolCalls = !terminate
      push each toolResult into context + newMessages
    - emit turn_end { message, toolResults }
    - prepareNextTurn?() -> may swap context/model/thinkingLevel for next turn
    - shouldStopAfterTurn?() -> if true: agent_end, RETURN
    - pendingMessages = getSteeringMessages() ?? []
  // inner exits when no tool calls and no steering
  followUpMessages = getFollowUpMessages() ?? []
  if followUpMessages: pendingMessages = followUpMessages; continue outer
  else break
emit agent_end
```

Key invariants:
- `hasMoreToolCalls` drives continuation. After a tool batch the loop **always** runs another
  turn so the model can react to results — unless every result set `terminate: true`
  (`:210`, `shouldTerminateToolBatch` `:544`).
- Steering is polled **at the start of the run** (`:167`, "user typed while waiting") and again
  **after each turn** (`:253`). Follow-up is polled only when the inner loop fully drains
  (`:257`).
- `agent_end` is emitted at exactly one of several exit points; `newMessages` (only the
  messages this invocation produced) is its payload.

### `streamAssistantResponse` (`src/agent-loop.ts:275`) — the LLM-call boundary

This is the **only place `AgentMessage[]` becomes `Message[]`**:

1. `transformContext?(messages, signal)` — optional `AgentMessage[]→AgentMessage[]` (pruning,
   external context injection). Must not throw.
2. `convertToLlm(messages)` — required `AgentMessage[]→Message[]`. Filters UI-only messages,
   maps custom roles to `user`/`assistant`/`toolResult`.
3. Build `Context { systemPrompt, messages, tools }`, resolve API key via `getApiKey(provider)`
   (important for expiring OAuth tokens — `:301`), call `streamFn(model, context, options)`.
4. Consume the `AssistantMessageEventStream`:
   - `start` → push partial to context, emit `message_start`.
   - text/thinking/toolcall deltas → replace last context message, emit `message_update` with
     the raw `assistantMessageEvent` (this is how UIs stream chunks).
   - `done`/`error` → `await response.result()` for the final message, replace partial, emit
     `message_end`, return.

Each emitted message object is a **shallow clone** (`{ ...partialMessage }`, `:319`) so
subscribers never alias the mutating partial.

### Tool dispatch (`executeToolCalls`, `src/agent-loop.ts:373`)

Mode selection (`:380-388`): if global `toolExecution === "sequential"` **or any tool in the
batch declares `executionMode: "sequential"`**, the **entire batch** runs sequentially. This is
the "poison pill" rule: one sequential tool forces serial execution for that assistant message.

Each tool call goes through a pipeline of small typed states:

- **`prepareToolCall`** (`:562`): find the tool (missing → immediate error result `:571`);
  `prepareArguments?()` shim (`:548`) then `validateToolArguments` against the TypeBox schema;
  run `beforeToolCall` hook (can `{ block: true, reason }` → immediate error result `:598`);
  honor abort at each checkpoint. Returns `PreparedToolCall` or `ImmediateToolCallOutcome`.
- **`executePreparedToolCall`** (`:628`): call `tool.execute(id, args, signal, onUpdate)`.
  `onUpdate` pushes `tool_execution_update` events (awaited via `updateEvents` array `:633`,
  `:654`). **Thrown errors are caught and turned into error tool results** (`:656-662`) — the
  documented contract is "throw on failure; do not encode errors in content."
- **`finalizeExecutedToolCall`** (`:665`): run `afterToolCall` hook. Field-by-field override
  merge of `content`/`details`/`isError`/`terminate` (`:689-696`); **no deep merge**. A throw
  here also becomes an error result.

**Sequential** (`:395`): for each call, emit `tool_execution_start`, prepare, execute,
finalize, emit `tool_execution_end`, then emit the `toolResult` `message_start`/`message_end`.
Breaks early on abort (`:440`).

**Parallel** (`:451`): preflight (`tool_execution_start` + prepare) runs **sequentially in
source order**; non-immediate calls are pushed as thunks; then `Promise.all` runs them
concurrently (`:502`). `tool_execution_end` fires in **completion order** (inside each thunk
`:494`), but the persisted `toolResult` messages are emitted afterward in **assistant source
order** (`:505-510`). This ordering split is explicit in the README.

Tool result message shape: `createToolResultMessage` (`:727`) → `{ role: "toolResult",
toolCallId, toolName, content, details, isError, timestamp }`.

### Message lifecycle summary (events)

`AgentEvent` union (`src/types.ts:403-418`):
`agent_start` → `turn_start` → (`message_start`/`message_update`*/`message_end` for assistant) →
(`tool_execution_start`/`tool_execution_update`*/`tool_execution_end`, then `message_start`/
`message_end` for each toolResult) → `turn_end {message, toolResults}` → … → `agent_end {messages}`.
`message_update` is **assistant-only** and carries `assistantMessageEvent`.

---

## 3. State management & sessions (`src/harness/session/*`)

### Two state models

- **`Agent.state`** (`src/agent.ts:241`, `AgentState` `src/types.ts:317`): in-memory only. A
  flat `messages: AgentMessage[]`, plus `systemPrompt`, `model`, `thinkingLevel`, `tools`, and
  read-only `isStreaming` / `streamingMessage` / `pendingToolCalls` / `errorMessage`. `tools`
  and `messages` are accessor properties whose **setters copy the top-level array**
  (`src/agent.ts:79-87`) — but mutating the returned array mutates live state. Events are
  reduced into state in `processEvents` (`:509`): `message_end` pushes to `messages`,
  `tool_execution_start/end` track `pendingToolCalls`, etc.

- **Session (harness)**: a **persistent, append-only, branching tree** of `SessionTreeEntry`.
  This is the durable model. The flat transcript is *derived* from a path through the tree.

### Session tree (`src/harness/session/session.ts`)

A `Session<TMetadata>` wraps a `SessionStorage` (`src/harness/types.ts:440`). Entry types
(`SessionTreeEntry` union, `src/harness/types.ts:409`), all sharing `{ type, id, parentId,
timestamp }` (`SessionTreeEntryBase:334`):

| Entry | Purpose |
|---|---|
| `message` (`:341`) | An `AgentMessage`. The transcript backbone. |
| `thinking_level_change` / `model_change` / `active_tools_change` | Branch-scoped config history. |
| `compaction` (`:362`) | Summary + `firstKeptEntryId` + `tokensBefore` + file-op `details`. |
| `branch_summary` (`:371`) | Summary of an abandoned branch you navigated back from. |
| `custom` / `custom_message` (`:379,385`) | App data; `custom_message` becomes a context message. |
| `label` (`:393`) | Human label pointing at a target entry (latest wins). |
| `session_info` (`:399`) | Session name (legacy type name). |
| `leaf` (`:404`) | **Durable cursor**: records the active tree leaf (`targetId`, or `null` for root). |

`Session` append methods (`appendMessage`, `appendModelChange`, `appendCompaction`,
`appendLabel`, `appendSessionName`, etc., `:132-244`) each generate an id, set `parentId` to the
current leaf, persist, and advance the leaf. `moveTo(entryId, summary?)` (`:246`) sets the leaf
to an arbitrary entry (tree navigation) and optionally appends a `branch_summary`.

### Deriving the transcript: `buildSessionContext` (`src/harness/session/session.ts:22`)

Given the path from leaf to root (`getPathToRoot`), it folds entries into a `SessionContext`
(`src/harness/types.ts:422`): `{ messages, thinkingLevel, model, activeToolNames }`. Crucially:

- `thinkingLevel`/`model`/`activeToolNames` are reduced from the **last** matching config entry
  (assistant messages also set the "current model", `:33`).
- **Compaction handling** (`:61-72`): if a `compaction` entry exists on the path, the derived
  message list is `[compactionSummaryMessage] + entries from firstKeptEntryId..compaction] +
  entries after compaction]`. Everything before `firstKeptEntryId` is dropped from context but
  remains in the tree. This is how compaction shrinks context without losing history.
- `branch_summary` and `custom_message` entries are turned into synthetic messages
  (`createBranchSummaryMessage` / `createCustomMessage`, `src/harness/messages.ts:81,103`).

### Storage backends & repos

- **`SessionStorage`** interface (`src/harness/types.ts:440`): `getLeafId`/`setLeafId`,
  `createEntryId`, `appendEntry`, `getEntry`, `findEntries`, `getLabel`, `getPathToRoot`,
  `getEntries`, `getMetadata`.
- **`InMemorySessionStorage`** (`src/harness/session/memory-storage.ts:40`) — maps + arrays,
  for tests and ephemeral use.
- **`JsonlSessionStorage`** (`src/harness/session/jsonl-storage.ts:161`) — durable
  **JSONL-on-disk**. First line is a `SessionHeader` (`type:"session", version:3, id, timestamp,
  cwd, parentSession?`); each subsequent line is one entry. Appends via `fs.appendFile`; full
  load via `loadJsonlStorage` (`:136`); cheap metadata-only read via `loadJsonlSessionMetadata`
  reading just line 1 (`:123`, `readTextLines(maxLines:1)`).
- **Leaf reconstruction on reopen**: `leafIdAfterEntry` (`:109`) returns `entry.targetId` for
  `leaf` entries else `entry.id`; replaying entries reconstructs `currentLeafId`. This is why
  `setLeafId` must persist a durable `leaf` entry, not just update memory
  (`docs/agent-harness.md` "Session storage implementations must persist leaf changes").
- **Repos** (`SessionRepo`, `src/harness/types.ts:468`): `create`/`open`/`list`/`delete`/`fork`.
  `JsonlSessionRepo` (`src/harness/session/jsonl-repo.ts:38`) stores sessions under
  `sessionsRoot/--<encoded-cwd>--/<timestamp>_<id>.jsonl`. `InMemorySessionRepo`
  (`memory-repo.ts`) keeps a `Map`. `fork` (`repo-utils.ts:32`) copies entries up to a target
  (`position: "before"|"at"`) into a fresh session — used to branch a conversation into its own
  session file.
- **IDs**: `uuidv7()` (`src/harness/session/uuid.ts:15`) — monotonic time-ordered UUIDv7 with a
  sequence counter; entry ids use the first 8 hex chars (`generateEntryId`, retries on collision).

### The "durable harness" concept (`docs/durable-harness.md`)

Full durability is impossible because tool implementations, model/auth providers, hook handlers,
resource loaders, and system-prompt callbacks are **runtime JS supplied by the host app** and
cannot be serialized. The realistic target is **semi-durable**: the *session* is the single
append-only source of truth for durable agent state (config changes, leaf, compactions, summaries,
messages), and on resume the **app re-provides** the non-serializable dependencies. Recovery
restarts from durable boundaries — **provider streams are not resumable**, and unfinished
non-idempotent tool calls are unsafe to retry. A future `AgentHarness.builder().restore()` and
durable queue/pending-write/operation entries are designed but **not yet implemented**.

---

## 4. Compaction (`src/harness/compaction/*`)

Compaction summarizes old history into a `compaction` entry so the live context fits the model's
window, while the full tree is preserved on disk.

### When (thresholds)

`DEFAULT_COMPACTION_SETTINGS` (`compaction.ts:112`): `{ enabled, reserveTokens: 16384,
keepRecentTokens: 20000 }`. `shouldCompact(contextTokens, contextWindow, settings)` (`:196`)
returns `contextTokens > contextWindow - reserveTokens`. **Note:** the harness's `compact()`
method (`agent-harness.ts:708`) is **manual** — `docs/agent-harness.md` states "Auto-compaction
decision point is not implemented in `AgentHarness` yet." Token estimates use real provider
`Usage` when available (`estimateContextTokens`, `:165`, uses the last successful assistant
`usage` + char-heuristic for trailing messages) falling back to `estimateTokens` (`:220`, ~4
chars/token; images ≈ 4800 chars `:201`).

### How (`prepareCompaction` → `compact`)

1. **`prepareCompaction(pathEntries, settings)`** (`compaction.ts:542`): no-op if empty or the
   last entry is already a compaction. Finds the previous compaction (iterative summaries chain
   off `previousSummary`), establishes `boundaryStart` at the prior `firstKeptEntryId`, then
   **`findCutPoint`** (`:329`) walks backward accumulating tokens until `keepRecentTokens` is hit
   and snaps to a valid cut point. Valid cut points (`findValidCutPoints`, `:261`) are
   user/assistant/bashExecution/custom/branchSummary/compactionSummary boundaries — **never
   mid tool-call/tool-result** (a `toolResult` is never a cut point, so an assistant + its tool
   results stay together). Produces `messagesToSummarize`, optional `turnPrefixMessages` (for a
   **split turn** — when the cut lands inside a turn), `firstKeptEntryId`, `tokensBefore`,
   `fileOps`. Returns a `Result` (typed `CompactionError`).
2. **`compact(preparation, model, apiKey, ...)`** (`compaction.ts:627`): calls `generateSummary`
   (`:456`) via `completeSimple` with `SUMMARIZATION_SYSTEM_PROMPT` and a strict structured
   format (Goal / Constraints / Progress / Key Decisions / Next Steps / Critical Context). For
   split turns it also runs `generateTurnPrefixSummary` (`:707`) in parallel and joins them. File
   operations (read/written/edited paths, mined from `read`/`write`/`edit` tool calls via
   `extractFileOpsFromMessage`, `utils.ts:24`) are appended as `<read-files>`/`<modified-files>`
   blocks (`formatFileOperations`, `utils.ts:62`) and stored in the entry's `details`.
3. The harness persists via `session.appendCompaction(...)` (`agent-harness.ts:745`).

`serializeConversation` (`utils.ts:91`) renders LLM messages to plain text for the summary
prompt, truncating each tool result to `TOOL_RESULT_MAX_CHARS = 2000`.

### Branch summarization (`branch-summarization.ts`)

Distinct from compaction: when you `navigateTree` away from a branch, the abandoned branch is
summarized. `collectEntriesForBranchSummary` (`:69`) finds the common ancestor between the old
leaf and the target, collects the entries unique to the old branch, and `generateBranchSummary`
(`:201`) produces a structured summary (own prompt at `:171`) prefixed with `BRANCH_SUMMARY_PREAMBLE`.
`prepareBranchEntries` (`:125`) fits messages within a token budget (`contextWindow - reserveTokens`).

---

## 5. Environment abstraction (`src/harness/env/*`)

`ExecutionEnv` (`src/harness/types.ts:332`) = `FileSystem` (`:268`) + `Shell` (`:321`). It is the
agent's sandbox-able filesystem + process layer, injected into the harness. Hard invariant: **all
`FileSystem`/`Shell` operations must never throw or reject — every failure is returned as a
typed `Result<T, FileError|ExecutionError>`** (`:266`, `Result` at `:5`).

- **`FileSystem`**: `readTextFile`, `readTextLines` (bounded by `maxLines` — used for cheap JSONL
  header reads), `readBinaryFile`, `writeFile`, `appendFile`, `fileInfo`/`listDir` (do **not**
  follow symlinks), `canonicalPath` (explicit symlink resolution), `exists`, `createDir`,
  `remove`, `createTempDir`/`createTempFile`, `absolutePath`/`joinPath`, `cleanup`.
  `FileError` codes: `not_found`, `permission_denied`, `is_directory`, `aborted`, … (`:111`).
- **`Shell.exec`** → `Result<{ stdout, stderr, exitCode }, ExecutionError>` with options
  `cwd`/`env`/`timeout`/`abortSignal`/`onStdout`/`onStderr`. `ExecutionError` codes:
  `aborted`, `timeout`, `shell_unavailable`, `spawn_error`, `callback_error` (`:137`).

`NodeExecutionEnv` (`src/harness/env/nodejs.ts:217`) is the only concrete impl (Node-only,
exported only via `./node`). It maps `node:fs/promises` errno codes (`ENOENT`→`not_found`, etc.,
`toFileError` `:65`), discovers a bash shell cross-platform (`getShellConfig` `:147`), spawns
detached process groups and **kills the whole tree on abort/timeout** (`killProcessTree` `:192`,
`process.kill(-pid)`), and streams stdout/stderr through `onStdout`/`onStderr`. A callback throw
converts to a `callback_error` and aborts the child (`:314-318`).

Built on top: `executeShellWithCapture` (`src/harness/utils/shell-output.ts:43`) wraps
`env.exec`, sanitizes binary bytes, caps in-memory output at `DEFAULT_MAX_BYTES*2`, spills full
output to a temp file once it exceeds 50KB, and tail-truncates (`truncate.ts`). The
truncation utilities (`src/harness/utils/truncate.ts`) implement line-and-byte-limited
head/tail truncation that never emits partial lines (except a tail edge case), with
runtime-neutral UTF-8 byte counting (no hard `Buffer` dependency).

---

## 6. Hooks & observability

### Two distinct event channels on `AgentHarness`

1. **`subscribe(listener)`** (`agent-harness.ts:1038`) — passive observers. They receive **every**
   `AgentHarnessEvent` (both low-level `AgentEvent`s re-emitted via `emitAny`, and harness-own
   events via `emitOwn`). Stored under the wildcard key `"*"` (`SUBSCRIBER_EVENT_TYPE`, `:141`).
   Return value ignored.
2. **`on(type, handler)`** (`agent-harness.ts:1050`) — typed, result-producing **hooks**. The
   handler's return type is fixed by `AgentHarnessEventResultMap` (`types.ts:704`). These can
   mutate behavior. Dispatched via `emitHook` (`:249`) which returns the **last non-undefined**
   result.

A handler error in either channel is normalized to `AgentHarnessError("hook", …)`
(`normalizeHookError`, `:154`) and propagates (state is **not** rolled back after commit —
`docs/agent-harness.md` "Error handling").

### Hook events (`AgentHarnessOwnEvent`, `types.ts:634`) and their result types

| Hook (`on(type)`) | Result | Effect |
|---|---|---|
| `before_agent_start` (`:523`) | `{ messages?, systemPrompt? }` | inject messages / override prompt at turn start (`agent-harness.ts:570`) |
| `context` (`:534`) | `{ messages }` | wired to loop `transformContext` (`:430`) — rewrite context before LLM |
| `before_provider_request` (`:539`) | `{ streamOptions? patch }` | mutate per-request transport/headers/metadata (`:268`) |
| `before_provider_payload` (`:546`) | `{ payload }` | rewrite the raw provider payload (`:294`) |
| `after_provider_response` (`:552`) | — | observe status + headers |
| `tool_call` (`:558`) | `{ block?, reason? }` | wired to loop `beforeToolCall` (`:434`) — block a tool |
| `tool_result` (`:565`) | `ToolResultPatch` | wired to loop `afterToolCall` (`:443`) — patch a result |
| `session_before_compact` (`:575`) | `{ cancel?, compaction? }` | veto/override compaction |
| `session_before_tree` (`:589`) | `{ cancel?, summary?, … }` | veto/override navigation |

Harness-emitted observational events (no result): `queue_update`, `save_point`,
`abort`, `settled`, `session_compact`, `session_tree`, `model_update`,
`thinking_level_update`, `tools_update`, `resources_update`.

The provider request/payload hooks chain as **ordered transforms** (each handler sees the prior
output); stream-option patches support explicit key deletion (`applyStreamOptionsPatch`, `:99`;
`undefined` value deletes a key, `headers: undefined` clears all).

### Designed-but-not-yet-built hook system (`docs/hooks.md`)

A cleaner future model: events carry their result type as a type-only phantom (`HookEvent<Type,
Result>`, `ResultOf<E>`); a single `AgentHarnessHooks` owns registration with `observe()` (all
events, read-only) vs `on(type)` (participates), plus `addCleanup`, `clear`, `dispose`, source-
scoped registration, and explicit reducer semantics per event (transform chain, first-block-wins
for `tool_call`, first-cancel-or-last for `session_before_*`). The current `agent-harness.ts`
implements a simpler hand-rolled version of these reducers inline.

### Observability (`docs/observability.md`) — design only

A planned, **vendor-neutral** tracing contract (`packages/observability`): `traceOperation(name,
payload, fn)` emits `start`/`end`/`error` span events propagated through `AsyncLocalStorage`-style
context, with stable names like `pi.agent.prompt`, `pi.agent.tool_call`,
`pi.ai.provider.request`. Adapters (OTel/Sentry/logs) subscribe; pi never imports them.
Safe-by-default payloads (provider, model, token counts, durations) vs unsafe (prompts,
completions, tool args, file contents). **Not implemented in this package yet.**

---

## 7. Tool definition & registration

### Defining a tool — `AgentTool` (`src/types.ts:361`)

Extends `pi-ai`'s `Tool<TParameters>` with:
- `name`, `description`, `parameters` (a TypeBox `TSchema`) — from `Tool`.
- `label: string` — UI display.
- `execute(toolCallId, params, signal?, onUpdate?) => Promise<AgentToolResult<TDetails>>`
  (`:370`). `params` is the validated, statically typed `Static<TParameters>`.
- `prepareArguments?(args) => Static<TParameters>` (`:368`) — optional shim to coerce raw
  arguments before schema validation (compatibility for sloppy models).
- `executionMode?: "sequential" | "parallel"` (`:383`) — per-tool override.

`AgentToolResult<T>` (`:345`): `{ content: (TextContent|ImageContent)[], details: T, terminate? }`.
`onUpdate(partialResult)` streams progress (→ `tool_execution_update`). **Error contract: throw,
don't return error content** — the loop wraps throws as `isError: true` results.

### Registration

- **`Agent`**: tools live in `state.tools` (`agent.state.tools = [...]`, array copied on assign).
  The loop finds a tool by `name` per call (`prepareToolCall`, `agent-loop.ts:569`).
- **`AgentHarness`**: tools registered in the constructor (`tools?: TTool[]`, deduped by name,
  `agent-harness.ts:207-213`) into a `Map<string, TTool>`. `activeToolNames` selects which are
  exposed to the model this turn (`createTurnState` `:336`). Mutators `setTools(tools,
  activeToolNames?)` / `setActiveTools(names)` (`:906,941`) validate uniqueness + existence,
  persist an `active_tools_change` entry (or queue it as a pending write when busy), and emit
  `tools_update`. Generic over `TTool extends AgentTool` so apps carry richer tool shapes.

### Invocation across `AgentMessage` / LLM `Message` boundary

The harness uses `convertToLlm` from `src/harness/messages.ts:120` (not the `Agent` default).
It maps custom message roles into LLM messages: `bashExecution`→user text (or dropped if
`excludeFromContext`), `custom`→user, `branchSummary`/`compactionSummary`→user text with prefix
wrappers, and passes through `user`/`assistant`/`toolResult`. Custom roles are added to
`CustomAgentMessages` via **declaration merging** (`messages.ts:54-61`), extending the
`AgentMessage` union (`types.ts:300-309`) — the package's main extensibility mechanism for apps.

---

## 8. Key types & data structures (file:line)

- `AgentMessage` — `src/types.ts:309` (`Message | CustomAgentMessages[...]`).
- `AgentEvent` — `src/types.ts:403` (the UI/event union).
- `AgentTool` / `AgentToolResult` — `src/types.ts:361` / `:345`.
- `AgentContext` — `src/types.ts:387` (`{ systemPrompt, messages, tools? }`).
- `AgentState` — `src/types.ts:317`.
- `AgentLoopConfig` — `src/types.ts:135` (`convertToLlm`, `transformContext`, `getApiKey`,
  `shouldStopAfterTurn`, `prepareNextTurn`, `getSteeringMessages`, `getFollowUpMessages`,
  `toolExecution`, `beforeToolCall`, `afterToolCall`).
- `StreamFn` — `src/types.ts:24` (must never throw; encode failures in the stream).
- `BeforeToolCallResult` / `AfterToolCallResult` — `src/types.ts:55` / `:72`.
- `SessionTreeEntry` union — `src/harness/types.ts:409`; base `:334`.
- `SessionContext` — `src/harness/types.ts:422`; `SessionStorage` — `:440`; `SessionRepo` — `:468`.
- `ExecutionEnv` / `FileSystem` / `Shell` — `src/harness/types.ts:332` / `:268` / `:321`.
- `Result<V,E>` + `ok`/`err`/`getOrThrow` — `src/harness/types.ts:5-27`.
- Error classes: `FileError` `:122`, `ExecutionError` `:146`, `CompactionError` `:161`,
  `BranchSummaryError` `:176`, `SessionError` `:196`, `AgentHarnessError` `:219`.
- `AgentHarnessPhase` — `src/harness/types.ts:492` (`idle|turn|compaction|branch_summary|retry`).
- `PendingSessionWrite` — `src/harness/types.ts:494` (entry shape minus `id/parentId/timestamp`).
- `CompactionSettings` / `CompactionPreparation` — `src/harness/types.ts:748` / `:754`.
- `AgentHarnessOptions` — `src/harness/types.ts:798`; `AgentHarnessEventResultMap` — `:704`.
- `Skill` / `PromptTemplate` / `AgentHarnessResources` — `src/harness/types.ts:46` / `:60` / `:70`.
- Custom message interfaces (`BashExecutionMessage`, `CustomMessage`, `BranchSummaryMessage`,
  `CompactionSummaryMessage`) — `src/harness/messages.ts:19-52`.

---

## 9. Gotchas, invariants, concurrency & abort

**Concurrency / single-flight:**
- `Agent` allows exactly one active run: `prompt()`/`continue()` throw if `activeRun` is set
  (`agent.ts:328,339`). Use `steer`/`followUp` to inject while running.
- `AgentHarness` uses an explicit `phase`. Structural ops (`prompt`, `skill`,
  `promptFromTemplate`, `compact`, `navigateTree`) require `phase === "idle"` and **set the phase
  synchronously before the first `await`** (`:631,711,768`); otherwise they reject with code
  `"busy"`. `steer`/`followUp`/`nextTurn`/`abort` and runtime config setters are allowed during a
  turn.

**Abort:**
- `Agent.abort()` (`:300`) aborts the run's `AbortController`. `AgentHarness.abort()` (`:1005`)
  aborts the run, **clears steer + follow-up queues** but **not the `nextTurn` queue** (those
  survive abort and prepend to the next user turn), then `waitForIdle()` and emits `abort`.
- The signal threads everywhere: into `streamFn`, `tool.execute`, `beforeToolCall`/`afterToolCall`
  (hooks are responsible for honoring it), and `ExecutionEnv` operations. The loop checks
  `signal?.aborted` after each preflight/hook checkpoint (`agent-loop.ts:440,478,497,591,606`).
- An aborted assistant message has `stopReason: "aborted"` and exits the loop immediately
  (`agent-loop.ts:196`). Provider failures are encoded as an error assistant message
  (`handleRunFailure` `agent.ts:476`, `createFailureMessage` `agent-harness.ts:49`), never thrown
  out of the stream — per the `StreamFn` contract.

**State / ordering invariants:**
- `transformContext`, `convertToLlm`, `getApiKey`, `shouldStopAfterTurn`, `prepareNextTurn`,
  `getSteeringMessages`, `getFollowUpMessages` **must not throw/reject** (contracts in
  `src/types.ts`); a throw breaks the loop without a normal event sequence.
- Emitted message objects are shallow clones so subscribers don't alias the mutating partial
  (`agent-loop.ts:319,338`).
- **Harness save points**: `prepareNextTurn` (`agent-harness.ts:457`) flushes pending session
  writes, then **rebuilds a fresh turn snapshot** (model/thinking/tools/system-prompt/stream
  options/session-id) — config changes made mid-run take effect on the *next* turn, never on the
  in-flight provider request. `message_end` persists the message **before** notifying subscribers
  (`:511-515`), preserving transcript order. `turn_end` flushes pending writes and emits a
  `save_point` (`:516-527`).
- **Pending session writes** (`pendingSessionWrites`, flushed in `flushPendingSessionWrites`
  `:484`): config setters / `appendMessage` while busy queue durable writes that flush at save
  points, at `agent_end`, and in failure cleanup — they are **never dropped**, flushed one-by-one,
  and rolled back into the queue if a queue-update notification fails (`drainQueuedMessages`
  `:409`).
- **Sequential poison pill**: any one tool with `executionMode:"sequential"` forces the whole
  assistant batch to run serially (`agent-loop.ts:381-388`).
- **`terminate` is all-or-nothing**: the loop only stops early when **every** finalized result in
  the batch sets `terminate:true` (`shouldTerminateToolBatch`, `:544`). Mixed batches continue.
  It's a runtime hint only; the persisted `toolResult` messages are ordinary LLM tool results.
- **`continue()` last-message rule**: the transcript must end in `user`/`toolResult`, never
  `assistant` (`agent-loop.ts:74,131`). `Agent.continue()` has special handling: if the last
  message is `assistant`, it drains steering then follow-up queues instead (`agent.ts:348-361`).
- **Compaction cut points never split a tool call from its results** (`findValidCutPoints`
  excludes `toolResult`, `compaction.ts:277`); a split *turn* gets a separate prefix summary.
- **Session leaf must be durable**: `setLeafId` appends a `leaf` entry (`jsonl-storage.ts:226`),
  not just an in-memory cursor, so tree navigation survives storage reopen.
- **Auto-compaction, retry phase, durable recovery, and the redesigned hook/observability
  systems are designed but NOT implemented** (`docs/agent-harness.md` implementation todo,
  `docs/durable-harness.md`, `docs/observability.md`).

---

## Quick file map

| Area | Files |
|---|---|
| Public API | `src/index.ts`, `src/node.ts` |
| Core loop | `src/agent-loop.ts` |
| Stateful wrapper | `src/agent.ts` |
| Core types | `src/types.ts` |
| Proxy stream fn | `src/proxy.ts` |
| Harness orchestrator | `src/harness/agent-harness.ts` |
| Harness types/events/errors | `src/harness/types.ts` |
| Message conversion + custom roles | `src/harness/messages.ts` |
| Sessions | `src/harness/session/{session,jsonl-storage,jsonl-repo,memory-storage,memory-repo,repo-utils,uuid}.ts` |
| Compaction | `src/harness/compaction/{compaction,branch-summarization,utils}.ts` |
| Execution env | `src/harness/env/nodejs.ts` |
| Skills / templates / sys-prompt | `src/harness/{skills,prompt-templates,system-prompt}.ts` |
| Utils | `src/harness/utils/{shell-output,truncate}.ts` |
| Docs | `docs/{agent-harness,durable-harness,hooks,observability}.md` |
| Tests | `test/`, harness tests `test/harness/` (run via `npm run test:harness`) |
