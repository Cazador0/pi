# `packages/tui` — Study Guide

`@earendil-works/pi-tui` (v0.79.0) is a minimal terminal-UI framework with **differential rendering** and **synchronized output** for flicker-free interactive CLI apps. It is pure TypeScript (ESM, Node ≥ 22.19) with two tiny optional native add-ons (macOS modifier polling, Windows console-mode). Runtime deps: `get-east-asian-width` (cell-width) and `marked` (Markdown). `@xterm/headless` and `chalk` are dev-only (tests/demos).

The mental model: the app is a **tree of `Component`s** under a root `TUI`. Each render, every component returns an **array of plain strings (one per terminal row)**, the `TUI` composites overlays on top, diffs the new line array against the previous one, and emits the **minimal cursor moves + line rewrites** needed to update the physical screen. There is no virtual DOM and no cell grid; the diff unit is a whole line (string equality).

---

## 1. Purpose & Public API (`src/index.ts`)

`src/index.ts` (107 lines) is a pure re-export barrel. Surface area by area:

- **Core engine** (`tui.ts`): `TUI`, `Container`, `Component`, `Focusable`, `isFocusable`, `CURSOR_MARKER`, overlay types (`OverlayAnchor`, `OverlayMargin`, `OverlayOptions`, `OverlayHandle`, `OverlayUnfocusOptions`, `SizeValue`).
- **Terminal** (`terminal.ts`): `Terminal` interface, `ProcessTerminal`.
- **Keys** (`keys.ts`): `Key`, `matchesKey`, `parseKey`, `decodeKittyPrintable`, `isKeyRelease`, `isKeyRepeat`, `isKittyProtocolActive`, `setKittyProtocolActive`, `KeyId`, `KeyEventType`.
- **Keybindings** (`keybindings.ts`): `KeybindingsManager`, `getKeybindings`, `setKeybindings`, `TUI_KEYBINDINGS`, plus types.
- **Input buffering** (`stdin-buffer.ts`): `StdinBuffer`.
- **Components** (`components/*`): `Box`, `Container`, `Text`, `TruncatedText`, `Input`, `Editor`, `Markdown`, `Loader`, `CancellableLoader`, `SelectList`, `SettingsList`, `Spacer`, `Image`, plus their theme/option types and `EditorComponent`.
- **Images** (`terminal-image.ts`): protocol detection + Kitty/iTerm2 encoders (`renderImage`, `encodeKitty`, `encodeITerm2`, `detectCapabilities`, dimension parsers, `hyperlink`, …).
- **Utilities** (`utils.ts`): `visibleWidth`, `truncateToWidth`, `wrapTextWithAnsi`.
- **Autocomplete** (`autocomplete.ts`): `CombinedAutocompleteProvider` + provider/item types.
- **Fuzzy** (`fuzzy.ts`): `fuzzyMatch`, `fuzzyFilter`.

### The `Component` contract (`tui.ts:39`)
```ts
interface Component {
  render(width: number): string[];     // one string per row; MUST be <= width visible cols
  handleInput?(data: string): void;     // raw terminal bytes, only when focused
  wantsKeyRelease?: boolean;            // opt in to Kitty release events (default filtered out)
  invalidate(): void;                    // drop cached render state
}
```
Hard rule (enforced at render time, `tui.ts:1356`): **any line wider than the terminal width throws** (after dumping a crash log to `~/.pi/agent/pi-crash.log` and restoring the terminal). Custom components must use `truncateToWidth`/`visibleWidth`. The TUI appends a full SGR + OSC-8 reset to every non-image line, so styles never bleed across rows (`tui.ts:993`, `applyLineResets` at `:995`).

---

## 2. Core rendering model

### Terminal abstraction (`terminal.ts:52`)
`Terminal` is a thin I/O port: `start(onInput,onResize)`, `stop`, `drainInput`, `write`, `columns`/`rows` getters, `kittyProtocolActive`, `moveBy`, cursor show/hide, clear ops, `setTitle`, `setProgress`. `ProcessTerminal` (`terminal.ts:99`) is the real implementation over `process.stdin/stdout`; `VirtualTerminal` (in `test/virtual-terminal.ts`, built on `@xterm/headless`) is the test double. This indirection is what makes the renderer unit-testable.

