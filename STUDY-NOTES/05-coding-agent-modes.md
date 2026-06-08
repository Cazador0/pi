# 05 — Coding Agent: Run Modes, CLI, Interactive TUI, RPC & Themes

**Scope:** `packages/coding-agent/src/cli/*` and `packages/coding-agent/src/modes/**`
(interactive TUI, its components, theme system, and the RPC/print modes).
Out of scope (other agents): `core/`, `utils/`, `examples/`. We reference `core/*` only where the
modes call into it.

This slice is the *presentation + transport* layer. All business logic lives in
`core/agent-session.ts` (`AgentSession`) and `core/agent-session-runtime.ts`
(`AgentSessionRuntime`). The three modes (interactive TUI, RPC, print) are three different
"front ends" wired onto the same session object.

---

## 0. The big picture

```
            ┌─────────────────────── src/main.ts (entry orchestration) ────────────────────────┐
            │ parseArgs() → resolveAppMode() → build AgentSessionRuntime (core) → dispatch mode  │
            └──────────────┬──────────────────────┬──────────────────────────┬──────────────────┘
                           │                      │                          │
                   appMode="interactive"   appMode="rpc"            appMode="print"|"json"
                           │                      │                          │
              modes/interactive/             modes/rpc/                 modes/print-mode.ts
              interactive-mode.ts            rpc-mode.ts                runPrintMode()
              (InteractiveMode + TUI)        (JSON stdin/stdout)        (single-shot)
                           │
              wires packages/pi-tui (TUI, Editor, Container, Loader…)
              + interactive/components/*  + interactive/theme/theme.ts
```

`main.ts` is NOT in our scope but is the dispatcher; the modes are entered at
`main.ts:1001-1045`. `modes/index.ts` is the barrel that re-exports the public surface
(`InteractiveMode`, `runPrintMode`, `runRpcMode`, `RpcClient`, RPC types).

---

## 1. CLI entry & argument parsing (`src/cli/*`, `src/cli/args.ts`)

### 1.1 Boot sequence (in `main.ts`, calling into `cli/`)
The binary's `main()` does roughly (line refs in `main.ts`):
1. `handlePackageCommand` / `handleConfigCommand` — subcommands like `pi install`, `pi config`
   (`main.ts:703-709`). These short-circuit before normal arg parsing.
2. `parseArgs(args)` (`cli/args.ts:63`) → `Args` struct.
3. Report `parsed.diagnostics` (warnings/errors) and exit on error (`main.ts:712-720`).
4. **Mode resolution:** `resolveAppMode(parsed, process.stdin.isTTY)` (`main.ts:722`, fn at
   `main.ts:102-113`).
5. `--version` / `--export` / fast-exit paths (`main.ts:728-745`).
6. Build runtime: session manager, trust store, auth, model registry, `AgentSessionRuntime`.
7. `--list-models` prints and exits (`main.ts:953-955` → `cli/list-models.ts`).
8. Dispatch to the chosen mode (`main.ts:1001-1045`).

### 1.2 `parseArgs` (`cli/args.ts:63-210`)
A hand-rolled linear scanner over `argv` (no library). Returns the `Args` interface
(`cli/args.ts:12-55`). Notable parsing rules:
- `Mode = "text" | "json" | "rpc"` (`args.ts:10`); `--mode` only accepts those three (`args.ts:78-82`).
- `@file` tokens → `fileArgs` (the `@` is stripped, `args.ts:186-187`); positional non-flags →
  `messages`.
- `--print/-p` greedily consumes the next token as the prompt *unless* it looks like a flag or
  `@file` (`args.ts:140-146`).
- `--models` is comma-split into patterns for Ctrl+P cycling (`args.ts:114`).
- `--thinking` validated against `VALID_THINKING_LEVELS` = off/minimal/low/medium/high/xhigh
  (`args.ts:57-61, 130-139`); invalid → warning diagnostic, not fatal.
