# Study Notes 04 — `packages/coding-agent` core (`src/core` + top-level `src/` + `src/bun`)

Scope: `packages/coding-agent/src/core/**` plus the top-level entry files in `src/*.ts` and
`src/bun/*`. The interactive TUI (`modes/interactive`), `modes/rpc`, `utils/`, and `examples/` are
out of scope (other agents cover them). All file paths below are absolute.

This package is **`pi`**, a self-extensible local coding agent. `core/` is the engine: it wraps the
lower-level `@earendil-works/pi-agent-core` `Agent` with sessions, tools, extensions, compaction,
skills, HTML export, trust, and the SDK. Modes (interactive/print/rpc) layer I/O on top of
`AgentSession`.

---

## 1. Entry points, SDK surface, and the bun bits

### Process entry chain
- `/home/user/pi/packages/coding-agent/src/bun/cli.ts` — bun-compiled-binary entry. Sets
  `process.title`, silences `process.emitWarning`, then (in order) `restoreSandboxEnv()`,
  `import("./register-bedrock.ts")`, `import("../cli.ts")`.
- `/home/user/pi/packages/coding-agent/src/bun/restore-sandbox-env.ts` — workaround for
  oven-sh/bun#27802: bun binaries inside sandboxes (nono) get an empty `process.env`; this rehydrates
  it from `/proc/self/environ` on Linux (`restore-sandbox-env.ts:15`). No-op when not bun or env
  already populated.
- `/home/user/pi/packages/coding-agent/src/bun/register-bedrock.ts` — 4 lines; calls
  `setBedrockProviderModule(bedrockProviderModule)` so the AWS Bedrock provider is bundled into the
  binary (tree-shaking would otherwise drop it).
- `/home/user/pi/packages/coding-agent/src/cli.ts` — the node entry. Sets `PI_CODING_AGENT=true`,
  configures the undici HTTP dispatcher, then `main(process.argv.slice(2))`.
- `/home/user/pi/packages/coding-agent/src/main.ts` — the real driver (1046 lines). Parses args
  (`parseArgs`), resolves app mode (`resolveAppMode`, `main.ts:102`: rpc / json / print / interactive
  — non-TTY forces print), runs migrations, builds the `SessionManager` (`createSessionManager`,
  `main.ts:253` handles `--fork`/`--session`/`--resume`/`--continue`/`--session-id`/`--no-session`),
  resolves project trust, builds a **runtime factory** (`createRuntime`, `main.ts:810`) and calls
  `createAgentSessionRuntime`, then dispatches to `runRpcMode` / `InteractiveMode` / `runPrintMode`.
  Also handles standalone subcommands: `--export` (HTML via `exportFromFile`), `--version`,
  package/config CLI (`handlePackageCommand`/`handleConfigCommand`). `MainOptions.extensionFactories`
  lets an embedder inject in-process extensions.
- Other top-level files (briefly): `src/config.ts` (paths, `VERSION`, `APP_NAME`, `getAgentDir`,
  `ENV_SESSION_DIR`, `isBunBinary`), `src/index.ts` (the **public package export surface** — the SDK),
  `src/migrations.ts`, `src/package-manager-cli.ts`.

### Public SDK (`/home/user/pi/packages/coding-agent/src/index.ts`)
Re-exports everything an embedder/extension needs: `createAgentSession`, the runtime factory
(`createAgentSessionRuntime`, `AgentSessionRuntime`, `createAgentSessionServices`,
`createAgentSessionFromServices`), `AgentSession`, the tool factories, compaction functions,
`SessionManager`, `SettingsManager`, `ModelRegistry`, `AuthStorage`, the entire extension type surface,
skills, and `main`. Note extensions can `import "@earendil-works/pi-coding-agent"` — the loader maps
that specifier back to this index (see §2).