### Data structures
The renderer's entire state is line-array + cursor bookkeeping (`tui.ts:266`–`292`):
- `previousLines: string[]` — last frame, the diff baseline.
- `previousWidth`/`previousHeight` — for detecting resize.
- `cursorRow` (logical end-of-content) vs `hardwareCursorRow` (actual terminal row, may differ due to IME positioning).
- `maxLinesRendered` — high-water mark of the terminal working area.
- `previousViewportTop` — top of the visible viewport when content exceeds the screen (for resize-aware moves).
- `previousKittyImageIds: Set<number>` — Kitty image ids drawn last frame, so they can be deleted before redraw.

### The render loop (`requestRender` → `scheduleRender` → `doRender`)
- `requestRender(force?)` (`tui.ts:655`) coalesces renders. Normal path sets a flag and schedules on `process.nextTick`; `scheduleRender` (`:684`) throttles to **≥16 ms between frames** (`MIN_RENDER_INTERVAL_MS`, ~60 fps) via a `setTimeout`. `force=true` wipes all cached state (forces a full clear next frame) and renders immediately on next tick.
- `doRender` (`tui.ts:1127`) is the heart. Steps: render component tree → composite overlays → extract the fake-cursor marker → apply per-line resets → choose a flush strategy → write one batched buffer.

### Differential strategies (`doRender`)
All writes are wrapped in **synchronized output** `\x1b[?2026h … \x1b[?2026l` so the terminal paints atomically (no flicker/tearing). Strategy selection:

1. **First render** (`previousLines.length === 0`, no resize, `:1196`): emit all lines with no clear (`fullRender(false)`), assuming a clean screen — preserves scrollback.
2. **Width changed** (`:1203`): wrapping changes everywhere → `fullRender(true)` (clear screen + scrollback `\x1b[2J\x1b[H\x1b[3J`). Width change always invalidates.
3. **Height changed** (`:1212`): full redraw to realign the viewport — **except under Termux** (`isTermuxSession`, `:133`), where the soft keyboard toggles height constantly and a full redraw would replay history each time.
4. **Clear-on-shrink** (`:1221`, opt-in via `setClearOnShrink`/`PI_CLEAR_ON_SHRINK=1`, default OFF): when content shrank below `maxLinesRendered` and no overlays, full redraw to wipe stale rows.
5. **Normal differential update** (`:1228`+): scan for first/last changed line (string `!==`), then move the cursor to the first change, `\x1b[2K` (clear line) + rewrite only `firstChanged..lastChanged`, and clear any now-deleted trailing lines. Single-line changes (e.g. spinner) touch only that row. Special cases: appended lines (`:1242`), all-changes-in-deleted-region (`:1263`), first change above the viewport → fallback to full redraw (`:1313`), and content scrolled below the viewport bottom → emit `\r\n` to scroll then continue (`:1325`).

`computeLineDiff` (`tui.ts:1137`) converts a target *buffer* row into a relative cursor move using `hardwareCursorRow`, `prevViewportTop`, and the new `viewportTop` — this is what keeps cursor math correct when content is taller than the screen.

### Cursor handling (two cursors)
- **Fake cursor**: components draw their own cursor with reverse video (`\x1b[7m…\x1b[27m`). This is the visible cursor.
- **Hardware cursor**: kept hidden by default. Focusable components emit `CURSOR_MARKER` (`tui.ts:90`, the zero-width APC `\x1b_pi:c\x07`) at the cursor spot. `extractCursorPosition` (`:1107`) scans only the visible viewport, computes the visual column via `visibleWidth(beforeMarker)`, strips the marker, and `positionHardwareCursor` (`:1463`) moves the real cursor there — for IME candidate-window placement. The hardware cursor is shown only when `showHardwareCursor` is enabled (constructor arg, `setShowHardwareCursor`, or `PI_HARDWARE_CURSOR=1`).