- **Unknown `--flags` are NOT errors** — they go into `unknownFlags: Map` (`args.ts:188-201`) so
  extensions can register custom CLI flags (e.g. plan-mode's `--plan`). Unknown *short* flags
  (`-x`) ARE errors (`args.ts:202-203`).
- `--approve/-a` and `--no-approve/-na` set `projectTrustOverride` true/false (`args.ts:180-183`).
- `printHelp(extensionFlags?)` (`args.ts:212`) renders the long help, splicing in extension-registered
  flags and the full env-var table.

### 1.3 Mode selection (`resolveAppMode`, `main.ts:102-113`)
Priority order:
1. `--mode rpc` → `"rpc"`.
2. `--mode json` → `"json"`.
3. `--print/-p` **OR stdin is not a TTY** → `"print"`. (Piping into `pi` auto-selects print mode.)
4. else → `"interactive"`.

`toPrintOutputMode` (`main.ts:115-117`) collapses `json`→json, everything else→text for
`runPrintMode`. Note `AppMode` ("interactive"|"print"|"json"|"rpc") is a superset of the CLI
`Mode` type.

### 1.4 Helper CLI files
| File | Role |
|---|---|
| `cli/args.ts` | Arg parsing + `printHelp`. The `Args` interface is the canonical CLI contract. |
| `cli/file-processor.ts` | `processFileArguments` — turn `@file` args into `<file name=…>` text blocks and `ImageContent[]` (auto-resizes images to ≤2000² via `resizeImage`, `file-processor.ts:57-68`). Exits process on missing/unreadable file. |
| `cli/initial-message.ts` | `buildInitialMessage` — concatenates stdin + `@file` text + first positional message into one initial prompt (`initial-message.ts:20-43`). **Mutates `parsed.messages` by shifting off the first.** |
| `cli/list-models.ts` | `listModels` — fuzzy-filtered (`fuzzyFilter` from pi-tui) tabular model dump for `--list-models [pat]`. |
| `cli/session-picker.ts` | `selectSession` — spins a throwaway `TUI` hosting `SessionSelectorComponent` for `--resume`; resolves to a path or null. |
| `cli/config-selector.ts` | `selectConfig` — throwaway `TUI` hosting `ConfigSelectorComponent` for `pi config`; inits theme + theme-watcher around it. |

`session-picker.ts` and `config-selector.ts` are good minimal examples of the
"throwaway TUI" pattern: `new TUI(new ProcessTerminal())`, `KeybindingsManager.create()` +
`setKeybindings(...)`, add a component as child, `ui.setFocus(...)`, `ui.start()`, and resolve a
Promise from callbacks that call `ui.stop()`.

---

## 2. Interactive mode (`modes/interactive/interactive-mode.ts`, ~5700 lines)

`InteractiveMode` (`interactive-mode.ts:265`) is the master controller. It owns the `TUI`, all
the layout containers, the editor, the footer, keybindings, and the agent-event subscription.

### 2.1 Construction & layout (`constructor` `:389-429`)
Holds a reference to `AgentSessionRuntime` (`runtimeHost`) and exposes the live `AgentSession` via
getters that always read `runtimeHost.session` (`:376-387`) — important because the session object
is *swapped* on `/new`, `/resume`, fork, etc. The constructor:
- Creates `TUI(new ProcessTerminal(), showHardwareCursor)` (`:400`).
- Builds the container tree (each is a pi-tui `Container`): `headerContainer`, `chatContainer`,
  `pendingMessagesContainer`, `statusContainer`, `widgetContainerAbove/Below`, `editorContainer`
  (`:402-407`).
- Creates `KeybindingsManager.create()` and calls global `setKeybindings(...)` (`:408-409`).
- Creates the `CustomEditor` (`./components/custom-editor.ts`) as `defaultEditor`; `editor` is the
  *active* editor (may be replaced by an extension editor) (`:412-417`).
- Builds `FooterDataProvider` + `FooterComponent` (`:419-421`).
- `setRegisteredThemes(...)` then `initTheme(settings.getTheme(), /*watch*/ true)` (`:427-428`).

### 2.2 Layout order (added to TUI in `init`, `:628-699`)
```
headerContainer        (logo + keybinding hints + changelog "What's New")
chatContainer          (the scrollback: messages, tool executions, bash, diffs…)
pendingMessagesContainer (queued steer/follow-up messages preview)
statusContainer        (working/compaction/retry loaders, transient status lines)
widgetContainerAbove   (extension widgets, placement "aboveEditor")
editorContainer        (the active editor)
widgetContainerBelow   (extension widgets, placement "belowEditor")
footer                 (cwd, tokens, context %, git branch, ext statuses)
```
`init()` (`:599-728`): registers signal handlers, loads changelog, **downloads `fd` and `rg`**
via `ensureTool` (`:609` — `fd` powers file autocomplete, `rg` powers grep), builds the header,
assembles the layout, `setupKeyHandlers()` + `setupEditorSubmitHandler()`, then `ui.start()`,
`rebindCurrentSession()` (binds extensions to this front-end), renders initial messages, and wires
`onThemeChange` + branch-change → `ui.requestRender()`.

### 2.3 The run loop (`run()` `:747-820`)
After `init()`, kicks off async background checks (new pi version, package updates, tmux keyboard
config), shows startup warnings, sends `initialMessage`/`initialMessages`, then enters the classic
loop:
```ts
while (true) {
  const userInput = await this.getUserInput();   // :812
  await this.session.prompt(userInput);          // delegates to core
}
```
**`getUserInput()` (`:3287-3299`) is a promise/callback bridge:** if a queued input exists it
returns immediately; otherwise it stores `onInputCallback` and resolves when the editor's
`onSubmit` fires (set in `setupEditorSubmitHandler`, `:2691-2695`). So the editor's submit handler
and the `run()` loop rendezvous through `onInputCallback` / `pendingUserInputs`.

### 2.4 Editor submit handler — the slash-command dispatcher (`:2512-2698`)
`defaultEditor.onSubmit = async (text) => {…}`. This is the *interactive* slash-command table
(built-in commands handled here, **not** by core). Sequence:
1. Big `if (text === "/foo")` ladder for built-in commands (`:2518-2644`). Each sets editor to ""
   and calls a `handle*`/`show*` method. Built-ins include: `/settings`, `/scoped-models`,
   `/model [search]`, `/export`, `/import`, `/share`, `/copy`, `/name`, `/session`, `/changelog`,
   `/hotkeys`, `/fork`, `/clone`, `/tree`, `/trust`, `/login`, `/logout`, `/new`, `/compact`,
   `/reload`, `/debug`, `/resume`, `/quit`, plus easter eggs `/arminsayshi`, `/dementedelves`.
2. **Bash mode:** `!cmd` runs bash (in context), `!!cmd` runs bash excluded from context
   (`:2646-2662`). Guarded against concurrent bash.
3. **During compaction:** extension commands run immediately; everything else is queued via
   `queueCompactionMessage(...)` (`:2664-2674`).
4. **During streaming:** anything else is sent through `session.prompt(text, {streamingBehavior:"steer"})`
   so core handles steering / extension-command / template expansion (`:2676-2685`).
5. **Normal case:** flush pending bash components into chat, then hand `text` to `onInputCallback`
   (the `run()` loop) or buffer in `pendingUserInputs` (`:2687-2696`).

The canonical list of built-ins is `BUILTIN_SLASH_COMMANDS` from `core/slash-commands.ts`
(used to build autocomplete, `:481`). `isExtensionCommand(text)` (`:3827-3835`) checks the
extension runner; `isSlashCommand`-style detection is just `text.startsWith("/")`.

### 2.5 Agent event → render bridge (`subscribeToAgent` / `handleEvent`, `:2700-...`)
`this.session.subscribe(event => this.handleEvent(event))` (`:2701`). `handleEvent`
(`:2706+`) is a big switch over `AgentSessionEvent` (defined in `core/agent-session.ts`). It is the
**heart of the render loop**: each event mutates the component tree and calls
`this.ui.requestRender()`. Key cases:

| Event | UI effect |
|---|---|
| `agent_start` (`:2714`) | clear `pendingTools`, optionally terminal progress bar, tear down retry loaders, start the "Working…" `Loader` in `statusContainer`. |
| `queue_update` (`:2741`) | refresh `pendingMessagesContainer`. |
| `session_info_changed` (`:2746`) | update terminal title + footer. |
| `thinking_level_changed` (`:2752`) | footer + editor border color. |
| `message_start` (`:2757`) | user/custom → append component; **assistant → create a streaming `AssistantMessageComponent`** and stash as `streamingComponent`/`streamingMessage`. |
| `message_update` (`:2779`) | feed partial assistant message into `streamingComponent.updateContent`; create/refresh `ToolExecutionComponent`s for each `toolCall` content, tracked in `pendingTools: Map<toolCallId, comp>`. |
| `message_end` (`:2814`) | finalize streaming component; on abort/error push error into pending tool components; else `setArgsComplete()` (triggers diff computation for edit tools). |
| `tool_execution_start/update/end` (`:2853-2894`) | mark started, stream partial result, finalize result + remove from `pendingTools`. |
| `agent_end` (`:2896`) | stop working loader, clear status, drop any leftover streaming component, check pending shutdown. |
| `compaction_start/end` (`:2917+`) | swap the escape handler to "abort compaction", show a dedicated compaction `Loader`. |

This handler is `async` and the very first thing it does is lazily `await this.init()` if the
agent fired before init finished (`:2707-2709`) — a guard for the startup race.

### 2.6 Key handling (`setupKeyHandlers`, `:2424-2488`)
Handlers are registered on `defaultEditor` so they work regardless of which editor is active
(they read `this.editor` for text). Two flavors:
- **Dynamic special handlers** (replaceable fields on `CustomEditor`): `onEscape`, `onCtrlD`,
  `onPasteImage`. `onEscape` (`:2427-2453`) is heavily overloaded: abort-stream →
  abort-bash → exit-bash-mode → **double-Escape (within 500ms) opens `/tree` or fork selector**
  based on `getDoubleEscapeAction()`.
- **App-action handlers** via `editor.onAction("app.xxx", fn)` (`:2456-2474`) — these bind to
  *named keybindings*, not raw keys. E.g. `app.clear`→`handleCtrlC`, `app.thinking.cycle`,
  `app.model.cycleForward/Backward`, `app.model.select`, `app.tools.expand`,
  `app.thinking.toggle`, `app.editor.external`, `app.message.followUp/dequeue`, `app.session.*`.
- `onChange` (`:2476-2482`) toggles bash-mode styling when text starts with `!`.
- Global `ui.onDebug` is wired so the debug hotkey works regardless of focus (`:2464`).

`handleCtrlC` (`:3311-3319`): double Ctrl+C within 500ms exits; single clears the editor.
`handleCtrlD` (`:3321-3324`): exits (only reachable when editor empty, enforced by `CustomEditor`).

### 2.7 Extension UI surface
The interactive mode implements a rich `ExtensionUIContext` (`createExtensionUIContext`, ~`:2012`)
giving extensions real TUI dialogs (select/confirm/input/editor), status, widgets (above/below the
editor), custom footer/header, custom editor component, autocomplete providers, and theme access.
Contrast with RPC mode, which implements the same interface but mostly as JSON messages or no-ops
(see §5.3). Extension shortcuts are registered in `setupExtensionShortcuts` (`:1658+`) and routed
through `CustomEditor.onExtensionShortcut`.

---

## 3. Interactive components (`modes/interactive/components/*`)

Barrel: `components/index.ts` (also re-exports helpers for extension authors). All are pi-tui
`Component`/`Container`/`Box` subclasses. Grouped by role:

### 3.1 Message / scrollback rendering
| File | Role |
|---|---|
| `assistant-message.ts` | `AssistantMessageComponent` — renders an assistant turn (markdown text + thinking blocks). Supports `hideThinkingBlock`, custom hidden-thinking label; emits OSC-133 shell-integration zone markers. Used for live streaming. |
| `user-message.ts` | `UserMessageComponent` — boxed user message (markdown), OSC-133 zones. |
| `custom-message.ts` | `CustomMessageComponent` — renders extension/custom message types, optionally via a custom `MessageRenderer` or embedded `Component`. |
| `tool-execution.ts` | `ToolExecutionComponent` — the workhorse: tool call header + args + (streaming) result, expand/collapse, inline image rendering, pending/success/error background. Tracked by `toolCallId`. |
| `diff.ts` | `renderDiff` / `RenderDiffOptions` — pretty unified-diff renderer (parses `+/-/space` prefixed lines) used by edit/write tool rendering. |
| `bash-execution.ts` | `BashExecutionComponent` — renders `!`/`!!` bash runs with truncated tail output (`truncateTail`). |
| `skill-invocation-message.ts` | `SkillInvocationMessageComponent` — collapsible render of a parsed `skill` block. |
| `branch-summary-message.ts` | `BranchSummaryMessageComponent` — boxed summary shown when navigating the session tree. |
| `compaction-summary-message.ts` | `CompactionSummaryMessageComponent` — boxed summary produced by context compaction. |

### 3.2 Input editor & footer
| File | Role |
|---|---|
| `custom-editor.ts` | `CustomEditor extends Editor` — adds app-keybinding dispatch on top of pi-tui's `Editor` (see §6). Holds `actionHandlers: Map<AppKeybinding, ()=>void>` and replaceable `onEscape/onCtrlD/onPasteImage/onExtensionShortcut`. |
| `footer.ts` | `FooterComponent` — single-line status: cwd (`~`-relativized), token counts, context-window %, git branch, extension statuses, auto-compact indicator. Reads from `AgentSession` + `FooterDataProvider`. |
| `keybinding-hints.ts` | Helpers `keyText/keyDisplayText/keyHint/rawKeyHint/formatKeyText` — resolve a keybinding name to its display string (mac `alt`→`option`) and theme it. The header/footer hint text is built from these. |

### 3.3 Selectors & dialogs (overlay components)
| File | Role |
|---|---|
| `model-selector.ts` | `ModelSelectorComponent` — fuzzy model picker (Ctrl+L / `/model`). |
| `scoped-models-selector.ts` | `ScopedModelsSelectorComponent` — manage the Ctrl+P cycling set (`/scoped-models`); supports reorder, enable-all, clear. |
| `session-selector.ts` | `SessionSelectorComponent` — resume/search sessions (used by both `/resume` and `cli/session-picker.ts`); threaded/recent/relevance sort, rename/delete. (~1023 lines.) |
| `session-selector-search.ts` | Search query parser/types (`SortMode`, `NameFilter`, `ParsedSearchQuery`) for the session selector. |
| `tree-selector.ts` | `TreeSelectorComponent` — the session DAG/tree navigator (`/tree`, double-Esc); fold/unfold, labels, filters. (~1251 lines, the largest component.) |
| `user-message-selector.ts` | `UserMessageSelectorComponent` — pick a prior user message to fork from (`/fork`). |
| `settings-selector.ts` | `SettingsSelectorComponent` — `/settings` TUI over `SettingsManager` values. |
| `config-selector.ts` | `ConfigSelectorComponent` — `pi config` resource enable/disable UI (extensions/skills/themes/prompts). |
| `trust-selector.ts` | `TrustSelectorComponent` — project-trust prompt (`/trust`). |
| `oauth-selector.ts` | `OAuthSelectorComponent` — provider OAuth login/logout (`/login`, `/logout`). |
| `login-dialog.ts` | `LoginDialogComponent` — API-key entry dialog (Focusable input + AbortController). |
| `theme-selector.ts` | `ThemeSelectorComponent` — theme picker (SelectList). |
| `thinking-selector.ts` | `ThinkingSelectorComponent` — thinking-level picker with per-level descriptions. |
| `show-images-selector.ts` | `ShowImagesSelectorComponent` — toggle inline image display mode. |

### 3.4 Extension-facing UI primitives
| File | Role |
|---|---|
| `extension-selector.ts` | `ExtensionSelectorComponent` — `pi.ui.select(...)` overlay. |
| `extension-input.ts` | `ExtensionInputComponent` — `pi.ui.input(...)` single-line dialog. |
| `extension-editor.ts` | `ExtensionEditorComponent` — `pi.ui.editor(...)` multi-line dialog. |

### 3.5 Loaders, decorations, misc
| File | Role |
|---|---|
| `bordered-loader.ts` | `BorderedLoader` — boxed spinner, optionally cancellable (AbortController). |
| `countdown-timer.ts` | `CountdownTimer` — interval-driven countdown (used for auto-retry backoff). |
| `dynamic-border.ts` | `DynamicBorder` — full-width horizontal rule, theme-colored. |
| `visual-truncate.ts` | `truncateToVisualLines` — wrap+truncate text to N visual lines with a skipped-count. |
| `armin.ts` / `daxnuts.ts` / `earendil-announcement.ts` | Easter-egg / branding image components (1-bit bitmap, hex pixel art, blog announcement image from `assets/clankolas.png`). |

---

## 4. Theme system (`modes/interactive/theme/`)

Files: `theme.ts` (logic, ~1237 lines), `dark.json`, `light.json` (built-ins),
`theme-schema.json` (JSON schema for editor tooling).

### 4.1 Theme definition
A theme is JSON validated by a TypeBox schema `ThemeJsonSchema` (`theme.ts:29-100`):
- `name`, optional `vars` (named color aliases), and a `colors` object with a **fixed set of
  required tokens** grouped as: core UI, backgrounds/content text, markdown, tool diffs, syntax
  highlighting (9 tokens), thinking-level border colors (6), and bash-mode (`theme.ts:33-92`).
- Color values are hex `#rrggbb`, a 256-color index (0-255 integer), a `vars` reference string, or
  `""` (= terminal default) (`ColorValueSchema`, `:22-25`).
- Optional `export` section for HTML-export backgrounds (`:93-99`).
- `ThemeColor` (foreground tokens) and `ThemeBg` (the six `*Bg` tokens) are the typed unions
  (`:106-159`).

### 4.2 The `Theme` class (`:322-421`)
Pre-renders every token to an ANSI escape string at construction, choosing **truecolor vs
256-color** based on `getCapabilities().trueColor` (`:577`). `fg(token, text)` /`bg(token, text)`
wrap text with the SGR sequence + a *scoped* reset (`\x1b[39m`/`\x1b[49m`, `:350-360`). Also
provides `bold/italic/underline/inverse/strikethrough` (via chalk) and helpers like
`getThinkingBorderColor(level)` and `getBashModeBorderColor()`. RGB→256 uses a weighted-Euclidean
nearest-color search over the 6×6×6 cube + grayscale ramp (`:213-252`).

### 4.3 Variable resolution
`resolveVarRefs` (`:289-305`) recursively dereferences `vars` with cycle detection (throws on
circular refs). `resolveThemeColors` maps every color through it before constructing the `Theme`.

### 4.4 Global theme + the Proxy gotcha
The active theme is stored on `globalThis[Symbol.for("@earendil-works/pi-coding-agent:theme")]`
(`:744-760`) so that **multiple module loaders (tsx + jiti in dev) share one instance**. The
exported `theme` is a `Proxy` that reads from globalThis on every access (`:749-755`) — it throws
`"Theme not initialized. Call initTheme() first."` if accessed before `initTheme`. This is why
`initTheme()` must run before any component renders.

### 4.5 Lifecycle functions
- `initTheme(name?, enableWatcher?)` (`:777-791`) — resolves default theme from terminal
  background (`getDefaultTheme`→`detectTerminalBackground` via `COLORFGBG`, `:714-737`), loads,
  sets global, optionally starts a watcher. Falls back to `dark` silently on invalid theme.
- `setTheme(name, enableWatcher?)` (`:793-814`) — runtime switch; returns `{success,error}`.
- `setThemeInstance(theme)` / `setRegisteredThemes(themes[])` — for extension/in-memory themes.
- **File watcher** (`startThemeWatcher`, `:829-900`): only watches *custom* themes (not built-in);
  debounced 100ms reload; re-reads from disk and fires `onThemeChangeCallback` → in interactive
  mode this is `ui.invalidate()` + re-render (`interactive-mode.ts:715-719`).
- TUI integration adapters: `getMarkdownTheme`, `getSelectListTheme`, `getEditorTheme`,
  `getSettingsListTheme`, `highlightCode`, `getLanguageFromPath` (`:1039-1238`) bridge theme
  colors into the pi-tui rendering primitives and cli-highlight.
- HTML export helpers: `getResolvedThemeColors`, `getThemeExportColors`, `ansi256ToHex`
  (`:920-1028`).

`themes.md` documents the same token set for end users; the code is the source of truth for which
tokens are *required*.

---

## 5. RPC mode (`modes/rpc/*`)

Four files: `rpc-types.ts` (protocol types), `jsonl.ts` (framing), `rpc-mode.ts` (server), and
`rpc-client.ts` (a Node subprocess client). Cross-ref `docs/rpc.md`.

### 5.1 Framing (`jsonl.ts`)
Strict JSONL: **LF-only** record delimiter. `serializeJsonLine(v)` = `JSON.stringify(v) + "\n"`
(`:10-12`). `attachJsonlLineReader(stream, onLine)` (`:21-58`) hand-rolls a `StringDecoder` buffer
that splits **only on `\n`** (stripping a trailing `\r`), deliberately *not* using Node `readline`
because readline also splits on U+2028/U+2029 which are valid inside JSON strings (`jsonl.ts:14-20`,
echoed in `rpc.md:28-37`). This is a real correctness gotcha for anyone writing a client.

### 5.2 Protocol types (`rpc-types.ts`)
- **`RpcCommand`** (`:19-69`): a discriminated union on `type`, each with optional `id` for
  request/response correlation. Categories: prompting (`prompt`/`steer`/`follow_up`/`abort`/
  `new_session`), state (`get_state`), model, thinking, queue modes, compaction, retry, bash,
  session (export/switch/fork/clone/get_messages/set_session_name…), and `get_commands`.
  `prompt` carries optional `streamingBehavior: "steer"|"followUp"`.
- **`RpcResponse`** (`:111-206`): `{id?, type:"response", command, success:true, data?}` per
  command, plus a catch-all error `{success:false, error}`. Strongly typed `data` per command.
- **`RpcSessionState`** (`:91-104`): the `get_state` payload (model, thinkingLevel, isStreaming,
  isCompacting, steering/followUp modes, session id/file/name, autoCompaction, message counts).
- **Extension UI** is bidirectional: `RpcExtensionUIRequest` (server→client, `:213-248`,
  select/confirm/input/editor/notify/setStatus/setWidget/setTitle/set_editor_text) and
  `RpcExtensionUIResponse` (client→server, `:255-258`).

### 5.3 RPC server (`rpc-mode.ts`, `runRpcMode`)
`runRpcMode(runtimeHost): Promise<never>` (`:53`) — never returns; keeps the process alive via a
hanging Promise (`:773`). Flow:
1. `takeOverStdout()` (output-guard) so stray writes can't corrupt the JSON stream; all output goes
   through `output()` → `writeRawStdout(serializeJsonLine(...))` (`:59-61`).
2. Builds an `ExtensionUIContext` that maps extension UI calls to RPC messages
   (`createExtensionUIContext`, `:135-310`). Interactive dialog methods use `createDialogPromise`
   (`:90-130`) which emits a request with a fresh UUID, registers it in `pendingExtensionRequests`,
   and resolves when a matching `extension_ui_response` arrives (with timeout + abort support).
   Many TUI-only capabilities (working indicator, custom footer/header, custom editor, theme
   switching, autocomplete) are deliberately **no-ops** in RPC (`:178-309`).
3. `rebindSession()` (`:316-360`) binds extensions to this mode (`mode:"rpc"`), subscribes to
   `session.subscribe(event => output(event))` so **all agent events stream to stdout**, and adds a
   backpressure hook via `waitForRawStdoutBackpressure()`.
4. `handleCommand(cmd)` (`:382-673`) is the big switch mapping each `RpcCommand` to an
   `AgentSession`/`runtimeHost` call and returning an `RpcResponse`. Notable: `prompt` responds
   *after preflight succeeds* (via `preflightResult` callback) so queued/immediate prompts both
   count as success, while errors surface only if preflight didn't already succeed (`:390-412`).
   `new_session`/`switch_session`/`fork`/`clone` call `rebindSession()` when not cancelled so the
   subscription follows the swapped session.
5. Input loop: `attachJsonlLineReader(process.stdin, handleInputLine)` (`:762-770`).
   `handleInputLine` (`:705-755`) parses JSON, routes `extension_ui_response` to the pending map,
   else dispatches the command, emits the response, and honors backpressure + shutdown.
6. Graceful shutdown on SIGTERM/SIGHUP/stdin-end (`:362-379, 681-703, 757-760`).

### 5.4 RPC client (`rpc-client.ts`)
`RpcClient` (`:54`) spawns `node <cliPath> --mode rpc …` (`:72-138`) and provides a typed async
method per command (`prompt`, `steer`, `setModel`, `compact`, `bash`, `fork`, `getMessages`,
`getCommands`, …). Internals:
- `send(cmd)` (`:514-563`) assigns `req_${n}` ids, writes a JSON line, and resolves the matching
  response from `pendingRequests` with a 30s timeout.
- `handleLine` (`:482-501`): a line with `type:"response"` + known id resolves a request; everything
  else is dispatched to `onEvent` listeners (i.e. it's an agent event).
- `waitForIdle`/`collectEvents`/`promptAndWait` (`:430-476`) — wait for the `agent_end` event.
- Robust process lifecycle: rejects pending requests on exit/error, `stop()` SIGTERM→SIGKILL after
  1s (`:143-165`).

This client is the recommended subprocess integration path; for in-process Node use, callers should
use `AgentSession` directly (per `rpc.md:5`).

---

## 6. Keybindings: from config to handler

Layered system spanning `core/keybindings.ts` (manager) and our `CustomEditor` (dispatch). Cross-ref
`docs/keybindings.md`.

1. **Definitions.** `KEYBINDINGS` (`core/keybindings.ts:63-202`) = pi-tui's `TUI_KEYBINDINGS`
   spread + all `app.*` actions, each `{defaultKeys, description}`. The `AppKeybindings` interface
   (`:13-55`) is module-augmented into pi-tui's `Keybindings` type (`:59-61`) so the whole app gets
   string-literal autocomplete for action names. Some defaults are platform-specific (e.g. suspend
   disabled on Windows, pasteImage `ctrl+v` vs `alt+v`).
2. **User overrides.** `KeybindingsManager.create(agentDir)` (`:348-352`) loads
   `<agentDir>/keybindings.json`, runs `migrateKeybindingsConfig` (renames legacy keys like
   `submit`→`tui.input.submit`, `:204-309`), and constructs over `TuiKeybindingsManager`. `reload()`
   re-reads the file (used by `/reload`).
3. **Global registration.** `InteractiveMode` calls `setKeybindings(this.keybindings)`
   (`interactive-mode.ts:409`) so helper functions like `keyText()` (`keybinding-hints.ts:34-36`,
   via `getKeybindings()`) resolve names→keys anywhere.
4. **Dispatch.** `CustomEditor.handleInput(data)` (`custom-editor.ts:30-79`) is where raw key bytes
   become actions:
   - extension shortcuts first, then `app.clipboard.pasteImage`;
   - `app.interrupt` (Escape) — only when autocomplete is NOT showing; uses dynamic `onEscape` else
     the registered handler;
   - `app.exit` (Ctrl+D) — only when editor empty;
   - then a loop over all other `actionHandlers` calling `keybindings.matches(data, action)`;
   - finally `super.handleInput(data)` for normal editing (pi-tui Editor handles `tui.editor.*`).
5. **Binding handlers.** `setupKeyHandlers` (`interactive-mode.ts:2456-2474`) registers the action
   callbacks via `editor.onAction("app.xxx", fn)`. So the path is:
   `keybindings.json` → `KeybindingsManager` → `CustomEditor.handleInput` matches the action →
   `actionHandlers.get(action)()` → an `InteractiveMode` method.

`getKeys("app.model.cycleForward")` etc. (`:619`) is used to render hints; the same manager is
injected into selectors so overlay components honor the same bindings (`getKeybindings()` in each
selector).

---

## 7. Key types / data structures (file:line)

- `Args` (`cli/args.ts:12-55`) — parsed CLI contract; `Mode` (`:10`).
- `AppMode` (`main.ts:100`) — runtime mode superset; `resolveAppMode` (`:102-113`).
- `InteractiveModeOptions` (`interactive-mode.ts:248-263`) — startup options (initial messages,
  migrated providers, verbose…).
- `InteractiveMode` private state of note: `chatContainer`/`statusContainer`/`pendingMessagesContainer`
  (`:268-269`), `pendingTools: Map<string, ToolExecutionComponent>` (`:309`),
  `streamingComponent`/`streamingMessage` (`:305-306`), `compactionQueuedMessages` (`:343`),
  `skillCommands: Map<string,string>` (`:318`), widget maps (`:355-356`).
- `AgentSessionEvent` (imported from `core/agent-session.ts`) — the event union driving
  `handleEvent`.
- `RpcCommand` / `RpcResponse` / `RpcSessionState` / `RpcExtensionUIRequest|Response`
  (`rpc/rpc-types.ts`).
- `Theme` class + `ThemeColor`/`ThemeBg`/`ColorValue`/`ThemeJson` (`theme/theme.ts:106-159, 322`).
- `AppKeybindings` / `AppKeybinding` / `KeybindingsManager` (`core/keybindings.ts:13-57, 340`).
- `CustomEditor.actionHandlers: Map<AppKeybinding, ()=>void>` (`custom-editor.ts:9`).

---

## 8. Gotchas & async/render pitfalls

1. **Session object is swapped, never mutated in place.** `/new`, `/resume`, fork, clone, and
   `switch_session` replace `runtimeHost.session`. Always access it via the `session` getter
   (`interactive-mode.ts:376`); both interactive and RPC modes must call `rebindSession()` to
   re-subscribe after a swap (`rpc-mode.ts:432-435`, etc.). Subscriptions captured against the old
   session will silently go dead.
2. **Mode is auto-selected by TTY.** Piping stdin (non-TTY) forces *print* mode even without `-p`
   (`main.ts:109`). Easy to trip in CI/scripts expecting interactive.
3. **Theme Proxy must be initialized first.** Any `theme.fg(...)` before `initTheme()` throws
   (`theme.ts:752`). Components constructed early must not render at construction time.
4. **JSONL framing is LF-only on purpose.** Using `readline` or splitting on Unicode line
   separators corrupts payloads containing U+2028/U+2029 in strings (`jsonl.ts`, `rpc.md`).
5. **`buildInitialMessage` mutates `parsed.messages`** by shifting the first element
   (`cli/initial-message.ts:36`); later code that reads `parsed.messages` sees the remainder.
6. **`handleEvent` lazily inits.** Because the agent can emit before `init()` finishes, the first
   line of `handleEvent` awaits `init()` (`interactive-mode.ts:2707-2709`). Removing that guard
   reintroduces a startup race.
7. **Dynamic escape-handler swapping.** `onEscape` is reassigned around compaction (`:2922-2925`),
   retry, and bash. The handlers must be restored on `agent_start`/`compaction_end`
   (`:2721-2724, 2947-2950`) or Escape's meaning gets stuck.
8. **Streaming render churn.** Every `message_update` rebuilds the streaming component content and
   calls `ui.requestRender()`; tool components are keyed by `toolCallId` to avoid duplicates
   (`pendingTools` map). Losing the key (e.g. provider re-emitting a different id) would create
   duplicate tool blocks.
9. **RPC `prompt` success semantics.** Success is emitted on *preflight*, not completion; the actual
   work streams asynchronously (`rpc-mode.ts:390-412`). Clients must use `waitForIdle`/`agent_end`,
   not the `prompt` response, to know a turn finished.
10. **RPC backpressure.** The server awaits `waitForRawStdoutBackpressure()` after responses/events
    (`rpc-mode.ts:357-359, 742`) so a slow client doesn't blow memory; a client that never reads
    stdout will stall the agent.
11. **Double-key timing.** Ctrl+C-twice (exit) and Esc-twice (tree/fork) both use a 500ms window
    (`:3313, :2441`). Worth knowing when writing tests that simulate keys.
12. **Theme watcher only watches custom themes.** Editing `dark.json`/`light.json` on disk won't hot-
    reload (`theme.ts:833`); only files under the custom-themes dir do.

---

## 9. Suggested teaching path

1. Trace one keystroke: type `/model`, Enter → `CustomEditor.handleInput` → `onSubmit` ladder
   (`:2528`) → `handleModelCommand` → `ModelSelectorComponent`.
2. Trace one agent turn: `run()` loop → `session.prompt` → events arrive at `handleEvent` →
   `message_start/update/end` + `tool_execution_*` mutate `chatContainer`.
3. Contrast the same `prompt` over RPC: client `send` → `rpc-mode` `handleCommand "prompt"` → events
   `output()` as JSON lines → client `onEvent`.
4. Show the theme Proxy + `initTheme` ordering and a custom-theme hot reload.
5. Show how a keybinding override in `keybindings.json` flows to a handler (§6).