### SDK core (`/home/user/pi/packages/coding-agent/src/core/sdk.ts`)
- `createAgentSession(options)` (`sdk.ts:166`) is the factory. It assembles `AuthStorage`,
  `ModelRegistry`, `SettingsManager`, `SessionManager`, `DefaultResourceLoader`, then:
  - Restores model/thinking from an existing session if present (`sdk.ts:195`), else
    `findInitialModel`. Clamps thinking level to model caps.
  - Computes active tool names: default `["read","bash","edit","write"]`; `noTools:"all"` → none;
    `tools` allowlist and `excludeTools` denylist applied (`sdk.ts:244`).
  - Constructs the agent-core `Agent` with a `streamFn` that pulls API keys/headers from
    `ModelRegistry` and calls `streamSimple` (`sdk.ts:301`), plus `onPayload`/`onResponse`/
    `transformContext` hooks that delegate to the `ExtensionRunner` via a mutable
    `extensionRunnerRef`. `convertToLlmWithBlockImages` strips images when `blockImages` is set
    (defense-in-depth, `sdk.ts:255`).
  - Restores prior messages into `agent.state.messages` or appends initial model/thinking entries.
  - Returns `{ session: AgentSession, extensionsResult, modelFallbackMessage }`.
- Tool factories re-exported for custom-cwd use: `createCodingTools`, `createReadOnlyTools`, plus
  per-tool `createReadTool`/`createBashTool`/… and `withFileMutationQueue`.

### Runtime / services split (important architecture point)
- `/home/user/pi/packages/coding-agent/src/core/agent-session-services.ts` — `AgentSessionServices`
  is the **cwd-bound infrastructure bundle** (`agentDir`, `authStorage`, `settingsManager`,
  `modelRegistry`, `resourceLoader`). `createAgentSessionServices` builds it (loads resources, flushes
  pending provider registrations, applies extension flag values). `createAgentSessionFromServices`
  then builds the `AgentSession`. Services are recreated whenever the effective cwd changes (because
  `--session`/`--resume` can point at a different project).
- `/home/user/pi/packages/coding-agent/src/core/agent-session-runtime.ts` — `AgentSessionRuntime`
  owns the current `AgentSession` + services and implements session **replacement**: `switchSession`,
  `newSession`, `fork`, `importFromJsonl`, `dispose`. Each teardown emits `session_shutdown`, disposes
  the old session, calls the stored `CreateAgentSessionRuntimeFactory` to build the replacement, then
  `finishSessionReplacement` (rebind + optional `withSession` callback). This is why extensions must
  not hold a stale `ctx` after `newSession`/`fork`/`switchSession`/`reload` (see invalidate messages).

### `AgentSession` (`/home/user/pi/packages/coding-agent/src/core/agent-session.ts`, 3129 lines)
The central object shared by all modes. Highlights:
- Subscribes to agent-core events (`_handleAgentEvent`, `agent-session.ts:476`) and **persists**
  messages to the `SessionManager` on `message_end` (`appendMessage` / `appendCustomMessageEntry`,
  `agent-session.ts:506`).
- Installs agent-core tool hooks once (`_installAgentToolHooks`, `agent-session.ts:403`):
  `beforeToolCall` → extension `tool_call` (can block), `afterToolCall` → extension `tool_result`
  (can rewrite content/details/isError).
- `prompt()` (`agent-session.ts:986`): handles `/extension-command` immediately, emits `input` event,
  expands `/skill:name` and prompt templates, queues via steer/followUp when streaming, validates
  model+auth, runs a pre-prompt compaction check, emits `before_agent_start`, then runs the agent
  loop (`_runAgentPrompt` → `_handlePostAgentRun` drives retry/compaction/continuation).
- Tool registry: `_refreshToolRegistry` (`agent-session.ts:2282`) merges built-in tool definitions,
  extension-registered tools, and SDK `customTools`, applies allow/deny lists, wraps them, and
  rebuilds the system prompt from snippets/guidelines. `_buildRuntime` (`agent-session.ts:2375`)
  constructs the `ExtensionRunner` and wires `bindCore`.
- Model/thinking management, auto-retry with exponential backoff (`_prepareRetry`,
  `agent-session.ts:2485`; retryable-error regex at `agent-session.ts:2466`), bash execution
  (`executeBash`, `agent-session.ts:2573`), tree navigation/branch summary (`navigateTree`,
  `agent-session.ts:2695`), stats, `getContextUsage`, `exportToHtml`/`exportToJsonl`.