### ANSI handling (`utils.ts`)
This file is the library's hardest engineering. Key pieces:
- `visibleWidth` (`utils.ts:213`): width in terminal cells, ignoring ANSI/OSC/APC; fast ASCII path, then a 512-entry LRU cache; uses `Intl.Segmenter` graphemes + `eastAsianWidth`; tabs = 3 cols; emoji = 2 (with a `couldBeEmoji` pre-filter to skip the expensive `\p{RGI_Emoji}` regex); regional-indicators forced to width 2 to avoid wrap drift (`:189`).
- `extractAnsiCode` (`:287`): one-token parser for CSI (`ESC[…m/G/K/H/J`), OSC (`ESC]…BEL/ST`, hyperlinks), and APC (`ESC_…`, the cursor marker).
- `AnsiCodeTracker` (`:366`): tracks active SGR attributes + OSC-8 hyperlink so styles can be **re-emitted at the start of each wrapped line** and closed at line end (`getLineEndReset` closes underline + reopens hyperlinks; OSC-8 terminator BEL-vs-ST is preserved because some terminals only make BEL links clickable).
- `wrapTextWithAnsi` (`:663`) / `wrapSingleLine` / `breakLongWord`: word-wrap preserving styles across breaks; force-breaks words longer than width at grapheme granularity.
- `sliceByColumn`/`sliceWithWidth` (`:1026`) and `extractSegments` (`:1086`): column-range extraction used by overlay compositing, with a pooled `AnsiCodeTracker` so the "after" segment inherits styling from before the overlay.
- `normalizeTerminalOutput` (`:279`): decomposes precomposed Thai/Lao AM vowels (same width) to avoid stale-cell repaint artifacts.

### Overlays (`tui.ts:459`–`991`)
`showOverlay(component, options?)` pushes an `OverlayStackEntry` (`:203`: component, options, `preFocus`, `hidden`, `focusOrder`) and returns an `OverlayHandle` (`hide/setHidden/isHidden/focus/unfocus/isFocused`). `compositeOverlays` (`:932`) pre-renders each visible overlay, computes layout, pads the base content up to `max(content, termHeight, minLinesNeeded)`, and splices each overlay line into the base with `compositeLineAt` (`:1049`) — which does a single-pass before/after extraction, pads, and **always re-verifies width** to avoid the width-overflow crash. `resolveOverlayLayout` (`:797`) resolves sizing (abs/percent/minWidth/maxHeight), anchor positions (9 anchors), percent and absolute row/col, offsets, and margin clamping. Overlay **focus restoration** is a small state machine (`OverlayFocusRestoreState`, `:211`) handling "eligible/blocked" so that temporary UI releasing focus correctly returns it to a still-visible capturing overlay or an explicit target.

---

## 3. Input handling

### Pipeline
`ProcessTerminal.start` (`terminal.ts:134`) enables raw mode, sets UTF-8, enables **bracketed paste** (`\x1b[?2004h`), forces a `SIGWINCH` to refresh dimensions after suspend/resume, enables Windows VT input (native helper), and negotiates the keyboard protocol. Raw stdin data is piped into a `StdinBuffer`, whose split sequences are forwarded to `TUI.handleInput`.

### `StdinBuffer` (`stdin-buffer.ts`) — sequence framing
stdin chunks can split an escape sequence across events. `StdinBuffer` (`:274`) accumulates bytes and emits **one complete sequence per `data` event** so `matchesKey`/`isKeyRelease` see atomic events. `isCompleteSequence` (`:29`) frames CSI / OSC / DCS / APC / SS3 / meta sequences, with special handling for old-style mouse (`ESC[M`+3 bytes) and SGR mouse (`ESC[<…M/m`). A 10 ms timeout flushes incomplete buffers. Bracketed-paste content is captured between `\x1b[200~`/`\x1b[201~` and re-emitted via a separate `paste` event (`:195`), which `ProcessTerminal` re-wraps with the markers for the editor. A subtle dedup (`emitDataSequence`, `:389`) drops a raw printable byte that duplicates a just-seen Kitty CSI-u codepoint (some terminals send both). `extractCompleteSequences` (`:192`) also splits WezTerm's `\x1b\x1b[…u` (ESC press + CSI-u release concatenated).

### `keys.ts` — key parsing (1400 lines)
Supports three encodings simultaneously:
1. **Kitty keyboard protocol** CSI-u: `\x1b[<cp>:<shifted>:<base>;<mod>:<event>u` (`parseKittySequence`, `:587`); flags requested = 7 (disambiguate + report-events + alternate-keys). Event types press/repeat/release (`:505`).
2. **xterm modifyOtherKeys** `CSI 27;mod;cp~` (`parseModifyOtherKeysSequence`, `:696`) — fallback when Kitty is unavailable.
3. **Legacy sequences** — big tables `LEGACY_KEY_SEQUENCES` / `LEGACY_SHIFT_SEQUENCES` / `LEGACY_CTRL_SEQUENCES` / `LEGACY_SEQUENCE_KEY_IDS` (`:368`+).

