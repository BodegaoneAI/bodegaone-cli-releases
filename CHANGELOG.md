# Changelog

All notable changes to Bodega One Code are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and the project aims to follow [Semantic Versioning](https://semver.org/spec/v2.0.0.html).
Versioned sections are cut at release (the release pipeline is tag-triggered on `v*`);
until the first tag, everything lives under **Unreleased**.

## [0.1.0] - 2026-07-06

The first public release of Bodega One Code (CLI): the terminal surface of the
Bodega One family. Local-first agentic coding with BYOK cloud models or fully
local models (Ollama / llama.cpp), a hard air-gap mode, MCP management,
parallel fleet runs with QEL-verified winner selection, scheduled QEL-gated
loops, and a data dir shared with the Bodega One desktop app and agent.

### Added
- **`bodega loops` — scheduled agentic tasks from the terminal** (gap C4). Create,
  list, run, inspect the history of, import, and delete Bodega Loops: recurring
  agent tasks that run on your machine in an isolated git worktree, gated by QEL
  verification. `bodega loops create --name nightly-docs --task "Sync inline docs"
  --every "0 2 * * *"` (cron; or `--every 30` interval minutes, `--on-comment`,
  `--on-map-stale N`); `--apply park|auto-if-pass|dry-run`. The CLI implements no
  scheduler of its own — loops live in the shared backend, so a loop created here
  is the same one the desktop app manages, and vice-versa. `run` waits for the
  verified result and derives its exit code from it (QEL-failed → 1) so a script
  can gate on the outcome. The scheduler fires while loops are enabled (Settings
  → AI Behavior → Loops).
- **Family-shared canonical data dir** (gap S1, CLI half). The three Bodega apps
  (desktop IDE, CLI, agent) historically resolved *three different* data dirs, so
  a provider key / MCP server / session set in one was invisible to the others.
  The CLI now resolves a neutral canonical location — `%LOCALAPPDATA%\BodegaOne`
  (Windows), `~/Library/Application Support/BodegaOne` (macOS),
  `${XDG_DATA_HOME:-~/.local/share}/bodegaone` (Linux) — shared with the other
  surfaces, with `BODEGA_USER_DATA_DIR` as the family override. Migration is
  strictly safe: **copy-not-move** (a legacy dir is never touched, so no data can
  be lost), auto-migrating only the unambiguous single-legacy-source case;
  when multiple populated legacy dirs exist it does **not** guess which is
  authoritative — it keeps using the app's own historical dir (zero regression)
  and drops a `.consolidation-pending.json` marker for a deliberate consolidation
  step. A hot source (a legacy app actively writing its DB) defers migration to a
  later launch. Verified live on a 3-dir machine: the CLI kept its own dir and the
  48MB desktop DB was byte-identical afterward. Agent + desktop halves land next.
- **`bodega fleet`** — run one task N ways in parallel and pick the best (gap
  C2a). `bodega fleet "<task>" --split 3` forks 2–5 isolated git worktrees
  (branch `fleet/<id>/m<n>`), runs each as an independent detached background
  run with its own backend process and its own QEL verdict, then
  `fleet attach` aggregates the results (status, exit, QEL score, diffstat per
  member) and RECOMMENDS the highest-QEL passing member — apply stays explicit
  (`fleet apply <id> --member <n> [--strategy squash|merge]`, clean-tree-only),
  matching the backend GUI's own manual-winner posture. `fleet
  status|list|discard` complete the lifecycle. The 2–5 split cap is the C2a
  concurrency gate (mirrors the backend's VRAM ceiling policy — unbounded
  concurrent heavy runs have taken machines down in live testing). Run flags
  after the prompt (`--model`, `--deny-tools`, `--qel-threshold`, …) forward to
  every member; `--cwd/--bg/--task-id/--output` are fleet-owned and rejected.
- **`bodega models`** — local runtimes and models from the terminal (gap C3). A
  keyless user on a bare machine no longer needs the desktop app to get a local
  model: `bodega models` shows runtime status (Ollama + bundled llama.cpp) plus
  hardware-fit model recommendations for THIS machine; `models install-runtime
  [--runtime ollama|llamacpp] [--flavor <gpu>]` drives the backend's existing
  checksum-pinned installers with live progress; `models pull <name>` /
  `models rm <name>` download and delete models via the backend's model hub
  (idempotent pulls — attaching to an in-flight download is not an error). The
  CLI re-implements zero install logic: platform installers, SHA256 manifests,
  path sandboxing, and the air-gap gate all stay in the shared backend (installs
  and pulls are refused under air-gap; read-only status still works). Onboarding-
  wizard integration ("set up a local model" as a first-run choice) is queued as
  the follow-up slice.
- **MCP management from the CLI** — `bodega mcp add / edit / remove / credential`
  give a CLI-only user full parity with the desktop app's MCP settings, by driving
  the backend's existing per-server CRUD routes (duplicate-name rejection,
  cmd validation, auto-connect on add, and live reconcile on edit all come from the
  shared backend for free). `add --from <file>` imports a portable
  `{"mcpServers": {...}}` JSON config (stdio entries only — remote url/SSE/HTTP
  entries are rejected loudly since the backend is stdio-only; `--url` is reserved
  and errors with the same explanation). Credentials go through the encrypted
  credential route: `credential set <server> --stdin` keeps secrets out of shell
  history; multi-var `env` blocks import their first key and warn about the rest.
- **Project-local `.bodega/mcp.json`** — a second, tool-portable declarative source
  next to the `.bodega.yml` `mcp:` block (YAML wins on any id collision, with a
  warning). Embedded credential values are never auto-stored at startup — a warning
  points at the one-time `bodega mcp add --from` import instead.
- **Headless MCP reconcile** — `bodega run` now applies the project's MCP config
  before streaming, exactly like the REPL always did. Previously the reconcile was
  REPL-only, so scripted/CI runs silently saw none of the project's MCP servers.
- The `.bodega.yml` `mcp:` block (and the reconcile diff) now supports
  `enabled_tools` / `disabled_tools` / `required` — these backend config fields were
  previously unreachable from the CLI's write path entirely.
- **`bodega self-update`** (alias `bodega update`) — a non-destructive check of the
  release channel that reports whether a newer version is published and how to get
  it. `--channel <url>` (or `$BODEGA_UPDATE_URL`) points at a release manifest;
  `--json` emits the raw status. This closes a shipped broken promise: the
  CLI-vs-backend version-skew warning already told users to "run: bodega self-update",
  but no such command existed. In-place binary replacement is deliberately deferred
  until signed releases are published and the swap can be verified per platform
  (the Windows running-exe lock is the tricky case); today the command checks and
  instructs. Dev/un-stamped builds and an unreachable channel both fail soft — a
  version check never breaks the CLI or a wrapper script.
- **Portable sessions** — `bodega session export <id> --out <file>` writes a session's
  transcript to a versioned, self-describing JSON artifact so it can be backed up,
  shared, or moved to another machine (`/resume` only reaches the local backend).
  `bodega session import <file> [--print]` validates the artifact (failing closed on
  an unknown/future version) and optionally renders it; `bodega session list` shows
  persisted sessions. Export refuses a transcript that looks like it contains secrets
  unless `--force` is passed.

### Security
- **`--deny-tools` is now honored no matter where it sits on the command line.**
  Because Go's flag parser stops at the first positional, `bodega run "<prompt>"
  --deny-tools shell` silently dropped the deny list (and folded the flags into the
  prompt text) — a headless run you believed was sandboxed had no tool restriction
  at all. `run` now permutes flags and positionals, so flags before *and* after the
  prompt are parsed. `--provider`/`--model`/`--mode` placed after the prompt were
  dropped the same way and are now honored too. (Backend companion fix: the deny
  list is now also alias-aware, so `--deny-tools shell` blocks `bash`/`run`/`git`.)

### Fixed
- **`--output stream-json` emits only the documented v1 envelopes.** The raw
  internal frame journal used to ride the same stdout, interleaving a second
  undocumented schema (in one run, 256 journal frames drowned 13 v1 envelopes)
  - a consumer filtering on `schema_version: "1"` saw almost no stream and
  everyone else got duplicate events. Raw frames remain available via
  `--trace <file>` and `bodega logs`.
- **The REPL frame always fills the terminal.** The rendered frame was shorter
  than the window, so the whole UI top-anchored in the alt-screen with dead rows
  under the input — a constant float that grew as an open `/`-command palette
  narrowed while typing. The frame is now trued up to the exact terminal height
  by construction (a drifted pane can cost a scrollback row, never float the UI).
- **You vs Bodega turns are now distinct at a glance.** The two turn labels were
  lowercase words differing only in color; the user turn is now `❯ You` with the
  body dimmed and the agent turn `◆ Bodega` in the brand accent, with a hairline
  rule between exchanges and a live typing cursor on the in-flight response.
- **`bodega fleet status` / `attach` default to the latest fleet.** A bare
  `fleet attach` errored "a fleet id is required" even though the universal flow
  is `fleet "<task>"` immediately followed by `attach`; the read-only verbs now
  target the most recently created fleet (apply/discard still require the id).
- **`bodega mcp` no longer reports a just-added server as disconnected.** Each
  invocation spawns a fresh backend that reports *live* connection state, so
  `mcp list` showed every enabled server "disconnected · 0 tools" a second after
  `mcp add` proved it connects. `list` now connects enabled servers first, then
  reports what actually happened.
- **Onboarding never offers an embedding model as the default.** The model pick
  listed the provider catalog alphabetically and the non-interactive path takes
  the first entry, so a machine whose first model was an embedder got an
  un-chattable default. Embedding models are filtered from the pick list.
- **Context-window overflow is no longer a silent failure.** When a run's prompt
  exceeds the model's resolved context window (common on local models — the
  backend caps `num_ctx` to a VRAM-safe size, so a big model on a modest GPU can
  resolve to as little as 4096), the model quietly loses the task, does no
  verifiable work, and the run exits 3 with an empty-looking result — no clue
  why. The human card now warns with the actual numbers ("the prompt (8190
  tokens) exceeded this model's context window (4096) on your hardware…") and
  suggests a smaller/larger-VRAM model; `--output json` carries a
  `contextOverflow` object. Found + fixed during in-house battle-testing of
  `bodega fleet`, where two members reproduced the failure identically.
- **`bodega models` reported a running Ollama as "not installed."** The status
  decoder used `installed`/`version` but the route emits
  `alreadyInstalled`/`pinnedVersion` — so the fields silently defaulted to
  false/empty. Now decodes the routes' real keys, with fixture-shaped regression
  tests so a future rename fails the test instead of the UI. (Battle-test find.)
- **Battle suite is hermetic against the host environment.** With
  `BODEGA_INSTALL_ROOT` (or `BODEGA_NODE`) set in the shell, the suite would drive
  the binary against the *real* backend instead of the in-repo mock — leaking a
  process and (on Windows) hanging ~60s then failing on temp-dir cleanup. The gate
  now neutralizes both for every battle test.

### Added
- **Image input** — attach images to a prompt for vision models: `@path/to/image.png`
  in the REPL composer, or repeatable `--image <path>` on `bodega run`
  (png/jpg/jpeg/gif/webp). Paths go through the same sandbox as `@file` refs. If the
  resolved model can't see images, the CLI warns you up front instead of silently
  dropping them.
- **Full-turn checkpoints** — `/undo` now reverts both the files *and* the
  conversation for the last turn (previously files only), and a new **`/rewind`** steps
  back N turns at once: `/rewind` lists available checkpoints, `/rewind <n>` rolls back
  to that point.
- **Per-session cost budget** — cap BYOK/cloud spend with `bodega run --max-cost <usd>`
  or `budget.max_cost_usd` in config. A `/budget` command shows spend against the
  ceiling in the REPL; the CLI warns at 80% and hard-stops at 100%. An over-budget
  headless run exits with a new distinct code **7**, so CI can tell "too expensive"
  apart from a QEL failure.
- **Two-phase backend readiness** — headless runs now wait (by default) for the
  backend's full toolset (MCP servers + skills) to finish loading after the health
  gate, so a run dispatched at startup doesn't execute with a partial toolset.
  `--no-wait-settle` opts out and dispatches at first ready; if the backend never
  reports settled within the timeout, the run proceeds with a note rather than failing.
- **`bodega run --bg`** — detach a headless run and get a task id back immediately,
  with `bodega attach <id>` to wait for the result, plus `logs` and `runs` to inspect
  background tasks.
- **`bodega run --mode ask|plan|act`** + **`--profile <name>`** — set the permission
  mode explicitly in headless runs, or apply a named permission profile stored in the
  backend's settings.
- **Versioned `stream-json` output** — `--output stream-json` events now carry a v1
  envelope, so scripts can pin against a stable event taxonomy.
- **Session lifecycle tracking** — the CLI now tracks session state transitions
  explicitly, so resumed, replaced, and ended sessions can't be written to after the
  fact.
- **Richer diff rendering in the REPL** — word-level inline highlighting for modified
  lines, and unchanged context collapses behind a summary line instead of filling the
  scrollback.
- **Permission-mode pill** — the REPL status bar shows the current permission mode in
  color, cycles it with Shift+Tab, and adds a prompt-cache hit-rate segment. — a GitHub webhook ingress that turns a labeled (or
  opened-with-label) issue into a verified, QEL-gated automation run. Every delivery is
  HMAC-SHA256-verified over the raw body against the shared secret
  (`--webhook-secret` or `$BODEGA_WEBHOOK_SECRET`) *before* its body is interpreted —
  an unsigned/mis-signed request is rejected `401` and never dispatches. A matching
  delivery (default trigger label `bodega`, set with `--webhook-label`) runs through the
  backend's verified-automation pipeline (isolated worktree → headless run → PR) via
  `POST /api/github/run-task`; the GitHub PAT stays server-side and never crosses the
  CLI boundary. Runs dispatch serially (the backend is single-flight) and asynchronously
  (within GitHub's ~10s delivery window). Binds to `127.0.0.1:8787` by default
  (`--webhook-addr`); front it with a tunnel/reverse-proxy for external delivery.
  Exactly one of `--acp` or `--webhook` is required.
- **`bodega run --tier fast|smart|code`** + **`/tier`** — force the routing tier for a
  turn. The CLI resolves the tier to a concrete model via the backend router
  (`llm.fast_model` / `smart_model` / `code_model`) and hard-pins it (`modelPinned=true`,
  so routing rules are skipped — an explicit tier outranks rules). `--tier` is mutually
  exclusive with `--model` and `--dual`; `/tier off` reverts to the pre-force model.
  Single-active presets (llama.cpp / LM Studio) resolve every tier to the default model
  and the CLI surfaces that verbatim. Reuses `POST /api/routing/classify`; no backend
  change.
- **Shell-fix offer in the REPL** — when a `!<cmd>` exits non-zero, a one-key
  "press y to ask the agent to fix it" appears; `y` sends the failing command +
  its output to the agent as the next prompt. Any other key dismisses it.
- **`bodega run --output-schema <file>`** — validate the run's final answer against a
  JSON Schema. On a mismatch (or a non-JSON answer) the run exits **6** — a distinct
  code so CI/jq pipelines can gate on a typed answer shape, separate from the QEL
  verdict. It's the last gate (only turns a passing run into a failure) and is skipped
  under `--dry-run`. A surrounding ```json fence is tolerated.
- **`bodega review`** + **`/review`** — AI code review of your uncommitted changes
  (working-tree diff, or branch-vs-base), printing structured Critical / High /
  Suggestions findings. `--full` / `/review full` ignores the per-project delta cache.
  Reuses the backend reviewer; honors air-gap (needs a local model when on).
- **`bodega init --analyze`** — scan the project and draft `BODEGA_PROJECT.md`
  (stack, build/lint/test commands, structure) in one command, instead of running
  the onboarding wizard. Reuses the backend ProjectAnalyzer.
- **`bodega run --fallback <csv>`** + **`/fallback`** — set the failover model chain
  (`llm.fallback_models`). On a retriable primary failure (rate-limit/5xx/timeout) the
  backend fails over to the next healthy model. `/fallback off` clears it.
- **`bodega run --dual architect=<id>,editor=<id>`** + **`/dual`** — enable the
  sequential dual-model pipeline (`agent.dual_model`): a planner model drafts, an editor
  model executes. Distinct from `/model architect …` (which writes concurrent routing
  rules). `/dual off` disables it.
- **`@file` line-ranges** — attach a precise slice of a file: `@path/file.go:10-40`
  (also `:N` for one line, `:N-` to EOF, `:-M` from the top). Strictly additive — a ref
  with no range reads the whole file as before; the path sandbox is unchanged.
- **`.gitignore` / `.bodegaignore`-aware `@`-picker** — the file autocomplete now reads
  the project-root `.gitignore` (and a `.bodegaignore` supplement) and prunes those
  paths, with glob support (`*.snap`, `gen/*`). Build output and generated files no
  longer clutter the picker.
- **`bodega run --effort`** — set the reasoning tier (off|low|medium|high|xhigh|max) in
  headless/CI runs (previously REPL-only). Out-of-range tiers clamp per model.
- **`bodega --bare`** — start the REPL with no project customizations (hooks, custom
  commands, MCP reconcile) for a clean, reproducible session and config bisecting.
- Interactive REPL (Bubble Tea): streaming, tool/plan approvals, unified diffs, the QEL
  score card, model + dual-model picker, slash commands, `@`-file / `!`-shell composer.
- Headless `bodega run` with the QEL-derived exit-code contract (0–5), `--output`
  human|json|quiet|stream-json, `--trace`, `--deny-tools`, `--qel-threshold`,
  `--dry-run`, `--answers`, `--air-gap`, and stdin-piped prompts.
- `bodega serve --acp` — ACP stdio relay for editor integration (`--check` to diagnose).
- Slash commands: `/model` (+ dual-model), `/provider`, `/connect` (masked in-session
  key entry), `/effort`, `/mode`, `/goal`, `/mcp`, `/loops`, `/diff`, `/undo`, `/resume`,
  `/proof`, `/repair`, `/tokens`, `/stats`, `/compact`, `/new`, `/clear`, `/help`.
- `bodega doctor` (environment diagnostics, `--json`), `init` onboarding wizard,
  `config get/set/list` with per-key provenance, `mcp` enable/disable.
- Lifecycle hooks, custom project commands (`.bodega/commands/*.md`), MCP reconcile from
  `.bodega.yml`, the `/repair` lint/test loop, and git-checkpoint `/undo`.
- Air-gap enforcement: outbound network paths fail closed when air-gap is enabled.
- Packaging: cross-platform builds, bundled Node runtime, install scripts
  (curl/irm + Scoop/Homebrew/npm), and a tag-triggered release pipeline.

### Changed
- **Faster interactive rendering** — the REPL scrollback re-renders incrementally from
  a cached prefix instead of rebuilding the whole transcript every frame, which keeps
  large sessions responsive.
- **Faster startup** — independent post-onboarding boot round-trips now run in
  parallel instead of sequentially.
- **Background file indexing** — the `@`-autocomplete file index is built off the UI
  thread at startup, so the first `@` no longer stalls the composer in large projects.
- The REPL usage text now documents `--repair` and `--effort`.
- The reasoning-tier valid set + validation are now a single source of truth in
  `internal/core/model` (shared by `bodega run --effort` and the REPL `/effort`).
- A pre-release battle-test suite now gates the first release (internal; no user-facing
  behavior change).

### Fixed
- `/undo` after switching sessions (`/resume` or `/new`) no longer restores files from
  the previous session — checkpoints are invalidated on any state-replacing transition.
- The REPL renders on the alternate screen buffer, so quitting restores your terminal
  scrollback instead of leaving the session transcript behind.
- Stacked overlays (pickers, approvals) no longer collapse the scrollback when the
  terminal is short — chrome height is clamped.
- `bodega review` sends the resolved project path instead of the raw `--cwd` flag
  value.

### Notes
- The release mirror is `BodegaoneAI/bodegaone-releases`.