Key exported types: `AgentSessionConfig` (`agent-session.ts:157`), `AgentSessionEvent`
(`:124`, extends agent-core events with compaction/retry/queue events), `PromptOptions` (`:199`),
`SessionStats` (`:221`), `parseSkillBlock` (`:112`).

---

## 2. The extension system (`core/extensions/*`) — how `pi` self-extends

Extensions are **TypeScript/JS modules** loaded at runtime that can subscribe to lifecycle events,
register LLM-callable tools, register slash commands / shortcuts / CLI flags / message renderers /
model providers, and drive the UI. They run **in-process with full user privileges** (no sandbox).

### Files
- `types.ts` (1603 lines) — the entire type surface. Key: `ToolDefinition` (`types.ts:433`),
  `ExtensionAPI` (`:1118`), `Extension` (`:1574`), `ExtensionRuntime` (`:1571`), `ExtensionContext`
  (`:300`), all events (tool/agent/session/input/message/provider/resources/project_trust),
  `defineTool` helper (`:491`), type guards (`isReadToolResult`, etc.).
- `loader.ts` (605 lines) — discovery + loading via **jiti**.
- `runner.ts` (1129 lines) — `ExtensionRunner`: holds extensions, dispatches events, owns the
  shared runtime/action bindings and UI context.
- `wrapper.ts` — wraps a `RegisteredTool`'s `ToolDefinition` into an agent-core `AgentTool`, injecting
  `runner.createContext()` as the tool's `ctx`.
- `index.ts` — barrel exports.

### Discovery (`loader.ts`)
`discoverAndLoadExtensions` (`loader.ts:557`) collects paths from, in order:
1. Project-local: `cwd/.pi/extensions/` (`CONFIG_DIR_NAME`).
2. Global: `~/.pi/agent/extensions/`.
3. Explicitly configured paths (CLI `-e`, settings).

`discoverExtensionsInDir` (`loader.ts:520`) rules (one level deep, no deep recursion): direct
`*.ts`/`*.js` files load directly; a subdir with `index.ts`/`index.js` loads that; a subdir with
`package.json` containing a `pi.extensions` manifest array loads what it declares
(`resolveExtensionEntries`, `loader.ts:478`). Paths are de-duplicated by resolved path.

### Loading & the jiti sandbox-ish boundary
`loadExtensionModule` (`loader.ts:331`) uses `createJiti`. Two modes:
- **Bun binary** (`isBunBinary`): `virtualModules: VIRTUAL_MODULES` + `tryNative:false` so jiti
  resolves *all* imports itself. `VIRTUAL_MODULES` (`loader.ts:44`) maps `typebox`,
  `@earendil-works/pi-agent-core`, `pi-ai`, `pi-ai/oauth`, `pi-tui`, **`pi-coding-agent`** (and the
  legacy `@mariozechner/*` aliases) to the statically-bundled instances. This is why extensions can
  `import { defineTool } from "@earendil-works/pi-coding-agent"` inside a single compiled binary.
- **Node/dev**: `alias: getAliases()` (`loader.ts:71`) resolves the same specifiers to
  workspace `dist/` paths or node_modules.

A factory must be the **default export** and a function `(api: ExtensionAPI) => void | Promise<void>`
(`loader.ts:340`). `loadExtensionFromFactory` (`loader.ts:396`) supports inline factories
(`MainOptions.extensionFactories`).

### The Extension object & API
`createExtension` (`loader.ts:348`) makes an `Extension` with empty `Map`s for `handlers`, `tools`,
`messageRenderers`, `commands`, `flags`, `shortcuts` (`types.ts:1574`). `createExtensionAPI`
(`loader.ts:177`) returns the `ExtensionAPI`:
- **Registration** (writes to the Extension): `on(event, handler)`, `registerTool(def)` (then
  `runtime.refreshTools()`), `registerCommand`, `registerShortcut`, `registerFlag`,
  `registerMessageRenderer`, `registerProvider`/`unregisterProvider`, `getFlag`.
- **Actions** (delegate to the shared `ExtensionRuntime`): `sendMessage`, `sendUserMessage`,
  `appendEntry`, `setSessionName`/`getSessionName`, `setLabel`, `exec` (spawn a subprocess via
  `execCommand`), `getActiveTools`/`setActiveTools`/`getAllTools`, `getCommands`, `setModel`,
  `get/setThinkingLevel`, `events` (an `EventBus`).