`matchesKey(data, keyId)` (`:820`) is the public matcher driven by a typed `KeyId` union (`Key.ctrl("c")`, `"shift+tab"`, etc.). It is mode-aware: many branches behave differently when `_kittyProtocolActive` (e.g. `\x1b\r` and `\n` mean shift+enter under Kitty, alt+enter in legacy). Cross-layout handling: matches against `baseLayoutKey` only for non-Latin/non-symbol keys (`:686`) so Cyrillic Ctrl+С matches Ctrl+c without breaking Dvorak/Colemak remaps. `parseKey` (`:1251`) is the inverse (bytes → key-id string). `decodeKittyPrintable`/`decodePrintableKey` (`:1349`,`:1398`) extract a literal character from a CSI-u/modifyOtherKeys sequence (plain or shift only) — used by inputs so disambiguated terminals still type normally. `isKeyRelease`/`isKeyRepeat` (`:527`,`:557`) cheaply sniff `:3`/`:2` event suffixes (guarding against paste content). The global `_kittyProtocolActive` flag is set by `ProcessTerminal`.

### Keyboard-protocol negotiation (`terminal.ts:220`)
`queryAndEnableKittyProtocol` writes `\x1b[>7u\x1b[?u\x1b[c` (request flags, query, then a DA sentinel). Responses are parsed by `parseKeyboardProtocolNegotiationSequence` (`:23`); a non-zero flags reply enables Kitty, a `0` reply or a DA-first reply enables `modifyOtherKeys` (`\x1b[>4;2m`). This is **response-driven, not timeout-driven** (a 0.79.0 fix), and a small buffer reassembles split responses (`:252`). On `stop`/`drainInput` it pops the protocol (`\x1b[<u`), disables modifyOtherKeys, disables bracketed paste, and pauses stdin so a buffered Ctrl+D can't leak to the parent shell over SSH.

### Routing & listeners (`TUI.handleInput`, `tui.ts:704`)
Order: (1) global `inputListeners` (can `consume` or rewrite `data`); (2) consume cell-size responses `\x1b[6;h;wt` (`:773`); (3) global debug key `shift+ctrl+d` → `onDebug`; (4) overlay focus reconciliation; (5) forward to `focusedComponent.handleInput` — but **key-release events are dropped unless the component sets `wantsKeyRelease`** (`:765`). After delivering input, a render is requested.

### Mouse / paste / focus
- **Mouse**: there is no mouse-event API; `StdinBuffer` only *frames* SGR/legacy mouse sequences so they don't corrupt key parsing.
- **Paste**: bracketed-paste is framed by `StdinBuffer` and re-wrapped; `Input` and `Editor` each buffer between markers in `handleInput`. The `Editor` turns **large pastes (>10 lines or >1000 chars) into a `[paste #N +K lines]` marker** (`editor.ts:1131`), storing the real content in a `pastes` map and expanding it on submit.
- **Focus**: `setFocus`/`setFocusInternal` (`tui.ts:332`) track a single `focusedComponent`; `Focusable` components get `.focused` toggled so they emit `CURSOR_MARKER`.

---

## 4. Component model (`src/components/`)

Components compose by **vertical stacking**: `Container.render` (`tui.ts:250`) concatenates child line-arrays; the root `TUI extends Container`. There is no horizontal layout engine — horizontal arrangement is done by emitting pre-spaced strings. Layout flows top-down: parent passes a width, child returns lines ≤ that width.

