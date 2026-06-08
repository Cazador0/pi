# Pi Monorepo — Master Study Index

A complete, teaching-quality knowledge base for the `pi` agent-harness monorepo
(`pi-monorepo`, repo `earendil-works/pi-mono`). This index stitches together
seven deep-dive workstreams into one mental model. Read this file first, then
dive into the per-workstream notes.

> How this was built: the codebase (~100k LOC of source across 4 packages plus
> tooling) exceeds a single reading pass, so it was partitioned into seven
> non-overlapping workstreams, each studied in depth and written up separately.
> Every claim in the notes is cited as `file:line`; key claims were spot-checked
> against source.

---

## 1. What pi is

Pi is a **self-extensible local coding agent** plus the reusable runtime layers
beneath it. It ships two CLI binaries:

- **`pi`** — the interactive coding agent (`@earendil-works/pi-coding-agent`).
- **`pi-ai`** — a thin CLI over the multi-provider LLM API (`@earendil-works/pi-ai`).

It is an npm workspaces monorepo with **lockstep versioning** (all packages share
one version; currently `0.79.0`). License MIT.

---

## 2. The four packages and how they stack

```
┌───────────────────────────────────────────────────────────────┐
│ @earendil-works/pi-coding-agent   (packages/coding-agent)      │
│   the `pi` CLI: sessions, tools, extensions, compaction,       │
│   skills, 3 run modes (interactive TUI / RPC / print)          │
└───────────────┬───────────────────────┬───────────────────────┘
                │ depends on            │ depends on
                ▼                       ▼
┌───────────────────────────┐   ┌───────────────────────────────┐
│ @earendil-works/pi-agent  │   │ @earendil-works/pi-tui         │
│   (packages/agent)        │   │   (packages/tui)               │
│   agent loop, durable     │   │   differential terminal        │
│   harness, sessions,      │   │   renderer + components        │
│   compaction, hooks       │   │   (independent leaf)           │
└───────────────┬───────────┘   └───────────────────────────────┘
                │ depends on
                ▼
┌───────────────────────────────────────────────────────────────┐
│ @earendil-works/pi-ai     (packages/ai)                        │
│   unified multi-provider LLM API: ~9 wire APIs, ~40 providers, │
│   one message model, one streaming protocol, model catalog,    │
│   OAuth, image generation   (independent leaf)                 │
└───────────────────────────────────────────────────────────────┘
```

Dependency facts (from each `package.json`):

| Package | npm name | Workspace deps | Notable external deps | Binary |
|---------|----------|----------------|-----------------------|--------|
| ai | `@earendil-works/pi-ai` | none | anthropic-sdk, openai, @google/genai, aws bedrock, mistral, typebox, partial-json | `pi-ai` |
| tui | `@earendil-works/pi-tui` | none | get-east-asian-width, marked | — |
| agent | `@earendil-works/pi-agent-core` | ai | ignore, yaml, typebox | — |
| coding-agent | `@earendil-works/pi-coding-agent` | ai, agent, tui | chalk, diff, glob, jiti, undici, photon-node, proper-lockfile, highlight.js | `pi` |

`typebox` is the shared schema/validation backbone across ai, agent, and
coding-agent (tool schemas, theme/config validation). `jiti` is how the
coding-agent loads TypeScript extensions at runtime.

---

## 3. The notes files (your reading map)

| # | File | Scope | Lines | Start here if you want… |
|---|------|-------|-------|--------------------------|
| 00 | `00-INDEX.md` | This file — the map + cross-cutting flows | — | the big picture |
| 01 | `01-ai.md` | `packages/ai` | 733 | how LLM calls are normalized across providers |
| 02 | `02-tui.md` | `packages/tui` | 225 | how the terminal renders without flicker |
| 03 | `03-agent.md` | `packages/agent` | 548 | the core agent loop, sessions, compaction |
| 04 | `04-coding-agent-core.md` | `coding-agent/src/core` | 417 | tools, extensions, the engine wrapping the agent |
| 05 | `05-coding-agent-modes.md` | `coding-agent/src/modes` + cli | 507 | the interactive UI, RPC protocol, CLI |
| 06 | `06-coding-agent-utils-examples-docs.md` | utils + examples + docs | 224 | how to extend pi; where the user docs live |
| 07 | `07-root-tooling-ci.md` | build, release, CI, `.pi` | 353 | how it builds, ships, and is hardened |

