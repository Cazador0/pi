# Study Notes 06 — coding-agent: utils, examples, docs

Scope: `packages/coding-agent/src/utils/*` (29 files), `packages/coding-agent/examples/*` (extensions + sdk + rpc client), and the prose docs in `packages/coding-agent/docs/*.md`. This is the "support layer + teaching material" slice. Core (`core/`) and modes (`modes/`) are documented by other agents; references to them here are informational.

All paths are absolute from `/home/user/pi/packages/coding-agent`.

---

## 1. Utilities (`src/utils/*`)

Grouped by theme. "Key exports" lists the public surface; `file:line` cites the definition.

### Process / shell / child processes

| File | Purpose | Key exports |
|------|---------|-------------|
| `src/utils/child-process.ts` | Cross-platform spawn wrappers (uses `cross-spawn` on win32) plus a robust process-wait that survives inherited stdio handles on Windows daemons. | `spawnProcess` (`:18`/`:24`), `spawnProcessSync` (`:28`), `waitForChildProcess` (`:46`) — the latter races `exit`/`close`/stdio-`end` with a 100ms `EXIT_STDIO_GRACE_MS` fallback (`:16`). |
| `src/utils/shell.ts` | Resolve a bash `ShellConfig` per-platform (Git Bash → PATH → `/bin/bash` → `sh`), build shell env with pi's bin dir prepended, sanitize binary output for `string-width`, and track/kill detached child process trees. | `getShellConfig` (`:57`), `getShellEnv` (`:112`), `sanitizeBinaryOutput` (`:134`), `trackDetachedChildPid`/`untrackDetachedChildPid`/`killTrackedDetachedChildren` (`:172`–`:184`), `killProcessTree` (`:190`, `taskkill /T` on win32, `kill(-pid)` process-group SIGKILL on unix). |
| `src/utils/sleep.ts` | Abort-aware delay. | `sleep(ms, signal?)` (`:4`) — rejects with `Error("Aborted")` on signal. |
| `src/utils/open-browser.ts` | Open URL/file in OS default handler **without a shell** (security: avoids `cmd /c start` metachar re-parsing). | `openBrowser` (`:10`) — `open`/`rundll32 url.dll`/`xdg-open`, best-effort `.unref()`. |

### Filesystem / paths

| File | Purpose | Key exports |
|------|---------|-------------|
| `src/utils/paths.ts` | Path normalization toolkit: realpath canonicalization, `~`/unicode-space/`@`-prefix/`file://` handling, cwd-relative formatting, and cloud-sync ignore xattrs. | `canonicalizePath` (`:28`), `isLocalPath` (`:41`, rejects npm:/git:/github:/http(s):/ssh:), `normalizePath` (`:57`), `resolvePath` (`:81`), `getCwdRelativePath` (`:87`), `formatPathRelativeToCwdOrAbsolute` (`:98`), `markPathIgnoredByCloudSync` (`:103`, Dropbox/fileprovider xattrs). |
| `src/utils/fs-watch.ts` | Thin `fs.watch` helpers that never throw. | `closeWatcher` (`:5`), `watchWithErrorHandler` (`:17`), const `FS_WATCH_RETRY_DELAY_MS=5000` (`:3`). |
| `src/utils/windows-self-update.ts` | Quarantine loaded native `.node`/`.dll` shared objects under a `node_modules/.pi-native-quarantine` dir so Windows self-update can replace files held open. Uses `process.report.getReport().sharedObjects`. | `cleanupWindowsSelfUpdateQuarantine` (`:50`), `quarantineWindowsNativeDependencies` (`:62`). |

### Git / sources / versioning