| File | Class | Role |
|---|---|---|
| `spacer.ts` (28) | `Spacer` | N blank lines for vertical spacing. |
| `text.ts` (106) | `Text` | Multi-line word-wrapped text with X/Y padding and optional background fn; caches by `(text,width,bg)`. Skips render when empty. |
| `truncated-text.ts` (65) | `TruncatedText` | Single line truncated to width (first line only); for status bars/headers. |
| `box.ts` (137) | `Box` | Container applying padding + background to all children; samples `bgFn("test")` to detect background changes for cache invalidation. |
| `input.ts` (447) | `Input` (`Component`+`Focusable`) | Single-line input with **horizontal scrolling** (`render`, `:378`), grapheme-aware editing, kill-ring (`Ctrl+U/K/W`, yank/yank-pop), undo stack, word nav, bracketed paste (newlines stripped). Emits fake reverse-video cursor + `CURSOR_MARKER`. |
| `editor.ts` (2243) | `Editor` (`Component`+`Focusable`) | The flagship multi-line editor (see below). |
| `markdown.ts` (814) | `Markdown` | Renders Markdown via `marked` with a `MarkdownTheme` of styling fns (heading/bold/code/quote/link/…), optional syntax highlight hook, OSC-8 hyperlinks when supported, padding, render cache. Custom `StrictStrikethroughTokenizer`. |
| `loader.ts` (92) | `Loader` (extends `Text`) | Animated braille spinner; `setInterval` drives frames and calls `tui.requestRender()`; configurable frames/interval. |
| `cancellable-loader.ts` (40) | `CancellableLoader` (extends `Loader`) | Adds an `AbortSignal` + `onAbort`, aborted on `tui.select.cancel` (Escape). |
| `select-list.ts` (229) | `SelectList` | Keyboard-navigable list (wrapping up/down, Enter/Escape), scroll window of `maxVisible`, two-column label+description layout, themed. Filter via `setFilter` (prefix). |
| `settings-list.ts` (250) | `SettingsList` | Settings panel: value cycling (Enter/Space), nested submenus (delegates input to submenu component), optional fuzzy search via an embedded `Input`. |
| `image.ts` (126) | `Image` | Inline image; emits Kitty/iTerm2 escape on capable terminals, otherwise a themed text placeholder. Reserves `rows` lines so the differ accounts for image height. |