---

## 4. Cross-cutting flow #1: a user prompt → model → tool → answer

This is the single most important end-to-end path. It spans all four packages.

1. **Keystroke → input** (tui + coding-agent modes). Raw stdin is framed by
   `StdinBuffer`, decoded by `keys.ts` (Kitty/xterm/legacy protocols), and fed
   to the `Editor`/`CustomEditor`. On submit, the interactive controller
   (`modes/interactive/interactive-mode.ts`) rendezvouses the editor's
   `onSubmit` with its `run()` loop via `getUserInput()`. → see **05**.

2. **Slash commands & input assembly** (coding-agent). Built-in slash commands
   are handled in the editor submit handler; otherwise the text becomes a user
   message. The active `AgentSession` (core) receives it. → see **04**, **05**.

3. **Session → agent loop** (coding-agent core → agent). `AgentSession` wraps
   the lower-level `Agent`/`runAgentLoop`. The loop's `runLoop`
   (`agent-loop.ts:155`) is two nested loops: the inner drains tool calls and
   steering input; the outer restarts on follow-ups. → see **03**.

4. **Context → wire format → provider** (agent → ai). `streamAssistantResponse`
   (`agent-loop.ts:275`) is the single `AgentMessage[] → Message[]` boundary. It
   calls into `pi-ai`'s `stream()` (`stream.ts:40`), which resolves the model's
   `api` from the registry, applies the shared `transformMessages` (image
   downgrade, thinking flattening, tool-id normalization), converts `Context` to
   the provider's wire shape, and opens a stream. → see **01**.

5. **Streaming back** (ai → agent → coding-agent → tui). The provider's chunks
   are reduced into the unified event protocol
   (`{start, text/thinking/toolcall_*, done|error}`) over an
   `AssistantMessageEventStream`. The agent loop surfaces these as
   `AgentSessionEvent`s; the interactive controller's `handleEvent` switch
   mutates components and calls `ui.requestRender()`; the tui diffs lines and
   flushes minimal cursor moves. **Invariant:** post-invocation errors are never
   thrown — they arrive as a terminal `error` event + partial message. → **01**, **03**, **05**, **02**.

6. **Tool calls** (agent → coding-agent core). When the assistant emits a tool
   call, the loop's prepare→execute→finalize pipeline (with
   `beforeToolCall`/`afterToolCall` hooks) dispatches it. Built-in tools live in
   `core/tools/` (read, bash, edit, write, grep, find, ls). Results are fed back
   as tool messages and the loop continues. Parallel tools emit end-events in
   completion order but persist results in source order; one `sequential` tool
   forces the whole batch serial. → see **03**, **04**.

7. **Persistence** (agent + coding-agent core). Sessions are an append-only
   **branching tree** of entries (JSONL v3, `parentId` links), not a flat list.
   The flat transcript is derived via `buildSessionContext`. → see **03**, **04**.

---

## 5. Cross-cutting flow #2: extensibility (why pi is "self-extensible")