### Runtime & lifecycle (`createExtensionRuntime`, `loader.ts:124`)
The runtime starts with **throwing stubs** for actions ("Extension runtime not initialized") so
extensions cannot call actions during load. `runner.bindCore()` (`runner.ts:306`) injects the real
implementations from the `AgentSession`. Provider registrations issued during load are queued
(`pendingProviderRegistrations`) and flushed by `bindCore`; afterward they take effect immediately.
`runtime.invalidate(message)` marks the runtime stale; `assertActive()` then throws everywhere
(prevents use of a captured `ctx` after session replacement/reload).

### ExtensionRunner (`runner.ts`)
- Constructed in `AgentSession._buildRuntime` with the extensions + runtime + cwd + sessionManager +
  modelRegistry. `createContext()` (`runner.ts:615`) returns a lazy, getter-based `ExtensionContext`
  (lazy so staleness checks fire); `createCommandContext()` (`runner.ts:682`) adds
  `newSession`/`fork`/`navigateTree`/`switchSession`/`reload`/`waitForIdle`.
- Event dispatch: generic `emit()` plus specialized emitters for events that transform/short-circuit:
  `emitToolCall` (first `block:true` wins), `emitToolResult` (accumulates content/details/isError
  changes), `emitMessageEnd` (chained message rewrite, role must be preserved), `emitContext`
  (chained on a `structuredClone`), `emitBeforeProviderRequest`, `emitBeforeAgentStart` (collects
  injected messages + system-prompt edits), `emitInput` (transform chain, `handled` short-circuits),
  `emitResourcesDiscover`, `emitUserBash`. Handler exceptions are caught and reported via
  `emitError`/`onError` rather than crashing the session.
- `getShortcuts` (`runner.ts:459`) resolves keybinding conflicts: a hard-coded
  `RESERVED_KEYBINDINGS_FOR_EXTENSION_CONFLICTS` list (`runner.ts:67`, e.g. `app.interrupt`,
  `app.exit`) blocks extension overrides; non-reserved collisions warn and last-wins.
- `emitProjectTrustEvent` (`runner.ts:197`) is a **static** function (used before a full session
  exists): the first user/global/CLI extension returning `yes`/`no` owns the trust decision.

**Self-extension flow:** an extension's `registerTool` adds a `ToolDefinition` to the Extension map;
`_refreshToolRegistry` wraps it (`wrapRegisteredTools`) into an `AgentTool` that the model can call,
and rebuilds the system prompt. Extensions can also register new model providers, slash commands,
and even spawn subprocesses — so `pi` can grow tools/commands/models entirely from user code.

---

## 3. Built-in tools (`core/tools/*`)

All tools are `ToolDefinition`s (typebox schema + `execute` + optional TUI `renderCall`/`renderResult`)
and are wrapped to `AgentTool` via `tool-definition-wrapper.ts`. Each has pluggable `*Operations`
(default = local FS / shell) so an extension can redirect to SSH/remote. Factory naming:
`create<Tool>ToolDefinition(cwd, options)` and `create<Tool>Tool(cwd, options)`. `index.ts` groups
them: `createCodingTools` = read/bash/edit/write; `createReadOnlyTools` = read/grep/find/ls;
`createAllToolDefinitions` = all 7. `ToolName` union: `read|bash|edit|write|grep|find|ls`
(`tools/index.ts:83`).