| File | Purpose | Key exports |
|------|---------|-------------|
| `src/utils/git.ts` | Parse git URLs (scp-like `git@host:path`, protocol URLs, shorthand) into a validated `GitSource`, splitting `@ref` and rejecting path-traversal/unsafe parts. Uses `hosted-git-info`. | `parseGitUrl` (`:172`), type `GitSource` (`:6`). Security helper `hasUnsafeGitInstallPart` (`:84`) blocks `\0`, `\`, leading `/`, `..` segments. |
| `src/utils/version-check.ts` | Compare semver-ish versions and poll `https://pi.dev/api/latest-version` for updates (honors `PI_OFFLINE`/`PI_SKIP_VERSION_CHECK`). | `comparePackageVersions` (`:32`), `isNewerPackageVersion` (`:48`), `getLatestPiRelease` (`:56`), `getLatestPiVersion` (`:89`), `checkForNewPiVersion` (`:96`). |
| `src/utils/changelog.ts` | Parse `CHANGELOG.md` `## [x.y.z]` sections and diff against last-seen version. | `parseChangelog` (`:14`), `compareVersions` (`:76`), `getNewEntries` (`:85`); re-exports `getChangelogPath` from `../config.ts` (`:99`). |
| `src/utils/pi-user-agent.ts` | Build HTTP UA string `pi/<v> (<platform>; node|bun/<v>; <arch>)`. | `getPiUserAgent` (`:1`). |

### Tooling / external binaries

| File | Purpose | Key exports |
|------|---------|-------------|
| `src/utils/tools-manager.ts` | Locate or download `fd`/`rg` (system PATH → pi bin dir → GitHub release archive). Handles tar.gz/zip extraction cross-platform, Termux `pkg` hints, Android skip, `PI_OFFLINE`. | `getToolPath` (`:85`), `ensureTool` (`:326`). `TOOLS` config table (`:29`) with per-OS asset-name resolvers; pins fd 10.3.0 on darwin/x64 (`:250`). |

### Formatting / text / markup

| File | Purpose | Key exports |
|------|---------|-------------|
| `src/utils/ansi.ts` | Strip ANSI/OSC/CSI sequences (vendored from `strip-ansi`/`ansi-regex`). Fast path skips strings without ESC/CSI introducers. | `stripAnsi` (`:46`). |
| `src/utils/html.ts` | Decode the 5 named HTML entities plus numeric `&#..;`/`&#x..;`. | `decodeHtmlEntity` (`:13`), `decodeHtmlEntityAt` (`:38`). |
| `src/utils/syntax-highlight.ts` | Wrap highlight.js: convert its HTML `<span class="hljs-...">` output into a theme-driven string by walking spans and applying per-scope `HighlightFormatter`s (with `.`/`-` prefix fallback and entity decoding). | `highlight` (`:134`), `renderHighlightedHtml` (`:80`), `supportsLanguage` (`:144`), types `HighlightTheme`/`HighlightOptions`. |
| `src/utils/highlight-js-lib-index.d.ts` | Ambient module decl for `highlight.js/lib/index.js`. | (types only) |
| `src/utils/frontmatter.ts` | Parse leading `---` YAML frontmatter (CRLF-normalized) into `{frontmatter, body}`. Used by skills/agents/prompts loaders. | `parseFrontmatter<T>` (`:28`), `stripFrontmatter` (`:39`). |
| `src/utils/json.ts` | Strip `//` comments and trailing commas from JSONC while preserving string literals (single regex pass). | `stripJsonComments` (`:2`). |
| `src/utils/deprecation.ts` | Emit each deprecation warning once (yellow chalk). | `warnDeprecation` (`:5`), `clearDeprecationWarningsForTests` (`:12`). |

### Clipboard

| File | Purpose | Key exports |
|------|---------|-------------|
| `src/utils/clipboard.ts` | Copy **text** to clipboard with a layered strategy: native addon (non-linux only) → platform tools (`pbcopy`/`clip`/`wl-copy`/`xclip`/`xsel`/`termux-clipboard-set`) → OSC 52 escape (for SSH/remote, capped at 100k encoded). | `copyToClipboard` (`:35`). |
| `src/utils/clipboard-native.ts` | Lazy-require the `@mariozechner/clipboard` native addon from two resolution roots; null on Termux/headless-linux. | `loadClipboardNative` (`:17`), const `clipboard` (`:30`), type `ClipboardModule`. |
| `src/utils/clipboard-image.ts` | Read an **image** from clipboard across Wayland (`wl-paste`), X11 (`xclip` TARGETS), WSL (PowerShell `Clipboard.GetImage` via temp file), and native addon. Converts unsupported formats to PNG via Photon. | `readClipboardImage` (`:254`), `isWaylandSession` (`:22`), `extensionForImageMimeType` (`:30`), type `ClipboardImage`. |

### Image processing (Photon WASM)

