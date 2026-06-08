# 07 — Root Tooling, Build System & CI

Study guide for the BUILD / TOOLING / INFRASTRUCTURE layer of the `pi` monorepo:
everything outside `packages/*/src` and `packages/*/test`. Citations are `path:line`.

---

## 1. Monorepo layout & lockstep versioning

The repo is an npm workspaces monorepo. Root `package.json:1-12` declares
`"name": "pi-monorepo"`, `"private": true`, `"type": "module"`, and workspaces
covering `packages/*` plus five `coding-agent/examples/extensions/*` example
packages.

The four shipped packages (`package.json` per package):

| Package | npm name | version | role |
|---|---|---|---|
| `packages/tui` | `@earendil-works/pi-tui` | 0.79.0 | terminal UI lib, differential rendering, native key/console addons |
| `packages/ai` | `@earendil-works/pi-ai` | 0.79.0 | unified multi-provider LLM API |
| `packages/agent` | `@earendil-works/pi-agent-core` | 0.79.0 | agent runtime / transport / state |
| `packages/coding-agent` | `@earendil-works/pi-coding-agent` | 0.79.0 | the `pi` CLI (bin `pi` → `dist/cli.js`) |

**Lockstep versioning.** All four share one version; every release bumps them
together (`AGENTS.md:122`). `version:patch|minor|major` (`package.json:24-26`) run
`npm version <type> -ws --no-git-tag-version`, then `scripts/sync-versions.js`,
then `npm install --package-lock-only --ignore-scripts`.
`sync-versions.js:37-45` hard-fails if the workspace versions are not identical,
then rewrites every inter-package dependency to `^<version>`
(`sync-versions.js:51-89`). Policy: `patch` = fixes+additions, `minor` =
breaking; no major releases (`AGENTS.md:122`).

**Build order.** `package.json:15` builds sequentially, dependency-first:
`tui → ai → agent → coding-agent`. Order matters because each downstream
`tsconfig.build.json` resolves the `@earendil-works/*` path aliases to the
upstream package's emitted `dist/*.d.ts` (e.g. `packages/coding-agent/tsconfig.build.json`
maps all three deps to `../<pkg>/dist/*.d.ts`), so the upstream `dist` must exist
first.

---

## 2. Build system

### TypeScript via tsgo (native preview)
Every package builds with `tsgo -p tsconfig.build.json` (e.g.
`packages/tui/package.json:9`, `ai:65`, `agent:25`, `coding-agent:30`). `tsgo` is
the **TypeScript native (Go) preview** compiler, pinned as
`@typescript/native-preview` `7.0.0-dev.20260120.1` (`package.json:42`);
classic `typescript` `5.9.3` is also present for `tsc`-API consumers and as the
type-checker behind `tsgo --noEmit` in `check`.

`tsconfig.base.json` is the shared compiler baseline: `target ES2022`,
`module/moduleResolution Node16`, `strict`, `declaration` + `declarationMap` +
`sourceMap`, and crucially `erasableSyntaxOnly:true` (`tsconfig.base.json:7`),
`allowImportingTsExtensions:true` (18) and `rewriteRelativeImportExtensions:true`
(19). The last two let source import siblings as `./foo.ts`; the compiler
rewrites them to `.js` on emit. `erasableSyntaxOnly` enforces Node strip-only
syntax (no enums/namespaces/parameter-properties — see `AGENTS.md:20`).