| File | Tool | Schema (typebox) | Side effects | Safety notes |
|------|------|------------------|--------------|--------------|
| `read.ts` | `read` | `path`, optional `offset`/`limit` | none (FS read) | Truncates to `DEFAULT_MAX_LINES`/`DEFAULT_MAX_BYTES` (head); images auto-resized to ≤2000px and dropped if model lacks vision; first-line-too-big points model at a `sed` fallback. macOS filename variants tried (`path-utils.ts`). |
| `bash.ts` | `bash` | `command`, optional `timeout` (s) | **runs arbitrary shell** | Spawns the user shell detached, streams stdout+stderr through `OutputAccumulator`; truncates output (last N lines/KB) and persists full output to a temp file; kills the whole **process tree** on abort/timeout (`killProcessTree`); honors `commandPrefix`/`shellPath`. No allowlist — full user privileges. |
| `edit.ts` | `edit` | `path`, `edits[]` of `{oldText,newText}` (`additionalProperties:false`) | overwrites file | Exact-match replacement on the **original** content (edits must be unique, non-overlapping). BOM-stripped, LF-normalized then line-endings restored. Serialized via `withFileMutationQueue`. `prepareArguments` shims legacy `oldText`/`newText` and JSON-string `edits` from some models. Produces diff + unified patch. |
| `write.ts` | `write` | `path`, `content` | **creates/overwrites file**, `mkdir -p` parents | Serialized via `withFileMutationQueue`; no diff/confirm — blind overwrite. |
| `grep.ts` | `grep` | `pattern`,`path?`,`glob?`,`ignoreCase?`,`literal?`,`context?`,`limit?`(100) | none | Shells out to **ripgrep** (`ensureTool`); truncates matches and long lines (`GREP_MAX_LINE_LENGTH`). |
| `find.ts` | `find` | `pattern`(glob),`path?`,`limit?`(1000) | none | Shells out to **fd**; ignores `.gitignore`/`.ignore`/`.fdignore`. |
| `ls.ts` | `ls` | `path?`,`limit?`(500) | none (readdir) | Truncates entries. |

Support files:
- `truncate.ts` — `truncateHead`/`truncateTail`/`truncateLine`, `DEFAULT_MAX_LINES`,
  `DEFAULT_MAX_BYTES`, `formatSize`, `TruncationResult`.
- `output-accumulator.ts` — streaming output buffer + temp-file spillover for bash.
- `file-mutation-queue.ts` — `withFileMutationQueue(path, fn)` serializes mutations targeting the
  **same real path** (resolves symlinks via `realpath`), running different files in parallel
  (`file-mutation-queue.ts:32`). Prevents lost-update races between concurrent edit/write tool calls.
- `path-utils.ts` — `resolveToCwd`, `resolveReadPath[Async]` with macOS filename normalization (NFD,
  narrow-no-break-space before AM/PM, curly quotes).
- `tool-definition-wrapper.ts` — `wrapToolDefinition` (def → AgentTool), and
  `createToolDefinitionFromAgentTool` (synthesize a minimal def from a plain AgentTool override).
- `render-utils.ts`, `edit-diff.ts` (diff/patch generation), `index.ts` (factories + groupings).

---

## 4. Compaction (`core/compaction/*`)

Compaction shrinks long conversations by summarizing old turns while keeping recent ones. The pure
logic lives here; `AgentSession` orchestrates I/O + events.

- `compaction.ts` (876 lines):
  - `CompactionSettings`/`DEFAULT_COMPACTION_SETTINGS` (`:115`/`:121`): `enabled`,
    `reserveTokens:16384`, `keepRecentTokens:20000`.
  - Token math: `calculateContextTokens(usage)`, `estimateTokens(msg)` (chars/4 heuristic;
    images ≈ 4800 chars), `estimateContextTokens(messages)` (trusts last real assistant `usage`,
    estimates trailing messages), `shouldCompact(tokens, window, settings)` = tokens > window −
    reserve.
  - `findCutPoint` (`:386`): walks backward accumulating estimated tokens until `keepRecentTokens`,
    cutting only at valid points (user/assistant/custom/bash/branchSummary/compactionSummary — never
    a tool result). Handles **split turns** (cutting mid-turn → also summarize the turn prefix).
  - `prepareCompaction(pathEntries, settings)` (`:644`) → `CompactionPreparation`
    (firstKeptEntryId, messagesToSummarize, turnPrefixMessages, isSplitTurn, tokensBefore,
    previousSummary, fileOps). Iterative: reads the previous compaction's summary + tracked files.
  - `compact(preparation, model, apiKey, …, streamFn)` (`:747`) generates summaries with the LLM
    (`generateSummary`, structured `## Goal / Progress / Next Steps …` format; `UPDATE_…` prompt when
    a previous summary exists), merges split-turn prefix summary, appends
    `<read-files>`/`<modified-files>` lists, returns `CompactionResult`.