| File | Purpose | Key exports |
|------|---------|-------------|
| `src/utils/photon.ts` | Load `@silvia-odwyer/photon-node`, patching `fs.readFileSync` so the baked-in `photon_rs_bg.wasm` path resolves next to the executable in Bun-compiled binaries. Lazy + cached. | `loadPhoton` (`:116`), type `PhotonImageType`. |
| `src/utils/exif-orientation.ts` | Parse EXIF orientation (1–8) from JPEG/WebP TIFF blocks and apply flips/rotations to a Photon image. | `applyExifOrientation` (`:147`). |
| `src/utils/image-convert.ts` | Convert any image to PNG (Kitty graphics needs `f=100`), applying EXIF orientation. | `convertToPng` (`:8`). |
| `src/utils/image-resize-core.ts` | In-process resize-to-fit (maxWidth/Height 2000, maxBytes 4.5MB below Anthropic's 5MB limit). Tries PNG+JPEG at descending quality, then shrinks dims ×0.75 to 1×1. | `resizeImageInProcess` (`:59`), types `ImageResizeOptions`/`ResizedImage`. |
| `src/utils/image-resize-worker.ts` | Worker-thread entrypoint wrapping the core resize; one-shot `parentPort` message. | (default worker, `isResizeImageWorkerRequest` guard `:15`). |
| `src/utils/image-resize.ts` | Public resize API: runs Photon in a worker thread (so WASM doesn't block the TUI loop), falls back in-process if the worker can't load (Bun edge cases). Also a coordinate-mapping note builder. | `resizeImage` (`:85`), `formatDimensionNote` (`:116`). |
| `src/utils/mime.ts` | Sniff supported image MIME from magic bytes (rejects animated PNG via `acTL`, CMYK JPEG `0xF7`). | `detectSupportedImageMimeType` (`:6`), `detectSupportedImageMimeTypeFromFile` (`:22`). |

---

## 2. Example extensions (`examples/extensions/*`)

The single best teaching corpus for the extension API. Every extension is `export default function (pi: ExtensionAPI) { ... }`. The dir `README.md` is itself a categorized index (lifecycle/safety, custom tools, commands/UI, git, compaction, providers, deps) and ends with two load-bearing idioms:
- **Use `StringEnum([...])` from `@earendil-works/pi-ai`** for string-literal params (Type.Union breaks Google API).
- **Persist state in tool-result `details`** (not external files) so forking/branching reconstructs state correctly; rebuild on `session_start` by walking `ctx.sessionManager.getBranch()`.

### The multi-file/dependency examples (the task's focus)

**`sandbox/`** (`index.ts`, with `@anthropic-ai/sandbox-runtime` dep) — OS-level sandboxing (sandbox-exec/bubblewrap).
- API exercised: `pi.registerFlag("no-sandbox")` (`:202`), `pi.registerTool({...localBash, execute})` to **override the built-in `bash`** (`:214`), `pi.on("user_bash")` to sandbox `!` commands (`:229`), `session_start`/`session_shutdown` for `SandboxManager.initialize/reset`, `pi.registerCommand("sandbox")`.
- Idioms: imports `createBashTool`, `getAgentDir`, type `BashOperations` from the package; injects a custom `operations.exec` (`createSandboxedBashOps` `:132`) that wraps the command via `SandboxManager.wrapWithSandbox` and spawns detached with process-group SIGKILL on abort/timeout. Config merged global+project (`.pi/sandbox.json`). `ctx.ui.setStatus`/`notify`/`theme.fg`.

**`gondolin/`** (`@earendil-works/gondolin` dep) — route **all** built-in tools into a local micro-VM (host cwd mounted at `/workspace`).
- API exercised: overrides `read/write/edit/bash/grep/find/ls` by spreading the local tool and swapping in a VM-backed `operations` object (`createGondolin*Ops`), `user_bash` hook, and `before_agent_start` to rewrite the cwd line in the system prompt to point at `/workspace` (`:522`).
- Idioms: imports the full tool-factory surface (`createReadTool`, `createGrepTool`, ... `truncateHead`, `truncateLine`, `formatSize`, `DEFAULT_MAX_BYTES`, the `*Operations`/`*ToolDetails`/`*ToolInput` types). Re-implements grep against the VM fs (`executeGondolinGrep` `:239`) honoring limits/truncation. Lazy VM start guarded by `ensureVm` promise (`:400`); host↔guest path mapping (`toGuestPath` `:74`). This is the canonical example for "tool operations injection."

**`subagent/`** (`index.ts`, `agents.ts`, `agents/*.md`, `prompts/*.md`) — delegate to specialized agents in isolated `pi` subprocesses.
- API exercised: one big `pi.registerTool({name:"subagent", parameters, execute, renderCall, renderResult})`. Modes single/parallel/chain, with `agentScope` (`StringEnum`) and `confirmProjectAgents` gating project-controlled agents behind `ctx.ui.confirm`.
- Idioms: spawns `pi --mode json -p --no-session [--model] [--tools] [--append-system-prompt tmpfile]` and parses `message_end`/`tool_result_end` JSONL events for streaming `onUpdate`. `getPiInvocation` (`:243`) resolves how to re-invoke pi (execPath vs `pi` on PATH, handles Bun virtual `/$bunfs/root/` script). Concurrency limiter (`mapWithConcurrencyLimit` `:213`, max 8 tasks/4 concurrent). Heavy custom rendering with `@earendil-works/pi-tui` `Container`/`Text`/`Markdown`/`Spacer` + `getMarkdownTheme`, `withFileMutationQueue` for temp prompt files (`:237`). `agents.ts` discovers `~/.pi/agent/agents/*.md` (user) and `.pi/agents/*.md` (project) via `parseFrontmatter` (`name`/`description`/`tools`/`model`).

**`custom-provider-anthropic/`** (`@anthropic-ai/sdk` dep) — full from-scratch provider.
- API exercised: `pi.registerProvider("custom-anthropic", { baseUrl, apiKey:"$ENV", api, models[], oauth{login,refreshToken,getApiKey}, streamSimple })` (`:569`).
- Idioms: implements PKCE OAuth against claude.ai (`loginAnthropic` `:78`), token refresh, and a complete `streamSimple` (`:334`) that translates pi `Context`/`Message` ↔ Anthropic blocks, builds an `AssistantMessageEventStream` via `createAssistantMessageEventStream`, and pushes `start/text_*/thinking_*/toolcall_*/done/error` events. Demonstrates OAuth "Claude Code stealth" tool-name remapping (`toClaudeCodeName`/`fromClaudeCodeName` `:172`) and `claude-code-20250219` beta headers. The deep reference for the provider streaming contract.

**`custom-provider-gitlab-duo/`** (`index.ts`, `test.ts`) — provider that **delegates** to pi-ai's built-in streamers.
- API exercised: same `registerProvider` shape, but `streamGitLabDuo` (`:306`) calls `streamSimpleAnthropic`/`streamSimpleOpenAIResponses` from `@earendil-works/pi-ai` over GitLab's AI-Gateway proxy URLs, injecting a cached "direct access" token + headers. Shows the much simpler path: reuse built-in API streaming and only handle auth/routing. Exports `MODELS`/`streamGitLabDuo` for its `test.ts`.

**`plan-mode/`** (`index.ts`, `utils.ts`) — Claude-Code-style read-only plan mode.
- API exercised: `registerFlag("plan")`, `registerCommand("plan"/"todos")`, `registerShortcut(Key.ctrlAlt("p"))`, `pi.setActiveTools([...])` to swap toolsets, `tool_call` (block non-allowlisted bash), `context` (filter stale plan messages), `before_agent_start` (inject hidden `[PLAN MODE ACTIVE]` message), `turn_end`/`agent_end` (extract numbered steps, track `[DONE:n]`), `pi.appendEntry`/`sendMessage`/`sendUserMessage`, and `session_start` restore from `ctx.sessionManager.getEntries()`. UI: `setStatus`/`setWidget`/`ui.select`/`ui.editor`. `utils.ts` is the pure, testable allow/deny command classifier and step extractor.

**`with-deps/`** (`ms` dep) — minimal extension with its own `package.json`/`node_modules`, proving **jiti resolves modules from the extension's own dir**. One `registerTool("parse_duration")`.

**`dynamic-resources/`** (`index.ts` + `SKILL.md`/`dynamic.md`/`dynamic.json`) — `pi.on("resources_discover")` returns `{skillPaths, promptPaths, themePaths}` (`:8`) so an extension can contribute skills/prompts/themes at runtime.

**`doom-overlay/`** (`index.ts`, `doom-component/engine/keys.ts`, `wad-finder.ts`, WASM build) — DOOM at 35 FPS as an overlay.
- API exercised: `registerCommand("doom-overlay")` → `ctx.ui.custom((tui,theme,keybindings,done)=>new Component, { overlay:true, overlayOptions:{width,maxHeight,anchor,margin} })` (`:53`). Guards `ctx.mode !== "tui"`. Persistent engine across invocations. The reference for **real-time custom overlay components**.

### Representative single-file extensions (hook coverage)

| File | Hook / API it teaches |
|------|----------------------|
| `hello.ts` | `defineTool(...)` + `pi.registerTool` — minimal tool. |
| `permission-gate.ts` | `pi.on("tool_call", ...)` returns `{block:true, reason}`; uses `ctx.hasUI` + `ctx.ui.select`. |
| `tool-override.ts` | Register a tool named `read` to **override the built-in**; no `renderCall/renderResult` → falls back to built-in renderer. Uses `getAgentDir`, `withFileMutationQueue`. |
| `structured-output.ts` | `terminate: true` in a tool result to end the turn without a follow-up LLM call; `promptSnippet`/`promptGuidelines`; custom `renderResult`. |
| `message-renderer.ts` | `pi.registerMessageRenderer("status-update", (msg,{expanded},theme)=>Box)` + `pi.sendMessage({customType, details})`. |
| `custom-compaction.ts` | `pi.on("session_before_compact")` returns `{compaction:{summary, firstKeptEntryId, tokensBefore}}`; uses `ctx.modelRegistry.find`/`getApiKeyAndHeaders`, `complete`, `convertToLlm`/`serializeConversation`. |
| `event-bus.ts` | `pi.events.on/emit` for inter-extension messaging. |
| `commands.ts` | `pi.getCommands()`, `getArgumentCompletions`, `SlashCommandInfo`. |
| `todo.ts` | State-in-`details` pattern + a `/todos` UI component (`matchesKey`, `truncateToWidth`). |

Other notable ones referenced in the README index: `git-checkpoint.ts`/`auto-commit-on-exit.ts` (git), `dirty-repo-guard.ts`/`confirm-destructive.ts`/`protected-paths.ts`/`project-trust.ts` (safety + `project_trust` event), `ssh.ts` (tool delegation over SSH), `dynamic-tools.ts` (register tools post-startup), `trigger-compact.ts`, `notify.ts` (OSC 777), `titlebar-spinner.ts`, `status-line.ts`, `custom-footer.ts`/`custom-header.ts`, `modal-editor.ts`/`rainbow-editor.ts` (`setEditorComponent`), `snake.ts`/`tic-tac-toe.ts`/`space-invaders.ts` (games), `rpc-demo.ts` (paired with the RPC client below).

---

## 3. SDK examples (`examples/sdk/`) — driving pi programmatically

Entry point is `createAgentSession(options)` returning `{ session }`. Run with `npx tsx examples/sdk/NN.ts`. Numbered ladder from defaults to full control:

| File | Teaches |
|------|---------|
| `01-minimal.ts` | `const { session } = await createAgentSession()`; `session.subscribe(event => ...)` filtering `message_update`→`text_delta`; `session.prompt(...)`; `session.dispose()` in `finally`. |
| `02-custom-model.ts` | `getModel(provider,id)` + `thinkingLevel`. |
| `03-custom-prompt.ts` | `DefaultResourceLoader({ systemPromptOverride })`, then `await loader.reload()`. |
| `04-skills.ts` | Discover/filter/replace skills via resource loader. |
| `05-tools.ts` | `tools: ["read","grep",...]` allowlist (matched across built-in/extension/custom); honors custom `cwd`. |
| `06-extensions.ts` | `DefaultResourceLoader({ additionalExtensionPaths, extensionFactories:[(pi)=>{...}] })` — inline extensions; includes a full example extension body in a comment. |
| `07-context-files.ts` | AGENTS.md context files. |
| `08-prompt-templates.ts` | File-based slash commands (README calls it `08-slash-commands.ts`). |
| `09-api-keys-and-oauth.ts` | `AuthStorage.create([path])`, `ModelRegistry.create/inMemory`, `authStorage.setRuntimeApiKey(provider, key)` (not persisted). |
| `10-settings.ts` | `SettingsManager.create/inMemory`, `applyOverrides`, `setDefaultThinkingLevel`, `flush()`, `drainErrors()`. |
| `11-sessions.ts` | `SessionManager.inMemory/create/continueRecent/open/list`; `session.sessionFile`/`sessionId`. |
| `12-full-control.ts` | Replace **everything**: custom `AuthStorage` path, `ModelRegistry.inMemory`, a hand-written `ResourceLoader` object (all `get*` return empty + `getSystemPrompt`), `createExtensionRuntime()`, `tools`, in-memory session+settings. No discovery. |
| `13-session-runtime.ts` | `createAgentSessionRuntime(factory, {...})` for replacing the **active** session (new/resume/fork/import). Key pattern: after `runtime.newSession()`/`switchSession()`, re-`bindExtensions` and re-`subscribe` to `runtime.session`. Uses `createAgentSessionServices`/`createAgentSessionFromServices`. |

The README's "Quick Reference" / "Options" / "Events" tables are the cheat-sheet (default `tools` = `["read","bash","edit","write"]`; events include `message_update`, `tool_execution_start/end`, `agent_end`).

**`examples/rpc-extension-ui.ts`** (top-level): a working **RPC client** — spawns `node dist/cli.js --mode rpc --no-session --no-extension --extension extensions/rpc-demo.ts`, speaks JSONL over stdin/stdout, and renders a real `@earendil-works/pi-tui` chat UI. Crucially shows how a host UI must answer `extension_ui_request` messages (`select`/`confirm`/`input`/`editor`/`notify`/`setStatus`/`setWidget`/`set_editor_text`) with `extension_ui_response`. Pair-read with `extensions/rpc-demo.ts`.

---

## 4. Documentation map (`docs/*.md`)

`docs/index.md` is the hub (Quick start, Start here, Customization, Programmatic usage, Reference, Platform setup, Development). `docs/docs.json` is the site nav/manifest. Per-file one-liners:

**Getting started / usage**
- `quickstart.md` — install (`npm i -g --ignore-scripts`, or `curl pi.dev/install.sh`), authenticate, first session.
- `usage.md` — interactive mode anatomy (header/messages/editor/footer), editor features (`@` files, image paste), slash commands, CLI reference.
- `index.md` — top-level documentation index / table of contents.

**Providers & models**
- `providers.md` — built-in subscription (OAuth `/login`) and API-key providers; auth file; cloud providers; resolution order.
- `models.md` — add custom providers/models via `~/.pi/agent/models.json` (Ollama/vLLM/LM Studio/proxies), supported APIs, per-model overrides, Anthropic/OpenAI compat.
- `custom-provider.md` — implement custom APIs/OAuth in an extension via `pi.registerProvider()`; points to the two `custom-provider-*` examples.

**Security & isolation**
- `security.md` — local trust boundary, **project trust** (what counts as a trust input, `trust.json`, non-interactive `--approve` behavior), `project_trust` event, vuln reporting.
- `containerization.md` — three isolation patterns: OpenShell (whole process), **Gondolin extension** (tools routed to micro-VM), plain Docker; tradeoffs table.

**Customization**
- `extensions.md` — the big one (~100KB): full extension authoring guide — locations, available imports, events (resource/session/agent/model/tool lifecycle), `ExtensionContext`/`ExtensionCommandContext`, `ExtensionAPI` methods, state mgmt, custom tools, custom UI, error handling, mode behavior, examples reference.
- `skills.md` — Agent Skills standard, locations, structure/frontmatter, validation, repositories.
- `prompt-templates.md` — Markdown `/name` prompt expansion; locations, frontmatter (`description`/`argument-hint`), positional args (`$1`, `$@`, `${@:N:L}`).
- `themes.md` — JSON theme files; locations, selection, color tokens/values.
- `packages.md` — bundle/share extensions+skills+prompts+themes as pi packages (npm/git; `pi` key in package.json or conventional dirs); install/filter/enable.
- `keybindings.md` — customize `~/.pi/agent/keybindings.json`; namespaced ids, key format, `/reload` to apply.

**Sessions & compaction**
- `sessions.md` — session storage, `-c`/`-r`/`--no-session`/`--name`/`--session`/`--fork`, `/session`.
- `session-format.md` — JSONL file format, entry types, tree via `id`/`parentId`, file location scheme, SessionManager API.
- `compaction.md` — auto-compaction vs branch summarization; triggers, source files, `session_before_compact` hook surface.

**Programmatic / integration**
- `sdk.md` — embed pi via `createAgentSession`; quick start, options, events; points to `examples/sdk/`.
- `rpc.md` — headless JSON-over-stdin/stdout protocol (`--mode rpc`); recommends in-process `AgentSession` for Node hosts; client at `src/modes/rpc/rpc-client.ts`.
- `json.md` — `--mode json` event-stream output; `AgentSessionEvent`/`AgentEvent`/message types; example `jq` pipeline.
- `tui.md` — `Component` interface and `@earendil-works/pi-tui` building blocks for custom extension UI.

**Platform / setup**
- `windows.md` — bash requirement and resolution order; custom `shellPath`.
- `termux.md` — Android/Termux install (Termux + Termux:API).
- `tmux.md` — `extended-keys on` so Shift/Ctrl+Enter survive.
- `terminal-setup.md` — Kitty keyboard protocol; per-terminal notes (Kitty/iTerm2 OK, Apple Terminal fallback).
- `shell-aliases.md` — enable aliases via `shellCommandPrefix` (`bash -c` is non-interactive).
- `settings.md` — global vs project `settings.json`, project-trust notes, full settings reference (model/thinking, compaction, retry, terminal, etc.).

**Development**
- `development.md` — local setup (`npm install && npm run build`, `pi-test.sh`), forking/rebranding via `piConfig`, path resolution rule (**always use `src/config.ts`, never `__dirname`**), `/debug` log, testing.

---

## 5. Gotchas & noteworthy idioms

- **Tool "operations" injection is the extension superpower.** `createBashTool`/`createReadTool`/… accept `{ operations }`; sandbox and gondolin keep pi's renderers/schemas while swapping where I/O actually happens (host → sandboxed shell → VM). Prefer this over reimplementing tools.
- **Overriding built-ins is by name.** Register a tool named `read`/`bash` to replace it; omit `renderCall`/`renderResult` to inherit the built-in renderer (see `tool-override.ts:127`, `sandbox/index.ts:214`).
- **State belongs in tool-result `details`, not files** — this is what makes fork/branch correct. Reconstruct on `session_start` from `getBranch()`/`getEntries()` (README idiom; `plan-mode` and `todo` both do this).
- **`StringEnum` not `Type.Union`** for string-literal params (Google API compatibility) — repeated everywhere (`agents`, `todo`, `subagent`).
- **`terminate: true`** lets a tool end the agent turn with no extra LLM round-trip (`structured-output.ts:42`).
- **`firstKeptEntryId` + `tokensBefore`** must be threaded back when returning a custom `compaction` result (`custom-compaction.ts:113`); return `undefined` to fall back to default compaction.
- **RPC hosts must service `extension_ui_request`** or interactive extensions hang; reply with `extension_ui_response` carrying `value`/`confirmed`/`cancelled` (`rpc-extension-ui.ts:377`).
- **Bun-compiled binaries break naive path/worker resolution.** `photon.ts` patches `fs.readFileSync` for the WASM blob; `image-resize.ts` tries a string worker path first under Bun; `subagent` special-cases `/$bunfs/root/` script paths. Watch for this pattern when adding native/worker assets.
- **Security-hardened helpers**: `open-browser.ts` never uses a shell; `git.ts` rejects path traversal in install sources; `clipboard.ts` caps OSC 52 at 100k and skips the native addon on Linux (X11-only crate doesn't keep Wayland selection ownership).
- **Worker thread for image resize** keeps Photon WASM off the TUI event loop, with graceful in-process fallback (`image-resize.ts:85`).
- **`process.report.getReport().sharedObjects`** is used to find loaded native libs for Windows self-update quarantine — an uncommon but useful Node API (`windows-self-update.ts:27`).
- **`changelog.ts:99` re-exports `getChangelogPath` from `../config.ts`** — a couple of utils intentionally proxy config to keep import sites tidy.
- **Subagent isolation** = a fresh `pi --mode json -p --no-session` subprocess per task; output is parsed from JSONL events and capped at 50KB/task to the parent model while full results stay in `details`.