Extensions are TypeScript modules loaded at runtime via **jiti**, discovered from
`.pi/extensions`, `~/.pi/agent/extensions`, and CLI paths, run by
`ExtensionRunner`. They can register tools, slash commands, keyboard shortcuts,
CLI flags, and LLM providers, and subscribe to lifecycle events. A registered
`ToolDefinition` (typebox-schema'd) becomes a model-callable `AgentTool` — that's
the self-extension mechanism.

The killer idiom (best taught from `examples/extensions/`): **tool-operations
injection**. Built-in tool factories (`createBashTool`, `createReadTool`, …)
accept `{ operations }`, so the `sandbox/` and `gondolin/` examples keep pi's
renderers and schemas while rerouting actual I/O into a sandboxed shell or a
Linux micro-VM. → see **06** (examples) and **04** (extension system internals).

**Security model:** there is NO built-in sandbox. Tools and extensions run with
full user privileges. "Project trust" (`~/.pi/agent/trust.json`) only gates
loading of project-local input/config. Isolation is the user's responsibility
(containerize / Gondolin / OpenShell — see `docs/containerization.md`). → **04**, **06**.

---

## 6. Cross-cutting flow #3: build → release → publish

- **Build** (dependency-ordered): `tui → ai → agent → coding-agent`, each via
  `tsgo` (TypeScript native preview); source imports `./x.ts` and emits `.js`
  thanks to `rewriteRelativeImportExtensions` + `erasableSyntaxOnly`.
- **Binaries**: `build-binaries.sh` packs six-platform single-file **bun**
  executables from the emitted `dist/bun/cli.js`.
- **Quality gate** (`npm run check`): biome (`--error-on-warnings`) + four custom
  checks (pinned-deps, ts-relative-imports, shrinkwrap freshness, browser-smoke)
  + `tsgo --noEmit`.
- **Supply-chain**: `.npmrc` `save-exact` + `min-release-age`; coding-agent
  shrinkwrap with a hard install-script allowlist; husky pre-commit blocks
  lockfile commits unless `PI_ALLOW_LOCKFILE_CHANGE=1`; daily `npm audit`.
- **Release**: `release.mjs` is tag-driven; the `v*` tag triggers CI whose
  `publish-npm` job publishes via **OIDC trusted publishing** with `--provenance`,
  idempotently. → see **07**.

---

## 7. Key concept glossary

- **`Context` / `AssistantMessage`** (ai) — the unified message model all
  providers are normalized to.
- **API vs provider** (ai) — an *api* is a wire protocol (`anthropic-messages`,
  `openai-responses`, …); a *provider* is a named endpoint/account mapping onto
  an api. Two registries: model catalog + api-provider registry.
- **Signatures** (ai) — `textSignature`/`thinkingSignature`/`thoughtSignature`
  preserve cross-provider reasoning continuity.
- **Agent loop** (agent) — stateless `runAgentLoop` at the bottom; the stateful
  `Agent` class; the durable `AgentHarness` on top (which calls the loop
  directly, not `Agent`).
- **Session tree** (agent/coding-agent) — append-only branching JSONL; flat
  transcript is *derived*, not stored. Compaction drops pre-`firstKeptEntryId`
  entries from context but keeps them on disk.
- **Hooks vs subscribe** (agent) — `subscribe()` is passive (all events);
  typed `on(type)` hooks are result-producing (can mutate/block/cancel) and
  bridge into `transformContext`/`beforeToolCall`/`afterToolCall`.
- **ExecutionEnv** (agent) — fs/shell ops never throw; failures are typed
  `Result`s.
- **Tool-operations injection** (coding-agent) — inject `{ operations }` into
  built-in tool factories to reroute I/O (sandboxing).
- **Differential rendering** (tui) — components return `string[]`; the engine
  diffs against `previousLines` and emits minimal cursor moves inside
  synchronized-output (`CSI 2026`) for atomic, flicker-free updates.
- **CURSOR_MARKER** (tui) — `\x1b_pi:c\x07` APC marker components emit to place
  the hidden hardware cursor (for IME), distinct from the drawn fake cursor.

---

## 8. Suggested learning paths

- **"I want to understand the whole thing"**: 00 → 01 → 03 → 04 → 05 → 02 → 06 → 07.
- **"I want to add a provider/model"**: 01 (providers, registries, generate-models)
  → 07 (`models.generated.ts` is generated; edit `scripts/generate-models.ts`).
- **"I want to write an extension"**: 06 (examples) → 04 (extension system) →
  `docs/extensions.md` (authoritative, ~100KB).
- **"I want to work on the UI"**: 02 (tui engine) → 05 (interactive components,
  themes, keybindings).
- **"I want to ship a release"**: 07 → `AGENTS.md` "Releasing" section.

---

## 9. Repo-wide facts worth memorizing

- Source dirs use **erasable TypeScript only** (Node strip-only mode): no `enum`,
  `namespace`, parameter properties, `import =`. (`AGENTS.md`.)
- `packages/ai/src/models.generated.ts` is generated — never edit by hand; change
  `packages/ai/scripts/generate-models.ts` and regenerate.
- Changelogs are per-package (`packages/*/CHANGELOG.md`); new entries go under
  `## [Unreleased]`; released sections are immutable.
- Multiple pi sessions may share a cwd; git hygiene is strict (stage explicit
  paths, never `git add -A`, never `--no-verify`).
- Contributor model is unusually strict: new-contributor issues/PRs are
  auto-closed unless gated by `.github/APPROVED_CONTRIBUTORS` (maintainer
  `lgtm`/`lgtmi` mutates it). → **07**.