- `utils.ts` — `FileOperations` tracking (`extractFileOpsFromMessage` reads read/write/edit tool
  calls), `serializeConversation` (flattens messages to `[User]/[Assistant]/[Tool result]` text so the
  model summarizes rather than continues; tool results truncated to 2000 chars), and
  `SUMMARIZATION_SYSTEM_PROMPT`.
- `branch-summarization.ts` — `generateBranchSummary`/`collectEntriesForBranchSummary` for the tree
  navigation path (summarizing an abandoned branch when jumping nodes).

### Relationship to `packages/agent` compaction
This is the **coding-agent layer's own** compaction, built on session-entry semantics
(`SessionEntry`, cut points, branch-aware `getBranch()`, persisted `CompactionEntry`). It does not
delegate to `@earendil-works/pi-agent-core`; it only uses agent-core types (`AgentMessage`,
`StreamFn`) and the AI SDK's `completeSimple`/`streamFn`. `AgentSession` invokes it in three places
(`agent-session.ts`): manual `compact()` (`:1636`), and `_checkCompaction`/`_runAutoCompaction`
(`:1793`/`:1876`) for **threshold** (no auto-retry) vs **overflow** (remove the overflow error
message, compact, then auto-retry, guarded by `_overflowRecoveryAttempted`). Extensions can override
or cancel via `session_before_compact` and observe via `session_compact`.

---

## 5. Sessions & session format (`core/session-manager.ts`, 1567 lines)

Sessions are **append-only JSONL tree logs**. `CURRENT_SESSION_VERSION = 3` (`session-manager.ts:30`).

- File = a `SessionHeader` line (`:32`: `type:"session"`, `version`, `id`, `timestamp`, `cwd`,
  optional `parentSession`) followed by `SessionEntry` lines, each with `id`/`parentId`/`timestamp`
  (`SessionEntryBase`, `:46`). The `parentId` links form a **tree**, enabling forks/branches inside
  one file; `getBranch()` (`:1150`) walks from the leaf to the root to get the active linear path.
- Entry types (`SessionEntry` union, `:140`): `message` (wraps an `AgentMessage`),
  `thinking_level_change`, `model_change`, `compaction` (`CompactionEntry`, `:69`, with `summary`,
  `firstKeptEntryId`, `tokensBefore`, `details`, `fromHook`), `branch_summary`, `custom` (extension
  state, **not** in LLM context), `custom_message` (extension content **injected** into LLM context),
  `label`, `session_info` (display name).
- `buildSessionContext()` (`:1165`) → `{ messages, thinkingLevel, model }` reconstructs the agent
  state from the current branch (this is what `createAgentSession` loads into `agent.state.messages`).
- Persistence is via `append*` methods (`appendMessage` `:950`, `appendCompaction`,
  `appendCustomMessageEntry`, `appendThinkingLevelChange`, `appendModelChange`, `appendLabelChange`,
  `appendSessionInfo`, `appendCustomEntry`), each `appendFileSync`-ing one JSONL line and updating the
  in-memory leaf.
- Branching/forking: `branch(id)`, `resetLeaf()`, `branchWithSummary()`, `createBranchedSession()`
  (writes a new file re-chained from a leaf), `newSession()`.
- Lifecycle statics: `create`, `open`, `continueRecent`, `inMemory` (no persistence — used for
  `--no-session`/help), `forkFrom`, `list`/`listAll`. Session id validation: `assertValidSessionId`
  (`:207`). Default dir from `getSessionsDir`/`ENV_SESSION_DIR`/settings.
- Migrations: `migrateV1ToV2` (`:226`, adds id/parentId tree + converts
  `firstKeptEntryIndex`→`firstKeptEntryId`) and `migrateV2ToV3` (`:255`, renames `hookMessage`→
  `custom`). `migrateSessionEntries`/`parseSessionEntries` exported.

Docs cross-ref: `docs/session-format.md`, `docs/sessions.md`.

---

## 6. HTML export (`core/export-html/*`)

`exportSessionToHtml(sm, state?, options)` (`export-html/index.ts:236`) and the standalone
`exportFromFile(inputPath, …)` (`:288`, used by `pi --export`) produce a single self-contained HTML
file.