### `Editor` deep dive (`editor.ts`)
State is `{ lines: string[], cursorLine, cursorCol }` (`:194`). Notable mechanics:
- **Word-wrap layout**: `wordWrapLine` (`:107`) produces `TextChunk`s (text + start/end indices); `layoutText` (`:829`) turns logical lines into `LayoutLine`s carrying cursor position; `buildVisualLineMap` (`:1638`) maps visual rows ↔ logical positions, the basis for up/down movement.
- **Sticky column** for vertical motion: `computeVerticalMoveColumn` (`:1383`) implements a 7-case decision table (documented inline) using `preferredVisualCol` so the cursor remembers its target column across short lines.
- **Paste markers as atomic segments**: `segmentWithMarkers` (`:32`) merges `[paste #N …]` into single grapheme units so cursor/delete/wrap treat them atomically; `snappedFromCursorCol` (`:285`) preserves intent when the cursor snaps to a marker boundary.
- **Vertical scrolling**: `render` (`:415`) keeps the cursor visible within `max(5, 30% of terminal rows)` and draws `─── ↑ N more`/`↓ N more` scroll borders.
- **History**: up/down at the first/last visual line browse a 100-entry prompt history (`navigateHistory`, `:375`).
- **Autocomplete**: debounced, abortable, async request pipeline (`requestAutocomplete`/`startAutocompleteRequest`/`runAutocompleteRequest`, `:2069`+) with snapshot-based staleness checks; renders an embedded `SelectList`; re-queries after cursor movement (0.79.0 fix). Slash-commands auto-trigger on `/` at line start; `@`/`#` and Tab trigger file completion.
- **Kill ring, undo, char-jump** (`Ctrl+]` forward / `Ctrl+Alt+]` backward), shift/ctrl/alt+Enter newline handling, and a `\`+Enter→newline workaround for terminals lacking Shift+Enter (`shouldSubmitOnBackslashEnter`, `:1183`).

`EditorComponent` (`editor-component.ts`) is the interface a custom editor must satisfy to be drop-in (vim mode etc.).

### Shared helpers
- `kill-ring.ts` — Emacs kill/yank ring with accumulate/prepend semantics.
- `undo-stack.ts` — generic `structuredClone`-on-push snapshot stack.
- `word-navigation.ts` — `findWordBackward`/`findWordForward` with optional custom segmenter + atomic-segment predicate (for paste markers).
- `fuzzy.ts` — subsequence fuzzy matcher (lower score = better; rewards consecutive + word-boundary matches) + `fuzzyFilter`.

---

## 5. Native add-ons (`native/`)

No build step is checked in (no `.gyp`/Makefile); **prebuilt `.node` binaries are committed** under `native/<platform>/prebuilds/<platform>-<arch>/` and shipped via `package.json` `files`. Both modules are hand-written N-API in C that resolve `napi_*` symbols dynamically with `dlsym`(darwin)/`GetProcAddress`(win32) — so they don't link against a specific Node ABI and degrade gracefully.

### macOS — `darwin-modifiers.node` (`native/darwin/src/darwin-modifiers.c`, 70 lines)
Exports `isModifierPressed(name)` returning a bool. It reads live modifier state via CoreGraphics `CGEventSourceFlagsState(kCGEventSourceStateCombinedSessionState)` and masks against shift/command/control/option (`modifier_mask_for_name`, `:23`). **Why**: Apple Terminal cannot encode Shift+Enter; `ProcessTerminal.normalizeAppleTerminalInput` (`terminal.ts:44`) needs to know if Shift is physically held when a bare `\r` arrives, and queries this helper through `native-modifiers.ts`.

### Windows — `win32-console-mode.node` (`native/win32/src/win32-console-mode.c`, 53 lines)
Exports `enableVirtualTerminalInput()`. It OR-s `ENABLE_VIRTUAL_TERMINAL_INPUT (0x0200)` into the stdin console mode (`SetConsoleMode`, `:33`). **Why**: without it, libuv's `ReadConsoleInputW` discards modifier info and Shift+Tab arrives as plain `\t`; with it, the console emits VT sequences (`\x1b[Z`, etc.). Must run **after** `setRawMode(true)`, which resets console flags (`terminal.ts:158`).

### Loading & JS fallback
- `native-modifiers.ts` (`loadNativeModifiersHelper`, `:21`): darwin + x64/arm64 only; tries three `require` candidate paths (dist-relative, module-relative, next-to-executable for packaged binaries); caches the result. `isNativeModifierPressed` (`:51`) **returns `false` if the module is missing or throws** — so Shift detection silently degrades.
- `enableWindowsVTInput` (`terminal.ts:338`): same three-candidate `require` dance; on failure, Shift+Tab simply isn't distinguishable from Tab. No crash.

So absent prebuilds, the library runs fully on the JS path; only Apple-Terminal Shift+Enter and legacy-Windows Shift+Tab fidelity are lost.

---

## 6. Theming / styling primitives

There is **no central theme system**; theming is *dependency-injected per component* as objects of `(text) => string` functions (typically `chalk`). Examples: `EditorTheme` (`editor.ts:206`), `MarkdownTheme`/`DefaultTextStyle` (`markdown.ts:34`,`:53`), `SelectListTheme` (`select-list.ts:18`), `SettingsListTheme` (`settings-list.ts:22`), `ImageTheme` (`image.ts:12`). `Box`/`Text` take a `bgFn`. The library stays color-library-agnostic; the framework's only styling responsibility is **preserving and resetting ANSI codes** correctly across wraps, lines, and overlay composites (all in `utils.ts`).

**Capabilities** are detected (not configured): `detectCapabilities` (`terminal-image.ts:65`) inspects env (`TERM`, `TERM_PROGRAM`, `KITTY_WINDOW_ID`, `WEZTERM_PANE`, `ITERM_SESSION_ID`, `TMUX`, …) to decide `images` (kitty/iterm2/null), `trueColor`, and `hyperlinks` (with a tmux `client_termfeatures` probe for OSC-8 forwarding). Cached; overridable in tests via `setCapabilities`/`resetCapabilitiesCache`.

**Keybindings** are the one global registry: `KeybindingsManager` (`keybindings.ts:155`) maps semantic ids (`tui.editor.cursorLeft`, `tui.select.confirm`, …) to one-or-more `KeyId`s, supports user overrides + conflict detection, and is reachable via `getKeybindings()`/`setKeybindings()`. Components call `kb.matches(data, "tui.…")` rather than hard-coding keys; the `Keybindings` interface is declaration-mergeable so downstream packages can add ids.

---

## 7. Key data structures & types (file:line)

- `Component` interface — `tui.ts:39`
- `Focusable` + `CURSOR_MARKER` — `tui.ts:74`, `tui.ts:90`
- `Container` (base of `TUI`) — `tui.ts:226`
- `TUI` class + render state fields — `tui.ts:265`–`292`
- `OverlayOptions` / `OverlayHandle` / `OverlayStackEntry` — `tui.ts:141`, `:188`, `:203`
- `OverlayFocusRestoreState` machine — `tui.ts:211`–`221`
- `Terminal` interface — `terminal.ts:52`; `ProcessTerminal` — `terminal.ts:99`
- `KeyId` union + `Key` helper — `keys.ts:152`, `:163`; `ParsedKittySequence` — `keys.ts:507`
- `StdinBuffer` — `stdin-buffer.ts:274`; sequence framing — `stdin-buffer.ts:29`
- `AnsiCodeTracker` — `utils.ts:366`; `visibleWidth` — `utils.ts:213`; `truncateToWidth` — `utils.ts:884`
- `EditorState` / `LayoutLine` / `TextChunk` — `editor.ts:194`, `:200`, `:90`
- `KeybindingsManager` / `TUI_KEYBINDINGS` — `keybindings.ts:155`, `:54`
- `TerminalCapabilities` / `ImageProtocol` — `terminal-image.ts:5`, `:3`
- `AutocompleteProvider` / `AutocompleteSuggestions` — `autocomplete.ts:241`, `:236`

---

## 8. Gotchas, performance, platform differences

**Gotchas**
- `render()` lines exceeding terminal width **crash** (deliberate, `tui.ts:1356`); always truncate. Overlay compositing has a final width re-check because width tracking can drift (`tui.ts:1085`).
- The diff is **whole-line string equality** — change one character and the whole line is rewritten (fine; lines are short). Conversely, a component that returns the *same* strings won't repaint even if it "thinks" it changed.
- Styles do **not** carry across lines (TUI resets each line); multi-line styled output must reapply styles per line or use `wrapTextWithAnsi`.
- `invalidate()` is required on `Component`; many components cache `(text,width)` and won't re-render on theme change unless invalidated.
- Key matching is **mode-dependent**: the same bytes (`\x1b\r`, `\n`) mean different keys under Kitty vs legacy. Test with both `_kittyProtocolActive` states.
- Key-release events are silently dropped unless a component opts in with `wantsKeyRelease`.
- `Input` strips newlines from pastes; `Editor` keeps them and may collapse large pastes into a marker — `getExpandedText()`/submit expands them.

**Performance**
- Render coalescing + ≥16 ms throttle (`tui.ts:279`,`:689`); synchronized output avoids flicker.
- Differential rendering only rewrites changed lines (spinner animation = one line).
- `visibleWidth` has an ASCII fast-path + 512-entry LRU cache (`utils.ts:45`); emoji width uses a cheap pre-filter before the costly Unicode regex.
- Overlay compositing avoids the historical-high-water-mark inflation that pushed content into scrollback (`tui.ts:964`).
- Kitty images are explicitly deleted before redraw of their region to prevent ghosting (`deleteChangedKittyImages`, `tui.ts:1034`).

**Platform differences**
- **Termux**: height changes don't force full redraw (soft-keyboard toggles).
- **Windows Terminal**: raw `0x08` is Ctrl+Backspace (heuristic in `matchesRawBackspace`, `keys.ts:730`); needs the native VT-input helper for Shift+Tab.
- **Apple Terminal**: Shift+Enter synthesized via the darwin native modifier helper.
- **tmux/screen**: images disabled; OSC-8 hyperlinks only when forwarding is confirmed.
- **SSH**: `drainInput`/`stop` pop the Kitty protocol and pause stdin to stop key-release/Ctrl+D leaking to the parent shell.
- Image protocol auto-selected per terminal (Kitty graphics for Kitty/Ghostty/WezTerm; iTerm2 inline for iTerm; text fallback otherwise).

**Debug hooks**: `PI_TUI_WRITE_LOG` (capture raw ANSI stream), `PI_TUI_DEBUG=1` and `PI_DEBUG_REDRAW=1` (render diagnostics), `PI_HARDWARE_CURSOR=1`, `PI_CLEAR_ON_SHRINK=1`. The `test/` dir (e.g. `tui-render.test.ts`, `tui-shrink.test.ts`, `keys.test.ts`, `stdin-buffer.test.ts`, `overlay-*.test.ts`, `wrap-ansi.test.ts`) drives the `VirtualTerminal` to assert exact screen output and is the best executable spec.