Root `tsconfig.json` is **type-check only** (`noEmit:true`, `tsconfig.json:4`).
It defines path aliases pointing straight at `packages/*/src/*.ts`
(`tsconfig.json:5-21`) so the whole repo type-checks against source (used by
`tsgo --noEmit` in `check`). Per-package `tsconfig.build.json` files instead set
`outDir: ./dist`, `rootDir: ./src`, and re-point aliases at upstream `dist`
`.d.ts` (so each package compiles against its dependencies' built types).

### esbuild
Two uses, both dev/CI-only (devDep `esbuild` 0.28.0, `package.json:43`):
- the browser-smoke bundle (§3, `scripts/check-browser-smoke.mjs:10`)
- not used for the production bundle (tsgo emits per-file JS; bun does the binary bundling).

### Native TUI addon
`packages/tui/native/` ships **prebuilt** `.node` addons, no node-gyp build in
this repo. Sources: `native/darwin/src/darwin-modifiers.c` and
`native/win32/src/win32-console-mode.c`; prebuilds live under
`native/{darwin,win32}/prebuilds/<platform>/...node` and are listed in the
package `files` array (`packages/tui/package.json:13-18`). `darwin-modifiers`
reads keyboard modifier state on macOS; `win32-console-mode` toggles console
mode on Windows. They are consumed at runtime by `tui/src/native-modifiers.ts`
and `tui/src/terminal.ts`. The binary build copies the right prebuild next to
the compiled binary (`scripts/build-binaries.sh:183-196`).

### Bun single-file binary (`build-binaries.sh`)
The `pi` CLI also ships as standalone bun executables. The compiled entry is the
tsgo-emitted `dist/bun/cli.js` (from `src/bun/cli.ts`), passed to
`bun build --compile` along with `src/utils/image-resize-worker.ts` as an
explicit second entrypoint — bun only embeds workers passed explicitly
(`scripts/build-binaries.sh:133-141`; in-package single-platform variant at
`packages/coding-agent/package.json:31`).

`scripts/build-binaries.sh` (mirrors `.github/workflows/build-binaries.yml`):
- `npm ci --ignore-scripts` (`:86`), then force-installs **all** platform
  variants of `@mariozechner/clipboard` so bun can cross-compile
  (`:93-105`, `--force --ignore-scripts --package-lock=false`).
- `npm run build` (`:112`), then loops the six targets
  `darwin-{arm64,x64} linux-{x64,arm64} windows-{x64,arm64}` calling
  `bun build --compile --target=bun-$platform` (`:131-141`).
- Per platform copies runtime assets next to the binary: `photon_rs_bg.wasm`,
  theme JSON, assets, export-html, docs, examples, the matching clipboard native
  package, and the TUI prebuild (`:146-197`).
- Packs `tar.gz` (unix) / `zip` (windows) and re-extracts for local testing
  (`:202-223`). Unix archives wrap a `pi/` dir for mise compatibility (`:210`).

---

## 3. Quality gates — `npm run check`

`package.json:16`:
```
biome check --write --error-on-warnings . && check:pinned-deps && check:ts-imports
  && check:shrinkwrap && tsgo --noEmit && check:browser-smoke
```

### Biome 2.3.5 (`biome.json`)
Lint + format in one tool. Notable: tabs, `indentWidth:3`, `lineWidth:120`
(`biome.json:22-24`); `useConst:error`, `noNonNullAssertion` and
`noExplicitAny` off (`biome.json:8-16`). `files.includes` (`biome.json:27-37`)
scopes to `packages/*/src|test` + examples and excludes generated files
(`models.generated.ts`, `test-sessions.ts`). `--error-on-warnings` makes any
warning fail CI/pre-commit.

### check:pinned-deps (`scripts/check-pinned-deps.mjs`)
Walks every `package.json` (skipping `.git/dist/node_modules`) and requires
every direct external dep across `dependencies/devDependencies/optionalDependencies`
to be an **exact** semver (`:5` regex, `:53`). Internal `@earendil-works/pi-*`
deps and non-registry specifiers (workspace/file/git/http) are exempt
(`:24-30`). Handles npm aliases (`:32-38`). Why: pinning + the npm age gate
(§4) is the supply-chain posture — no floating ranges that silently pull new
code.

### check:ts-imports (`scripts/check-ts-relative-imports.mjs`)
Uses the TS compiler API to parse every non-`.d.ts` `.ts` file and bans
**relative `.js` import specifiers** (import/export/dynamic-import/`import type`)
(`:23-25`, `:47-62`). Enforces the `.ts`-extension source convention that pairs
with `rewriteRelativeImportExtensions` (§2). `update-source-imports-to-ts.sh` is
the bulk fixer (perl rewrite of `./x.js` → `./x.ts` in package `src`).

### check:shrinkwrap (`generate-coding-agent-shrinkwrap.mjs --check`) — see §4.

### tsgo --noEmit
Full-repo type check using root `tsconfig.json` against source.

### check:browser-smoke (`scripts/check-browser-smoke.mjs` + `browser-smoke-entry.ts`)
esbuild-bundles `browser-smoke-entry.ts` with `platform:"browser"`
(`check-browser-smoke.mjs:10-17`); a build failure means a browser-facing export
of `pi-ai`/`pi-agent-core` accidentally pulled in a Node-only module. The entry
exercises a wide API surface (`getModel`, `Agent`, session repo, skill/prompt
formatting, etc., `browser-smoke-entry.ts:1-60`) so the bundler sees real import
paths. Pre-commit only runs it when `packages/ai`, `packages/web-ui`, or the
root package/lockfile changed (`.husky/pre-commit:20-27`).

---

## 4. Supply-chain hardening

### `.npmrc`
Two lines (`.npmrc:1-2`): `save-exact=true` (installs pin exact versions) and
`min-release-age=2` — npm refuses to resolve dependency versions published less
than 2 days ago, blunting compromised-release / typosquat windows. AGENTS notes
releases override it with `npm_config_min_release_age=0`
(`AGENTS.md:149-152`).

### Shrinkwrap generation + lifecycle-script allowlist
`scripts/generate-coding-agent-shrinkwrap.mjs` deterministically builds
`packages/coding-agent/npm-shrinkwrap.json` from the root lockfile so the
published CLI pins its **entire transitive tree**. It walks the coding-agent
dependency graph (`:290-335`), inlines the internal workspace packages as if
published (rewriting `resolved` to registry tarball URLs, `:195-207`,
`:127-130`), and copies external lock entries verbatim (stripping `dev`/`link`,
`:79-86`). `validateShrinkwrap` (`:224-288`) rejects link entries, local
`resolved` values, and — critically — **any package with `hasInstallScript`**
unless it is in `allowedInstallScriptPackages` (`:13-16`: only
`@google/genai@1.52.0` and `protobufjs@7.5.9`, each with a justification). It
also fails if an allowlisted package disappears (`:257-261`) and requires at
least one platform-specific optional dep (`:280-283`). `--check` mode diffs the
generated content against the committed file and fails if stale (`:341-353`);
this is `check:shrinkwrap`.

### Husky pre-commit lockfile guard
`prepare: husky` installs the hook (`package.json:36`). `.husky/pre-commit`
runs `check-lockfile-commit.mjs` first, then `npm run check`, then conditionally
the browser smoke, then restages formatted files (`:39-43`).
`scripts/check-lockfile-commit.mjs` blocks any commit that stages
`package-lock.json` (`:82-84`) **unless** `PI_ALLOW_LOCKFILE_CHANGE=1|true|yes`
(`:5-6`, `:86-89`) or the only changes are workspace package metadata
(`:54-56`, `:92-95`). On block it prints a per-package added/removed/changed
diff (`:58-75`, `:97-120`) reminding the reviewer to check new lifecycle scripts
and age gates. The point: dependency tree changes are reviewed code, never an
accidental drive-by commit.

### npm-audit workflow
`.github/workflows/npm-audit.yml` runs daily (cron `:5`): `npm ci
--ignore-scripts`, `npm audit --omit=dev --audit-level=moderate`, and
`npm audit signatures --omit=dev` (registry signature verification).

---

## 5. Release pipeline

### `scripts/release.mjs <major|minor|patch|x.y.z>`
Orchestrates a release (`release.mjs:9-19`): refuses on a dirty tree
(`:149-155`); bumps via `npm run version:*` or an explicit version that must be
strictly greater (`:80-97`); rewrites each `CHANGELOG.md` `## [Unreleased]` →
`## [version] - date` (`:107-126`); regenerates artifacts
(`generate-models`, `generate-image-models`, `shrinkwrap:coding-agent`,
`:169-171`); runs `npm run check` (`:176`); commits `Release vX.Y.Z` + tags
(`:181-184`); re-adds fresh `## [Unreleased]` sections and commits them
(`:128-143`, `:193-194`); pushes `main` and the tag (`:199-201`). The tag push
triggers CI publishing — the script itself never publishes.

### `scripts/local-release.mjs`
Builds + `npm pack`s all four packages into tarballs and installs them into an
**isolated dir outside the repo** (`:107-109` guards against in-repo output) so
file-resolution can't leak workspace sources. Creates a Node install
(`npm install --omit=dev --ignore-scripts`, `:226`), an optional Bun install
(`bun install --production --ignore-scripts`, `:238`), and a bun binary release
via `build-binaries.sh --skip-install --skip-deps --skip-build` for the current
platform (`:134-154`), with `pi` shims (`:156-169`). `npm run release:local`;
smoke-test recipe in `AGENTS.md:126-145`.

### `scripts/publish.mjs [--dry-run]`
The actual publisher (`package.json:29` and the CI publish step). Asserts
lockstep versions across the four packages (`:85-88`), checks each `dist` exists
(`:46-50`), queries npm and **skips already-published** versions — idempotent
(`:58-74`, `:108-111`), then `npm publish --access public --provenance
--ignore-scripts` (`:113`). `--provenance` emits a signed provenance
attestation; `--ignore-scripts` avoids running dependency lifecycle scripts at
publish.

### CI publish — `.github/workflows/build-binaries.yml`
Triggers on `v*` tag push (or manual dispatch, `:3-16`). Top-level
`permissions: {}` (`:18`) with per-job least-privilege; all actions are SHA-pinned.
- **build** job (`contents: write`, `:23-24`): bun 1.3.10, runs
  `build-binaries.sh`, extracts the changelog section for the version
  (`:49-62`), and `gh release create`/`upload`s the six archives (`:64-88`).
- **publish-npm** job (`needs: build`, `environment: npm-publish`,
  `permissions: id-token: write`, `:90-99`): installs, builds, checks, tests,
  asserts release artifacts are committed (`git diff --exit-code`, `:132-133`),
  upgrades to npm 11.16.0, and runs `node scripts/publish.mjs` (`:135-141`).
  Uses **npm trusted publishing via GitHub Actions OIDC** — the `id-token`
  permission + `npm-publish` environment means no npm token/OTP/WebAuthn is
  needed (`AGENTS.md:156`).

### Other workflows (`.github/workflows/`)
- **ci.yml** — push/PR to main: install (`--ignore-scripts`), `build`, `check`,
  `test`; installs cairo/pango/etc. system deps + `fd`/`ripgrep` (`ci.yml:26-42`).
  Concurrency-cancels stale runs (`:9-11`).
- **pr-gate.yml / issue-gate.yml / approve-contributor.yml** — the contributor
  gate. New non-collaborator issues are auto-closed (`issue-gate.yml:97-119`)
  and new-contributor PRs auto-closed (`pr-gate.yml:114-126`) unless the author
  is in `.github/APPROVED_CONTRIBUTORS` (capability `issue` or `pr`) or a
  write+ collaborator. A maintainer commenting `lgtmi` (→`issue`) or `lgtm`
  (→`pr`) edits the approvals file and pushes it
  (`approve-contributor.yml:34-145`). The approvals file is a flat
  `<user> <capability>` list (`APPROVED_CONTRIBUTORS:7+`).
- **openclaw-gate.yml** — labels issues/PRs `possibly-openclaw-clanker` when the
  author has activity on `openclaw/openclaw` (`:92-122`).
- **npm-audit.yml** — see §4.
- `ISSUE_TEMPLATE/` — `bug.yml`, `contribution.yml`; `config.yml` disables blank
  issues and links Discord.

---

## 6. Dev / profiling / stats scripts (`scripts/`)

These read `~/.pi/agent/sessions/<encoded-cwd>/*.jsonl` transcripts (cwd encoded
by replacing `/` with `-` and wrapping in `--…--`, e.g. `cost.ts:32-36`).

- **`cost.ts`** — per-day, per-provider USD cost breakdown for a cwd over N days
  (`--dir`, `--days`), summing `message.usage.cost.{input,output,cacheRead,cacheWrite,total}`.
- **`stats.ts`** — like cost.ts but token-centric: per local-day and per-provider
  token + cost + message/session counts, defaulting to the current cwd and 7
  days (`stats.ts:114-145`).
- **`tool-stats.ts`** — aggregates tool-call counts, result sizes, error rates,
  bucketed token estimates across sessions and renders an HTML report opened in
  the browser (`tool-stats.ts:18`, uses coding-agent's `open-browser`).
- **`read-tool-stats.mjs` / `edit-tool-stats.mjs`** — focused histograms for the
  read/edit tools (top-N, model filter, `--since` auto-derived from the tool/
  extension file mtime, ASCII bar charts).
- **`session-context-stats.mjs`** — context-window usage stats, cross-referencing
  `packages/ai/src/models.generated.ts` and `~/.pi/agent/models.json`
  (`:9-12`).
- **`session-transcripts.ts`** — extracts a cwd's transcripts, splits into
  ~100k-char (~20k-token) files (`:20`), optionally spawns `pi` subagents to
  analyze patterns (`--analyze`). Imports `parseSessionEntries` from
  coding-agent source.
- **`profile-coding-agent-node.mjs`** — startup profiler for the CLI in `tui` or
  `rpc` mode (`npm run profile:tui|rpc`, `package.json:21-22`). Supports
  node/bun/auto runtimes, warmup/runs, optional CPU profiles to
  `profiles-node/`/`profiles-bun/`, isolated agent dirs, and offline env
  (`PI_OFFLINE`, `PI_SKIP_VERSION_CHECK`) (`profile-…:18-48`).

---

## 7. In-repo `.pi/` project config

`packages/coding-agent/package.json:6-8` sets `piConfig.configDir = ".pi"`, so
when running `pi` inside this repo it loads project-local config from `.pi/`:

- **`.pi/prompts/`** — slash-command prompt templates (markdown with frontmatter
  `description`/`argument-hint`): `cl.md` (audit changelogs before release),
  `is.md` (analyze GitHub issues), `pr.md` (review PRs from URLs), `sa.md`
  (update a GitHub security advisory), `wr.md` ("wrap it" — finish task with
  changelog/commit/push). These encode the maintainer workflows referenced
  throughout AGENTS.md.
- **`.pi/skills/add-llm-provider.md`** — a skill: the full checklist for adding a
  provider to `packages/ai` (types → impl → lazy registration → model gen →
  test matrix → coding-agent wiring → docs).
- **`.pi/extensions/`** — TS extensions loaded into the running agent:
  `tps.ts` (prints tokens/sec + token usage after each turn via `agent_start`/
  `agent_end` hooks), `redraws.ts` (registers `/tui` to show full-redraw count),
  `prompt-url-widget.ts` (renders a header widget with PR/issue/advisory metadata
  when one of the `pr`/`is`/`sa` prompts is active, fetching via `gh`).
- **`.pi/git/.gitignore`, `.pi/npm/.gitignore`** — each just `*` + `!.gitignore`,
  so the dirs exist (as scratch/state dirs) but their contents stay untracked.
  `.gitignore:38-39` also ignores `.pi/hf-sessions{,-backup}/`.

---

## 8. Running `pi` from source

- **`pi-test.sh`** — runs the CLI straight from TS source via `tsx` against the
  root `tsconfig.json`: `node_modules/.bin/tsx --tsconfig tsconfig.json
  packages/coding-agent/src/cli.ts "$@"` (`pi-test.sh:57`). `--no-env` unsets
  all provider API-key env vars first (`:17-55`) for clean-room testing.
  `pi-test.ps1` (PowerShell) and `pi-test.bat` (delegates to the .ps1) are the
  Windows equivalents; `.gitattributes` keeps `.ps1`/`.bat` as CRLF and `.sh` as
  LF (`.gitattributes:5-10`).
- **`test.sh`** — the canonical non-e2e test runner (`CONTRIBUTING.md:54-57`,
  `AGENTS.md:30`). It backs up and removes `~/.pi/agent/auth.json` (restoring on
  exit via `trap`, `:6-20`), sets `PI_NO_LOCAL_LLM=1`, unsets every provider/
  cloud credential env var (`:25-73`), then runs `npm test`. This guarantees the
  vitest suite never hits a real provider (the full suite would otherwise run
  paid e2e tests when endpoint/auth vars are present).
- **tmux interactive testing** recipe (drive the TUI headlessly) in
  `AGENTS.md:91-102`.
- Per-package tests: `npm run test` runs vitest (`--run`) except `tui` which uses
  `node --test test/*.test.ts` (`packages/tui/package.json:10`).
  `packages/coding-agent/vitest.config.ts` aliases the `@earendil-works/pi-*`
  specifiers back to sibling source (`:19-27`) and externalizes the photon wasm
  (`:14-16`).

---

## Quick-reference: key invariants

- One version, all four packages (lockstep); `sync-versions.js` enforces + fans out `^`.
- External deps pinned exact (`check:pinned-deps`) + 2-day npm age gate (`.npmrc`).
- Source imports use `.ts` extensions (`check:ts-imports`); tsgo rewrites to `.js`.
- Published CLI ships a fully-pinned shrinkwrap with a hard install-script allowlist.
- Lockfile commits require `PI_ALLOW_LOCKFILE_CHANGE=1`.
- Releases are tag-driven; CI publishes via OIDC trusted publishing, idempotently.
- `npm run check` is the single quality gate (biome + 4 custom checks + tsgo).