Pipeline:
1. `generateHtml` (`:143`) reads four template assets from `getExportTemplateDir()`:
   `template.html`, `template.css`, `template.js`, and **vendored** `vendor/marked.min.js` +
   `vendor/highlight.min.js` (markdown + syntax highlighting shipped inline so the export works
   offline). Theme colors are injected as CSS custom properties (`generateThemeVars`,
   `deriveExportColors` computes light/dark page/card/info backgrounds from `userMessageBg`).
2. Session data (`header`, `entries`, `leafId`, optional `systemPrompt`/`tools`/`renderedTools`) is
   JSON-stringified, **base64-encoded**, and substituted into `{{SESSION_DATA}}`; the browser-side
   `template.js` (1864 lines) renders the conversation client-side.
3. Built-in tools `bash/read/write/edit/ls` are `TEMPLATE_RENDERED_TOOLS` (`:178`) — rendered by the
   template JS directly. Custom/extension tools are **pre-rendered** server-side:
   `tool-renderer.ts` (`createToolHtmlRenderer`) invokes each tool's TUI `renderCall`/`renderResult`
   to produce ANSI `Component` output, and `ansi-to-html.ts` (`ansiLinesToHtml`) converts ANSI escape
   sequences to styled HTML spans (collapsed + expanded variants). On render error it falls back to
   the structured result.
4. Output written to `pi-session-<basename>.html` (or `outputPath`).

In-memory sessions cannot be exported (`:246`).

---

## 7. Skills (`core/skills.ts`)

Skills are markdown instruction files (Agent Skills spec) loaded into the system prompt.

- `Skill` (`skills.ts:74`): `name`, `description`, `filePath`, `baseDir`, `sourceInfo`,
  `disableModelInvocation`. Frontmatter parsed via `parseFrontmatter`; name defaults to parent dir
  name. Validation (`validateName`/`validateDescription`): name ≤64 chars, `[a-z0-9-]`, no
  leading/trailing/double hyphens; description required, ≤1024 chars. Missing description → skipped.
- Discovery (`loadSkillsFromDir`, `:168`): a dir with `SKILL.md` is a skill root (no deeper recursion);
  otherwise direct `*.md` children load and subdirs are recursed for `SKILL.md`. Respects
  `.gitignore`/`.ignore`/`.fdignore`, skips `node_modules`, follows symlinks, de-dupes by real path.
- `loadSkills` (`:387`) merges `~/.pi/agent/skills` (user), `cwd/.pi/skills` (project), and explicit
  paths; name collisions emit diagnostics (first wins).
- `formatSkillsForPrompt` (`:335`) emits the `<available_skills>` XML block (excluding
  `disable-model-invocation` skills, which are still callable via `/skill:name`). Telling the model to
  `read` the SKILL.md when relevant.
- `AgentSession` integrates skills: `_expandSkillCommand` (`agent-session.ts:1173`) turns
  `/skill:name args` into a `<skill name=… location=…>…</skill>` block (parsed back by
  `parseSkillBlock`, `agent-session.ts:112`). System prompt assembly uses
  `resourceLoader.getSkills()`.

---

## 8. Key types, gotchas, and the security model

### Cited key types
- `AgentSessionConfig` `agent-session.ts:157`; `AgentSessionEvent` `agent-session.ts:124`;
  `PromptOptions` `agent-session.ts:199`; `SessionStats` `agent-session.ts:221`.
- `CreateAgentSessionOptions` `sdk.ts:34`; `CreateAgentSessionResult` `sdk.ts:86`.
- `AgentSessionServices` `agent-session-services.ts:74`; `AgentSessionRuntimeDiagnostic` `:26`.
- `ToolDefinition` `extensions/types.ts:433`; `ExtensionAPI` `:1118`; `Extension` `:1574`;
  `ExtensionRuntime` `:1571`; `ExtensionContext` `:300`; `LoadExtensionsResult` `:1587`;
  `ProjectTrustContext`/`ProjectTrustEventResult` `:513`/`:508`.
- Tool I/O: `BashToolInput`/`BashToolDetails` `tools/bash.ts:29`/`:31`; `ReadToolInput`
  `tools/read.ts:26`; `EditToolInput`/`EditToolDetails` `tools/edit.ts:55`/`:61`; `WriteToolInput`
  `tools/write.ts:19`. `BashOperations`/`createLocalBashOperations` `tools/bash.ts:40`/`:66`.
- Session: `SessionHeader` `session-manager.ts:32`; `SessionEntry` union `:140`;
  `CompactionEntry` `:69`; `CustomEntry` vs `CustomMessageEntry` `:100`/`:131`; `SessionContext`
  `:164`; `CURRENT_SESSION_VERSION` `:30`.
- Compaction: `CompactionSettings`/`DEFAULT_COMPACTION_SETTINGS` `compaction.ts:115`/`:121`;
  `CompactionPreparation` `:626`; `CompactionResult` `:103`. `FileOperations` `utils.ts:12`.
- Trust: `ProjectTrustStore`/`ProjectTrustDecision`/`hasProjectTrustInputs` `trust-manager.ts:125`/
  `:7`/`:101`.

### Gotchas
- **Stale contexts**: never reuse a captured `pi`/command `ctx` after `newSession`/`fork`/
  `switchSession`/`reload`. `runtime.invalidate()` + `assertActive()` enforce this with an explicit
  error message (`loader.ts:157`, `runner.ts:508`). Move post-replacement work into `withSession`.
- **Runtime action stubs throw during load** (`loader.ts:124`): only registration methods are valid
  while an extension's factory runs; actions are bound later by `bindCore`.
- **Message persistence vs. agent state**: `_replaceMessageInPlace` (`agent-session.ts:585`) mutates
  the finalized message object so `message_end` extension rewrites stay consistent with what
  `SessionManager.appendMessage` persists.
- **Context-usage after compaction** is `null` until a fresh post-compaction assistant `usage`
  exists (kept pre-compaction usage is stale) — see `getContextUsage` (`agent-session.ts:2961`) and
  `_checkCompaction`'s compaction-boundary checks (`:1812`).
- **Overflow recovery is one-shot** per turn (`_overflowRecoveryAttempted`).
- **File mutation queue keys on `realpath`** — symlinked paths to the same inode serialize together.
- **`blockImages`** filtering happens in `convertToLlm` wrapper (`sdk.ts:255`) and is re-checked
  dynamically so a mid-session setting change takes effect.
- **Extension keybinding reservations** are a hard-coded list (`runner.ts:67`); extensions silently
  lose conflicts with reserved bindings.
- **bash kills the whole process tree** on abort/timeout (detached spawn + `killProcessTree`); a
  child that re-parents can still leak.

### Security model (see `docs/security.md`)
- **No built-in sandbox.** Built-in tools (`bash`/`write`/`edit`/`read`) and **extensions** run with
  full user-process privileges. Extensions are arbitrary TS that can spawn subprocesses (`exec`),
  register providers, and rewrite tool results. Real isolation must come from OS/VM/container.
- **Project trust** (`trust-manager.ts`) is only an **input-loading guard**, not a runtime sandbox.
  `hasProjectTrustInputs(cwd)` (`:101`) is true when `.pi/`, an `AGENTS.md`/`CLAUDE.md` ancestor, or
  `.agents/skills` exists. Decisions stored per canonical cwd in `~/.pi/agent/trust.json`
  (`ProjectTrustStore`, lockfile-guarded). Untrusted projects skip project-local settings/extensions/
  skills/prompts/themes/context files until approved; user/global + CLI `-e` extensions still load and
  can answer `project_trust` (`runner.ts:197`). Non-interactive modes don't prompt — they ignore
  project inputs unless `--approve`. `main.ts:624` (`resolveProjectTrusted`) is the resolution logic.
- Prompt injection from repo files/build output is accepted local-agent risk.

---

### Quick mental model
`bun/cli → cli.ts → main.ts` builds a **runtime factory** → `AgentSessionRuntime` owns a swappable
`AgentSession`. `AgentSession` wraps an agent-core `Agent`, persists to a JSONL **tree** session,
runs **built-in tools**, hosts the **ExtensionRunner** (jiti-loaded TS that can add tools/commands/
providers — the self-extension mechanism), auto-**compacts** long histories, and can **export** to
self-contained HTML. **Project trust** gates which project-local inputs load; there is otherwise **no
sandbox**.
