# Changelog

All notable changes to Bodega One Code are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and the project aims to follow [Semantic Versioning](https://semver.org/spec/v2.0.0.html).
Versioned sections are cut at release (the release pipeline is tag-triggered on `v*`);
until the first tag, everything lives under **Unreleased**.

## [0.2.10] - 2026-09-11

### Changed

- **Bundles engine `v1.0.0-beta.42.1`** (`BACKEND_REF` in `.github/workflows/release.yml`, resolved with
  `git ls-remote` and hard-checked at build). The engine is beta.42 with its macOS release pipeline
  repaired; the CLI itself is unchanged from 0.2.9.

## [0.2.9] - 2026-09-11

### Changed

- **Bundles engine `v1.0.0-beta.42`** (`BACKEND_REF` in `.github/workflows/release.yml`, resolved with
  `git ls-remote` and hard-checked at build). Bring Your Own Harness, the `ask_user` inline question,
  Bodega as an MCP server, MCP OAuth, plugin update-with-diff and export, and GPT-5/6 on OpenAI's
  Responses API all ride on this engine.

### Added

- **The agent can ask you a question without stopping.** When the engine's new `ask_user` tool fires
  mid-turn, a question card appears beside the composer while the agent keeps working. Answer it
  there (Enter, or pick an option), or press Esc to keep typing your next prompt and come back with
  Ctrl-Y. If the turn finishes before you answer, your answer is sent as the next message instead of
  being lost. The pre-run interview is unchanged: that one still waits for you.
- **A live diff pane.** `/diff` now opens a panel beside the transcript that shows your
  uncommitted changes — tracked edits against HEAD and untracked files — and keeps it current
  as the agent works: it re-reads git when a tool finishes and at the end of every turn,
  including turns run on an external harness, and after `/undo`. Scroll it with the mouse
  wheel or Shift+PgUp/PgDn, click a file in its list to jump there, `/diff` again (or
  `/diff close`) to put it away. The panel is what `git diff` says, never what a tool claimed
  it wrote, so a refused write shows nothing and an edit made by hand shows up too. Terminals
  under 110 columns keep the one-shot `/diff` (the diff printed once in the scrollback).

### Fixed

- **`bodega plugin install` now speaks the backend's real wire shape.** The client decoded a
  `pluginName` the backend never sends, read `strippedGrants` as a list when it is an object, sent skill
  names where the backend expects the staged skill objects, and treated the commit route's `201` as an
  error — so the command could never have completed against a real backend, while its tests (fake client
  shaped like the client) stayed green. The transport is now pinned against JSON recorded from the
  backend, and the mock backend answers `201` like the real route.

### Added

- **`bodega mcp login <name-or-id>` / `bodega mcp logout`** — OAuth 2.1 sign-in for a remote (HTTP) MCP
  server whose auth mode is `oauth` (`bodega mcp edit <name> --auth oauth`): prints and opens the sign-in
  link, then waits for the backend's loopback callback to connect the server. The list shows
  `sign in required` for such a server until then.
- **Plugin updates print a changes table.** Installing a plugin whose name is already installed shows
  each skill and MCP server as new / changed / removed / unchanged; `--skip skill:<name>,mcp:<name>`
  keeps members as they are. A plugin that changed between preview and commit is refused plainly.
- **`bodega plugin export <out-dir> --name <n> --skill … --mcp …`** — write your own skills and MCP
  declarations as an Agent Plugins folder. Credentials and tokens are never written.

### Removed

- **The npm publish release job.** It had never run once (no `NPM_TOKEN` was ever set, and the
  app never used the npm channel), so every release carried a job whose only outcome was a
  "skipped" note. The `packaging/npm` wrapper stays in the tree but is not published; the
  install paths are the GitHub release assets, the one-line installers, Scoop and Homebrew.

### Added

- **Shell completion.** `bodega completion bash|zsh|fish|pwsh` prints a completion script for
  your shell that completes every top-level command plus `run`'s flags. It's generated straight
  from the same command list and flag definitions the CLI itself uses, so it can't fall out of
  sync as commands or flags are added.
- **Sub-agent visibility.** When a run uses `--subagents`, the REPL now shows a live strip under
  the conversation listing each sub-agent's agent, provider/model, status, and tool-call count as
  it works — press Ctrl-G to cycle through them and see the selected one's recent activity, its
  report, and why it stopped early if it did. Headless runs (`--output stream-json`, `--bg`) get
  the same information as `subagent_status` NDJSON frames, and a capped/partial child's stop
  reason now shows up in the run's final summary too.
- **Reuse an already-running engine instead of spawning a second one.** Every `bodega` command
  used to spawn its own backend against the same local data directory the desktop app uses, with
  nothing stopping two processes from writing to the same database at once. The backend now
  leaves a small marker behind while it's running; the next `bodega` command that starts up checks
  for it and, when it finds a live, reusable one, talks to that instead of starting another.
  `bodega doctor` reports whether it found one and whether this run will reuse it.

### Fixed

- **Windows: closing the backend now closes its whole process tree.** Previously, stopping the
  bundled backend on Windows only killed the top-level process — a spawned `llama-server` or MCP
  server could keep running in the background after `bodega` exited. It now uses the same
  `taskkill /T /F` tree-kill the CLI's editor integration (ACP) already relied on.
- **Engine reuse now works with the desktop app open.** The app's backend has no per-launch
  bearer secret at all (it trusts the local loopback connection itself), so its marker file never
  carried one — and a `bodega` run always treated a marker with no secret as "can't safely reuse
  this," spawning a second backend against the same database anyway, exactly the problem the
  reuse feature exists to prevent. A `bodega` run now checks whether the engine actually requires
  a secret before giving up: if an unauthenticated request to it still succeeds, the run reuses it
  with no credential needed. `bodega doctor` reports this case as reusable ("loopback, no secret")
  separately from an engine that truly requires a secret it doesn't have.

## [0.2.8] - 2026-09-03

### Changed

- **Bundles engine `v1.0.0-beta.41`** (`BACKEND_REF` + `BACKEND_EXPECTED_COMMIT` in `release.yml`,
  resolved with `git ls-remote` and hard-checked at build). Sub-agents: the engine can hand a focused
  task to another agent on its own provider and model and get its report back — a `delegate` that
  edits in a throwaway copy with the diff verified before it lands, a read-only `researcher`, and any
  custom agent marked spawnable. Off by default, with its own spend cap, one local sub-agent at a
  time, never recursive, refused under air-gap.

### Added

- `bodega run --subagents` lets THAT run hand focused tasks to sub-agents, which run on their own
  provider and model and report back. Off by default in an unattended run: nobody is watching one, and a
  run that can spawn its own agents compounds wall-clock and spend. A sub-agent still cannot spawn a
  sub-agent — that limit lives on the child's own run and no flag relaxes it. Pair it with `--max-cost`,
  which covers the whole run including its sub-agents.
  The flag is per run: the two settings it flips are snapshotted before the run and put back when it
  ends, however it ends, so the desktop app and later headless runs are not left with sub-agents on.
- `bodega agents` lists this backend's custom agents — each one's provider ("(active)" when it
  follows yours), model, the isolation it runs under when spawned, and whether the main agent may
  hand it work. `--json` for scripts, `--all` to include disabled ones. `bodega run --agent <name>`
  has worked for a while; until now there was no way to see which names were valid without opening
  the app, and a typo failed closed with no list to check against.

## [0.2.7] - 2026-09-02

### Changed

- **Bundles engine `v1.0.0-beta.40`** (`BACKEND_REF` + `BACKEND_EXPECTED_COMMIT` in `release.yml`,
  resolved with `git ls-remote` and hard-checked at build). Runs are governed by progress now: the
  step and time limits are ceilings a run grows toward while it keeps landing files and fixing
  tests, and a stalled run stops early and says why. Also in the engine: Fable 5.1, a hardening
  audit (the local API server's agentic endpoint refuses without a key; `str_replace` cannot edit
  git hooks), old thinking no longer replayed into every local prompt, a calibrated context
  estimate, gateway providers billed at their own prices. Every forced stop names its reason, and
  `bodega run` renders it.

### Added
- `bodega run --max-time <dur>` and `--max-steps <N>`: absolute ceilings for a run. The engine
  (beta.40) now extends a run's budget while it keeps making progress and stops it early when it
  stalls; headless runs never ask, so these are the operator's hard limits. `--max-cost` now also
  sets the engine's per-run spend cap, so the CLI and the engine hold one number.

### Changed
- The end-of-run card names the real stop reason: a stagnation stop says the run stopped making
  progress, a spend-cap stop names `--max-cost`, and the time and step ceilings are named as such.
  "Stopped at a limit" is gone except for reasons this CLI does not know yet.

## [0.2.6] - 2026-09-01

### Changed

- **Bundles engine `v1.0.0-beta.39.1`** (`BACKEND_REF` + `BACKEND_EXPECTED_COMMIT` in
  `release.yml`, resolved with `git ls-remote` and hard-checked at build). A pure-fix engine
  release — everything in it that lives in the engine applies here too:

  - **An agent given full autonomy actually writes its files.** A contract gate answered any
    file write whose name your prompt had not happened to mention with a silent "skipped", so a
    run could report an hour of tool calls and leave nothing on disk. It now keys on the
    autonomy you granted rather than on how the session was launched. This affected cloud
    models as much as local ones.
  - **"Thinking: Off" turns thinking off.** On llama.cpp and Qwen models the reasoning-effort
    setting was read before the off switch, so requests went out with thinking enabled anyway.
  - **A model that is working is no longer cut short for "talking too much."** The cap meant to
    stop a model narrating instead of acting was also counting a model that *was* acting, ending
    runs mid-repair about a third of the way through their budget.
  - **A model that quietly loaded on the CPU says so** — instead of reporting itself ready while
    running many times slower, with nothing explaining why.
  - **Runs that hit a limit end honestly.** A starved context, a turn truncated mid-thought, and
    a tool call cut off mid-write are each detected, recovered where possible, and named when
    not.

### Fixed

- **Cloud spend for Sonnet 5 is billed at the correct rate.** A scheduled price increase that
  the model's vendor publicly cancelled had already taken effect in the engine's pricing table,
  so Sonnet 5 usage recorded from 2026-09-01 was overstated by 50%. Corrected to the standard
  $2/$10 per million tokens. Affects `bodega usage` totals and any spend cap keyed on them.

## [0.2.5] - 2026-08-31

### Changed

- **Bundles engine `v1.0.0-beta.39`** (`BACKEND_REF` + `BACKEND_EXPECTED_COMMIT` in `release.yml`).
  Everything in the app release that lives in the engine applies here too, notably: your
  messages keep their `<` and `>` characters (a server-side filter was silently deleting every
  angle bracket from everything you sent — pasted code lost its generics, and attaching a
  document tripped a false "message too long" error); the agentic loop defers history
  summarization when your local model is busy instead of cutting old messages; the routed
  model tiers now do what the Routing page says; quieter logs.


### Added

- **An answer that was cut short says so.** When the bundled engine stops a run at a limit -
  it stopped narrating and answered early, ran out of steps, or ran out of time - the CLI now
  prints `stopped at a limit (<reason>) - answer may be incomplete`, in the REPL and on both
  `bodega run` outputs (the card and `--output json` as `forcedStop`). Matches the app's marker
  exactly, so both transcripts agree about whether an answer was cut short.

## [0.2.4] - 2026-08-27

### Changed

- **Bundles engine `v1.0.0-beta.38`.** Everything in the app release that lives in the engine
  applies to the CLI too, notably: MCP tools with whole-number parameters (page sizes, limits)
  work now instead of being rejected for every value; a model whose last load never finished is
  not auto-loaded on the next start; managed model downloads verify the publisher's checksum;
  air-gap also blocks chat requests on every provider path, including the two that previously
  slipped through at request time; a stuck local model stops
  in seconds instead of burning the whole iteration budget; provider errors say what is actually
  wrong; eight new provider presets (36 total).
- The model catalog gains the opt-in uncensored section (`conduct` tag with provenance). The
  `--uncensored` list filter is planned for a future CLI release; the catalog data already
  arrives via the bundled catalog refresh.

## [0.2.3] - 2026-08-23

### Added

- Bundles engine `v1.0.0-beta.37` (`BACKEND_REF` + `BACKEND_EXPECTED_COMMIT` in `release.yml`) — the
  backend this CLI bundles now carries the agent-browser wave (snapshot/refs, `wait_for`, six new preview
  actions, persisted site approvals, never-automate list) and the V2 hardening pass.
- `bodega config set browser.widened_enabled=true` / `browser.persistent_sessions=true`
  now accepts the explicit `--i-understand-this-allows-agent-web-browsing` flag
  as an alternative to opening the desktop app's Settings → Safety panel — the
  same "the agent may act as you, until revoked" consequence is printed as a
  warning either way, so a CLI-only operator has a documented path forward
  instead of being told to go install the app.
- Headless runs (`bodega run`) now name WHY a browser-approval frame
  (`preview_interaction:*` / `preview_script`) was auto-rejected —
  `headless_no_approver` — in the run log's `approval_resolved` event and on
  the outgoing approval POST, instead of a silent, unlabeled auto-reject. The
  CLI has never been able to render this client's browser consent card, in
  any mode; the rejection itself is unchanged, only its reason is now stated.

### Fixed

- `bodega harness revert` no longer names "memory and knowledge" as the reason
  a revert is refused — those domains have had revert wiring since the
  pinned engine's beta.36, so the docs and the refusal message were stale.
  The refusal path itself is unchanged: an unsupported domain still refuses
  plainly rather than faking success, and now reads the backend's structured
  refusal code instead of matching a substring in its error text.
- `bodega --effort minimal` (and `/effort minimal` in the REPL) is accepted —
  the backend has supported it since it added GPT-5's minimal reasoning tier;
  the CLI was rejecting it client-side before the request ever reached the
  backend.
- `bodega --help` now shows `bodega serve --webhook` and `bodega skills
  revoke` — both commands worked before this fix, they just didn't appear in
  the top-level help.
- Documented that the app's "Uncensored" prompt template has no CLI
  equivalent today (it's an app setting); a CLI user gets the same effect via
  `bodega refine "persona: ..."` or the app's own template picker.

## [0.2.2] - 2026-08-20

> `v0.2.2`'s own first publish attempt (2026-08-20) also made an EMPTY
> release on the mirror — same silent-publish shape as `v0.2.0` below, a
> different cause: beta.36's backend re-exports the `provider-wire-core`
> package, and a cold clone has no `dist/` for it until `build:core` runs
> first, so all five platform builds failed `TS2305` after `create-release`
> had already cut the (asset-less) release. Fixed same-day by building
> `provider-wire-core` before the backend payload's `tsc` — the same
> prerequisite step the app repo's own pipeline already carries. The tag
> below is the fixed, actually-published build.

### Added

- **`bodega skills revoke <name>`** — withdraws SEC-2 approval for a project
  skill (`.bodega/skills/`). There was previously no way to undo `bodega
  skills approve` short of hand-editing the trust file on disk. `bodega
  skills trust` now also tells you which command undoes an approved entry.

### Changed

- **`bodega skills approve` now confirms before it acts**, the same way
  `bodega loops delete` and `bodega reset` do: on a real terminal it asks
  y/N, in a script or CI it refuses unless you pass `--yes`/`-y`. This is
  also what fixes a real gap — the backend's approval gate now requires the
  caller to report that a human actually confirmed the action whenever the
  app is in Ask or Plan permission mode, and the CLI had no way to send that
  report at all, so `bodega skills approve` would fail with a 403 in those
  modes no matter what you did. If you still hit that 403 (for example the
  permission mode changed between the prompt and the request), the CLI now
  says plainly what to do about it instead of printing a bare "unexpected
  status 403".



> `v0.2.0` was tagged but never published — its release run failed in the
> backend-bundling step before any asset was uploaded (the pinned engine's
> lockfile lost its libc metadata to an npm-11 regeneration, so the bundler
> installed two Linux anydoc slices and its own guard refused the bundle; the
> mac/windows jobs hit a missing root-dependency install in the same step).
> Nothing shipped as 0.2.0; this release supersedes it with the bundler fixed.
> Everything below describes what changed since 0.1.7.

Bundles engine `v1.0.0-beta.36` (`BACKEND_REF` in
`.github/workflows/release.yml`), which carries the backend half of every
command added below.

### Added

- **`--agent <name-or-id>`** on `bodega run` selects a custom agent for the
  turn — the CLI half of the app's custom agents feature. Accepts either the
  agent's name (matched case-insensitively) or its numeric id; a numeric value
  is used directly as the id with no lookup, matching how the app's own picker
  sends one. Resolved against `GET /agent-definitions` before the stream
  opens, so a typo'd name or a disabled agent fails the run immediately
  instead of silently falling back to the default agent.
- **`/memory`** — read back what the agent has remembered across sessions,
  scoped to the current project when it knows one and global otherwise. The app
  has had persistent memory for a while; the CLI had no way to see it, so you
  could not tell whether something had been remembered until it either came up or
  didn't.
- **The status bar shows the active session's title**, next to the model. Useful
  the moment you have more than one session and `/resume` stops being obvious.
- `bodega skills list` / `bodega skills show <name>` and a `/skills` REPL
  command — see what skills are available (including the twelve built-in
  ones) and their descriptions. This is discovery only: skills already ran
  from the CLI before this (the REPL forwards an unrecognised slash command
  to the backend, which trigger-matches it), but there was no way to see
  what existed without reading source. Enabled/disabled state still lives
  backend-side; the REPL doesn't register skills as its own slash commands.
- `bodega plugin install <path>` — install a plugin (a folder or `.zip` bundling
  skills and/or MCP servers) from the command line. Prints what the plugin
  contains before anything is written, asks for approval on any skill that
  wants elevated permissions (`--approve-grants skill:grant,...` outside a
  terminal), and warns rather than fails when one of the plugin's MCP servers
  needs network access you've blocked with air-gap mode. `--dry-run` previews
  without installing anything.
- `bodega harness log` / `bodega harness revert <id>` — see a history of
  those edits and undo one by id. A revert either fully happens or is
  refused with a clear reason (a newer edit stands in the way, or a file was
  changed outside the app since); it never reports success on something it
  didn't actually undo. Reverting memory or knowledge-base edits isn't
  supported yet and this command says so plainly instead of pretending it
  worked.
- `bodega refine "<instruction>"` — propose (and, with `--apply`, write) a
  continual-harness edit from a plain-language instruction. This is a
  deterministic heuristic today, not an AI-authored diff: an instruction
  starting with `persona:` (e.g. `persona: always be terse`) proposes a
  change to your persona overlay, and anything else proposes a shared memory
  note recording the instruction text. Without `--apply` it only prints what
  it would do; nothing is written until you add `--apply`, and applied edits
  show up in `bodega harness log` and can be undone with
  `bodega harness revert`. Project-rule and skill edits aren't supported
  yet.

### Changed

- Startup now verifies the bundled skills and the documentation corpus, not
  just the Node runtime and the server entry point. Every file in a bundle is
  listed in its integrity manifest, but startup only re-checked two of them, so
  a modified skill file or documentation corpus loaded without complaint. Both
  are read by the agent — skills as instructions, the corpus as answers to your
  questions about Bodega — so they are now checked every time. Costs under a
  millisecond. `bodega doctor` still verifies the whole bundle.

- If you're timing `bodega run` on a verification-heavy task (lots of file
  reads to check its own work) and see it taking longer or doing more
  iterations than before, that's expected with this release — a fix
  landed that stops the harness from cutting off verification early. Slower
  wall-clock time here is the harness reading more thoroughly, not a
  performance regression.

### Fixed

- **Four of the six lifecycle hook events never actually ran.** `PreToolUse`,
  `PostToolUse`, `SessionStart` and `SessionEnd` were documented, validated and
  put through the trust prompt — then never fired. Only `UserPromptSubmit` and
  `Stop` did. A hook you wrote, approved, and saw accepted would sit there doing
  nothing, with no way to tell. All six fire now. If you have hooks configured
  for those events, **they will start running**: a `PreToolUse` hook exiting 2
  now genuinely blocks the tool, and a `SessionStart` hook exiting 2 stops the
  session from opening. Worth re-reading anything you wrote against the old
  behaviour before you upgrade.
- Hook scripts had no way to tell which event invoked them — the event name was
  always sent empty, so one script couldn't branch on it. It's populated now.
- `SessionEnd` hook output was written to a screen already torn down, so a
  failing or blocked end-hook was completely silent. It prints properly now.
- Zed's generated custom-agent config snippet (`bodega serve --acp
  --print-config zed`) omitted `"type": "custom"` on the `agent_servers`
  entry, so pasting it into Zed's settings produced a config Zed would not
  accept. Fixed and pinned by a test.
- The CLI never told the backend it was running from a terminal instead of
  the desktop app. This meant a fix that adjusts documentation answers for
  CLI users (so `query_docs` doesn't confidently describe an app-only UI
  panel to someone at a command line) was silently doing nothing for every
  CLI session. Every place the CLI starts the backend — `run`, `serve`
  (both `--acp` and `--webhook`), the interactive REPL, and every other
  command that talks to the backend — now sets this correctly.
- Ship the 12 built-in skills in CLI bundles (`bab5436`). Every assembled
  bundle since the skills feature shipped was missing the `skills/` folder —
  the backend looked for it, found nothing, and loaded silently with zero
  skills. No error was shown. If you noticed the built-in skills weren't
  available, this is why.
- Ship the correct native module for each platform, and catch this kind of
  bug automatically going forward (`b3c4fe9`). Two of five release builds
  were getting a native binary built for the wrong OS/CPU baked in
  (Windows builds shipping a Linux binary; Intel Mac builds shipping an
  Apple Silicon one). Both would fail immediately on the affected machine.
  A packaging test now checks the contents of every release bundle before
  it ships.
- The interactive REPL (`bodega` with no arguments) sent a reasoning-effort
  level of "medium" to the model on every message, even when you didn't ask
  for one. `bodega run` already did the right thing here — it leaves effort
  unset unless you pass `--effort`, so the model (or your settings) picks
  its own default. The REPL now matches. If you run a local model whose
  default reasoning effort is not "medium", you'll see different (and
  usually better-suited) behavior in the REPL starting with this release.
  This also means: if you're comparing timing or quality between an older
  and a newer CLI, effort level is one more thing that can differ — check
  with `/effort` if you want to pin it explicitly.
- `bodega --effort bogus` used to be silently accepted and sent to the
  backend as-is. It's now rejected at startup with a list of valid values,
  matching how `bodega run --effort bogus` already behaved.
- Updating an installed plugin could delete MCP servers it had nothing to do
  with, including ones you'd enabled yourself. The update logic treated "not
  in this update's bundle" as "remove it," when it should have only removed
  servers the plugin itself had previously declared and then dropped.
- Verification was matching a written file to its expected deliverable by
  filename alone, so a write to `test/index.ts` could be graded against a
  deliverable declared as `src/index.ts` — passing or failing work for the
  wrong file. Matching is now path-aware, with a bare-filename fallback for
  deliverables that don't specify a directory.
- `bodega serve --webhook` and `bodega self-update` started anyway when they
  couldn't read your configuration, instead of refusing. That meant an
  admin-pinned air-gap policy could be silently skipped if the config file
  was unreadable or corrupt — an unknown air-gap state was being treated as
  "off." Both now refuse and say why when the configuration can't be read.
- `bodega run --agent <name>` could silently fall back to the default agent
  instead of failing when the backend transport couldn't be asked to list
  agent definitions. In normal use this never triggered — only the real
  backend client serves a run, and it always supports the lookup — but a
  named `--agent` is an explicit selection, and a path existed where it could
  be dropped without a word. It's a hard error now, matching the existing
  unknown-name and disabled-agent failures.
- A PreToolUse hook that rewrites a tool's input on stdout was captured
  internally but had no way to reach the backend (the approval POST has no
  field for it yet). The REPL already surfaced a notice saying so rather
  than silently running the original input as if the rewrite had applied —
  that behavior is now pinned by a test so it can't regress unnoticed while
  the wire protocol catches up.

### Security

- `bodega skills trust` / `bodega skills approve <name>` — a cloned repo's
  `.bodega/skills/` do not load until approved; this is the CLI-side view and
  approval path for that gate (approving from the app worked before this,
  approving from the CLI did not). `trust` lists each project skill and
  whether it's approved; `approve <name>` approves one and prints the content
  hash it was approved against. **Approval is bound to that exact file
  content** — edit an approved skill afterward and it goes back to pending,
  because the hash it was approved under no longer matches. Re-approving
  after every edit is the intended behavior, not a bug: approval means "I've
  reviewed *this* text," not "I trust this filename forever."
- The CLI now refuses `preview_interaction` approvals by name, with a reason,
  instead of happening to reject them because it couldn't read a field it didn't
  know about. Same outcome as before, but a decision rather than an accident — a
  browser consent card can't render in a terminal, so it shouldn't be approvable
  from one.
- A project's `.bodega.yml` can no longer turn air-gap *off* for your whole
  install. Project config can tighten, never loosen; your own flag or env var
  still wins, as does a machine policy pin.
- `--air-gap` no longer runs a turn with air-gap unenforced when the backend
  connection can't apply settings. Previously that case was skipped silently: you
  passed the flag, nothing was pushed, and the run used whatever air-gap state the
  backend already had. It now fails the run and says which value it couldn't
  enforce. Runs where you expressed no air-gap opinion are unchanged — those never
  touched the backend's setting and still don't.
- `bodega config set` refuses to flip the agent-browser switches
  (`browser.widened_enabled`, `browser.persistent_sessions`) and points at the
  app's Safety panel instead. Those enable outbound browsing and persistent
  logins, and both are meant to go through the confirmation there rather than a
  one-liner in a terminal.


## [0.2.1] - 2026-08-14

Superseded by `v0.2.2` above; documented here for the record rather than left
as a silent gap in the version history. `v0.2.1` republished `v0.2.0`'s
feature set unchanged, with only the fix below — see the `v0.2.0` note further
down for what `v0.2.0` itself was.

### Fixed

- The release bundler needed the same two fixes already applied to the app
  repo's own pipelines (an npm-11 lockfile regeneration losing libc metadata,
  and a missing root-dependency install step), so `v0.2.0`'s asset-less
  release could actually publish.

## [0.1.7] - 2026-08-05

Bundles engine `v1.0.0-beta.34` (app commit `ff479d15`), pinned so a moved tag
cannot swap the backend under this release. The previous bundle was
`v1.0.0-beta.33`.

### Engine fixes that land with this refresh

Fixed in the app and shipped here. Listed because several matter more to the
command line than they do in the window.

- **DeepSeek models reason by default again.** The engine was sending an explicit
  instruction not to think unless you had chosen a reasoning level yourself.
  DeepSeek turns thinking on by default, so anyone who had not touched that
  setting was running the model with its headline capability off. On a public
  benchmark of 89 tasks it cost 14 solutions.
- **Long answers from DeepSeek are no longer cut off early.** The output ceiling
  was set to a fraction of what the current models actually support.
- **Three safety rules stopped refusing ordinary commands.** Deleting a directory
  inside your project read as an attempt to wipe the filesystem; flags containing
  the letters of a shutdown command were refused; and a one-line script printing a
  variable named `key` was treated as leaking a certificate. This costs more
  headless than in the window: there is nobody to re-prompt, so a false refusal
  can take the whole run with it.
- **Verification stops failing work for checks it could not run.** This matters
  more at the command line than in the window, because the exit code is the whole
  interface: a run that did the job correctly could still exit non-zero, and in a
  script that is indistinguishable from real breakage. Checks that do not apply
  to the request - no framework to look for, no content requirements to match, no
  test to run - were counted as zero rather than left out, so a short correct file
  could not reach a passing score at any setting. They are now excluded from the
  total, and a file is credited for existing even when it is short.

  Partial, and worth knowing the shape of it: the case that prompted the fix went
  from 35 to 50 out of 100 and still does not pass. The rest is blocked on a
  minimum-length rule that cannot be lifted yet without letting stub files
  certify as verified.
- **Overnight and scheduled runs record their verification results.** Runs
  started without a window never passed a project path into the result handler,
  so nothing they verified was ever written down, and nothing could be re-checked
  for regressions later. This is a headless-only bug: it never affected anyone
  working in the app, and it silently affected everything running under `run`.
- **The model's reasoning is saved for runs without a window.** The window used
  to be the only thing writing it, so headless runs stored none at all.
- **Long tasks on a small context window stop losing their place.** The trigger
  that decides when to summarise was measuring against the whole window,
  including the part that cannot be summarised away, so on a small window it
  summarised roughly every third step and reclaimed almost nothing. One task
  spent eighty-six tool calls re-reading the same nine files. This lands hardest
  on local models and on long unattended runs, which is the CLI's territory.
- **Summaries stopped piling up on top of each other**, each one shrinking the
  room left for the next.
- **The task-list reminder reaches the model again.** It was skipped whenever the
  turn's text began with a brace, which is most of them.
- **A background task can no longer have its working copy deleted while it is
  still running**, and a failed comparison no longer reports "nothing to apply".
- **Answers are no longer written to the transcript twice.**
- **Connected tool servers**: arguments containing spaces are no longer split,
  paginated servers expose all their tools rather than the first page, results
  that are not text say so instead of reporting an empty success, and cancelling
  a run now cancels the request on the server.

### Added - in the CLI itself, not waiting on the engine tag

- **`--output stream-json` now records why a run stopped.** A run that ran out
  of time and a run that finished looked identical in the log. The engine has
  always announced its own budget exits - whether it declined to start another
  step because the remaining time would not cover finishing up, or a step was
  cut off mid-flight - and the CLI was discarding those lines before they
  reached the log.

  Both now appear as `diagnostic` lines carrying the reason in the engine's own
  words, the elapsed time, and the budget it was working against. The reason is
  the only place the actual amount of time held back for finishing up is
  recorded; it varies per run, so a fixed number cannot be assumed.

  This is one line per run at most. Found while measuring the engine on a public
  benchmark: the same set of runs was analysed twice and produced opposite
  conclusions about whether the agent was quitting early or being cut off,
  because the line that settles it was being thrown away.

- **The count of files a run changed now includes files written through the
  shell.** It only counted the dedicated file tools, so writing with a redirect,
  a heredoc, `tee` or `sed -i` - how a model actually writes in a terminal -
  counted as nothing. Runs that had done substantial work reported changing zero
  files, and, decisively, so did six runs that completed their task
  successfully.

  The figure remains a documented lower bound rather than a file list, and it
  deliberately stays conservative: a command that only reads must never be
  counted. Two ways of miscounting were found by replaying 635 real commands
  from a benchmark - a `>` inside quotes is text rather than a redirect, and a
  heredoc feeding a script to an interpreter is input being read, not a file
  being written.

- **`--output stream-json` now records what a run decided to do about its own
  verification result.** The verdict was already in the log; the decision taken
  on it was not. A run could grade its work a failure and stop, and nothing said
  whether a repair had even been considered - so a run that gave up early and a
  run that tried and could not were indistinguishable afterwards.

  Each verification round now writes a `diagnostic` line carrying the score, the
  verdict, and which repair round produced it. A score of zero, a failed verdict
  and "repair round 0" are all written out explicitly rather than omitted,
  because those three together are precisely the case worth finding: a run that
  graded itself a total failure and never attempted a fix.

  Found while measuring the engine against other tools on a public benchmark -
  29 of 33 graded failures had never attempted a repair, and the run logs could
  not explain why. Runs that never reach verification produce byte-for-byte the
  same log as before.

- **`--output stream-json` now records when the engine is pacing itself
  against a provider's rate limit.** A client-side wait of up to 45 seconds
  before a request left no trace in the log - the message rode a channel the
  headless route replaces with a no-op - so a slow run and a paced run looked
  identical. Both the wait and the case where the engine decides not to wait
  now appear as `diagnostic` lines.

  This is bounded by construction: the engine only emits it when a delay is
  actually applied or deliberately skipped, so a healthy run against a
  well-provisioned account writes none of these lines. Found the same way as
  the budget-exit gap above - the cost had to be inferred from timing until
  something finally named it.

- **The `bodega-run-v1.json` schema now describes the diagnostic lines it
  already ships.** The `diagnostic` frame gained roughly two dozen fields over
  recent releases - the per-call timing breakdown, the rate-limit pacing
  fields, the loop-exit reason, the verification-repair fields - and the
  published schema still only listed the original six. Anyone validating the
  stream against the schema we publish would have rejected every diagnostic
  line the engine now emits. The schema still rejects unknown fields; it now
  also lists the ones that are real. A validating test for the diagnostic
  frame was added so the two cannot drift apart again silently.
- **`bodega run` can forward a sandbox-widening root to the backend again,
  now safely.** The app moved this switch from an environment variable
  (`BENCH_SANDBOX_ROOT`) to a backend argv flag, because an environment
  variable inherits across every child process a shell spawns - a leftover
  export from an earlier session silently widened the agent's filesystem
  sandbox on a later, unrelated run. Argv does not inherit, so the new form
  cannot do that by accident.

  This closes the gap that switch left in the CLI: there was no way to pass
  the new flag through, so a Terminal-Bench task whose deliverables lived
  outside the task's working directory could not pass. The new flag is
  intentionally undocumented in `bodega run --help` - it is an operator/bench
  knob, not something a normal run needs - and it validates the path at the
  command line before the engine ever sees it: a value shaped like a Windows
  path is rejected immediately, naming the Git-Bash path-rewriting bug that
  causes it and the two ways to avoid it, rather than failing later with less
  context. A run that does not pass the flag is unaffected - the engine
  receives exactly the arguments it always has.

### Also worth carrying at the same time

- The installer shrank by about 39% in the app. The CLI bundles the engine rather
  than the shell, so the packaging work does not carry directly - but the backend
  dependency pruning does, and the bundling step should be re-measured after the
  refresh to see whether the same `--omit=dev` staging helps here.

## [0.1.6] - 2026-07-27

### Added
- **The bundled engine is now beta.33, the verification release.** The CLI ships
  its own copy of the engine, and that copy was two releases behind, so none of
  the grading work had reached the command line. It has now.

  The change that matters: the engine used to report finished work as failed.
  Measured against an independent verifier across 89 real tasks, it solved 35 and
  called 29 of those failures, because "I could not observe this working" was
  treated the same as "I watched this fail". Only an observed failure fails a task
  now. If you gate CI on `bodega run`, expect fewer spurious non-zero exits.

  Carried over from the release hold that followed: a dropped connection can no
  longer cause the same request to run twice and apply every file edit a second
  time; the credential scanner can no longer be defeated by padding a secret with
  filler, and a second copy of that scanner, which feeds output that leaves the
  machine, had been missed entirely; and a correct plain-text or Markdown file is
  no longer scored 20 out of 100 and reported as a failure, which had capped every
  non-code artifact the engine produced.

  One behaviour change worth knowing: a request naming a range of files with "to"
  or a hyphen ("create out1.txt to out5.txt") is no longer expanded into the files
  in between. The files you actually named are contracted and the run reports as
  unverified rather than claiming success over a partial list. Ranges written with
  "through" still expand.

- **Runs report what they cost.** A finished run now carries token counts, cache
  usage and spend alongside the existing timing and tool counts, in the streamed
  output and in `--output json`. Previously a headless run said nothing about its
  own usage, so there was no way to see what a long job had consumed without
  opening your provider's dashboard.
- **A run identifies itself as unattended.** `bodega run` now tells the engine it
  is running headlessly. Several behaviours meant for unattended work were gated
  on that signal and had never once taken effect from the command line, because
  the signal was never sent: capable models were held to guard rails intended for
  small local ones, and an unattended run was not given the extra room it was
  meant to have. Interactive and editor sessions are unaffected, and the field is
  omitted entirely when it does not apply, so an older engine sees no change.

### Changed
- **Exit code 3 now also means "the run could not be graded".** The engine has
  started reporting a new outcome: when it edits files under a task it could not
  classify, the only thing it can check the work against is the list of files it
  just edited, so every check would pass by construction. Rather than report a
  pass it did not earn, it now marks the run ungraded.

  Read literally, that outcome looks like a verification failure, and without this
  release the CLI would report it as one — a run that exited `0` yesterday would
  exit `1` today and print `FAILED`, with a score to match and nothing you could
  act on. That is wrong twice over: the work usually happened, and nothing about
  it was actually checked.

  So an ungraded run now exits `3` — the code that already means "the run
  finished and nothing was verified" — and reads `NOT VERIFIED`, not `FAILED`. It
  is deliberately not `0`: nothing was verified, so it should stay visible rather
  than pass silently. `--qel-threshold` no longer applies to these runs, because
  the score it would compare against is meaningless. If your pipeline should not
  fail on an unverified run, `qel: { no_verification_is: pass }` in `.bodega.yml`
  maps `3` to `0` and now covers this case — while a genuine verification failure
  still exits `1` and is still not maskable that way.

  **If you gate CI on the exit code, check how you treat `3`.** `--output json`
  and `--output stream-json` carry a new `ungraded` field alongside `passed`, so
  a script can tell an ungraded run from a task that simply had nothing to verify.
- **`bodega loops run` still exits 0 for a parked run**, and now accepts
  `--output json` so a script can read the run's `status` (`applied`, `parked`,
  `no_changes`, `qel_failed`, `failed`, `cancelled`) and act on the difference. A
  parked run means nothing was applied and it needs your review — a real outcome,
  not a break — and more runs will park now that the engine grades more strictly.
  Changing the exit code would have broken every pipeline that reads non-zero as
  "the loop failed", so the detail lives in the output instead.

### Fixed
- **Cloud Boost spend now counts against `--max-cost`.** Boost is billed on a
  separate channel from your own provider keys, and the CLI was only watching the
  key-based one. A run served entirely by Boost reported a cost of $0 and never
  reached its ceiling, no matter how much it spent. The interactive session had
  the same blind spot: the on-screen counter sat at $0, and the warning at 80% and
  the block at 100% never fired. Both surfaces now count it. The two channels are
  tracked separately and added together, so neither figure can distort the other.
- **`.bodega.yml` keys that the built-in help documented now actually work.**
  `bodega help air-gap` described `privacy: { air_gap: true }` and
  `bodega help modes` described `permission_mode:`; neither key was ever read, so
  a config written from that help did nothing and said nothing — including, in the
  air-gap case, leaving someone who believed their data stayed local un-gapped.
  Both spellings are now honoured, and the help has been corrected to name the
  real keys (`general.air_gap` and `mode`).
- **Built-in help no longer documents things that do not exist.**
  `bodega help air-gap` offered `bodega --air-gap` for an interactive session;
  there is no such option there and it exited with an error. Use
  `BODEGA_AIR_GAP=1` or `.bodega.yml` for the REPL, and `bodega run --air-gap`
  for a single run. `bodega attach --help` and `bodega run --help` listed an exit
  code 5 that is reserved and never returned, and `attach --help` omitted codes
  6, 7 and 8 that it genuinely forwards.
- **Prompts that start with a dash no longer fail to run.** A prompt whose first
  line began with `-` was read as an unknown command-line option and the run
  refused to start. Prompts beginning with `-` or `--` now work, `--` still ends
  the options explicitly, and genuinely malformed options still fail loudly.
- **A failed run now tells you why.** Runs that ended without a verdict exited
  with a bare error code and no explanation at all. They now report what
  happened, how long they ran and how much work they did — in the streamed
  output, in `--output json`, and as a line on standard error.
- **Works on older Linux distributions again.** The bundled database library was
  built against a newer system C library than Debian 11, RHEL 8 and similar
  ship, so the CLI failed on startup there. Linux builds now target an older
  baseline, and the build fails if that ever regresses.
- **A run that produced nothing now says where it stopped.** If a run stalled
  before it began working, the log held one line saying it had started and then
  nothing at all — no error, no clue, no way to tell a stuck engine from a slow
  model. The engine had been reporting its startup progress the whole time and
  the CLI was discarding it. Those messages now appear in the run log, so silence
  is diagnosable rather than opaque.
- **The count of changed files was almost always zero.** Writes were matched
  against a list of tool names that did not include the one the engine actually
  uses to write files. Anything reading that count — a script deciding whether a
  run did any work — saw finished work as a no-op. It now also confirms a file
  was written rather than merely read, so the count does not swing the other way.
- **Runs report per-step progress.** The engine marks each step of its loop and
  the CLI was dropping those markers, so a long quiet stretch could not be
  explained: there was no way to tell one very slow step from several ordinary
  ones. They now appear in the streamed output, which is enough to see where a
  run's time actually went.

## [0.1.5] - 2026-07-24

### Changed
- **The bundled engine is updated to the beta.32.1 desktop release.** Headless
  and REPL runs now inherit a wave of agentic-loop and small-model reliability
  fixes: a deadlock fix for tool calls that pass array/string parameters in
  unexpected shapes, a better greeting/small-talk gate and coaching messages
  for lighter local models, a cap on repeated narration with a nudge when a
  model invents its own tool-call syntax, recovery for three more invented
  tool-call formats, better routing of memory-related requests, compatibility
  with community model files that used a strict system-prompt template and
  previously rejected every request, a fallback for reading free GPU memory
  when the primary method is unavailable, and a fix for stale verification
  state leaking between sessions. The CLI's own code has no changes required
  for this wave.

## [0.1.4] - 2026-07-17

### Added
- **A workspace trust prompt before a cloned repo's config can run anything.**
  A project's `.bodega.yml` or `.bodega/mcp.json` can define MCP servers, and
  `.bodega.yml` can define lint and test commands for the repair loop. Opening a
  repository no longer applies those automatically. The first time a project
  wants to register an MCP server or run a project-defined command, it is held
  until you approve it, and approval is remembered per project. Editing the
  command, its arguments, its credential variable, or its network setting asks
  again. In a non-interactive run, an unapproved command is skipped rather than
  run.
- **A machine policy that a single session cannot loosen.** An administrator can
  place a policy file that forces air-gap on, adds tools to the deny list, or
  disables the repair loop. A per-session flag can make a session stricter but
  never weaker than the policy, at startup and during a session.

### Changed
- **The bundled engine is updated to the beta.32 desktop release.** Headless and
  REPL runs now inherit that release's agent-reliability and safety work,
  including the vendor-tuned sampling defaults for local model families, the
  centralized cloud-spend recording and caps, and the wider air-gap coverage.

### Fixed
- **Secrets are removed from traces and session exports.** A `--trace` file and
  a `session export` now redact API keys, tokens, and private keys that appear
  in tool output, so a run that reads a credentials file does not copy it into
  the exported record. Redaction happens before anything is written to disk.
- **The built-in update channel is pinned to its official host.** The default
  self-update source is checked to be the official host before it is used, so a
  tampered configuration cannot silently repoint it. An explicitly configured
  custom channel is still honored.

## [0.1.3] - 2026-07-10

### Changed
- **Bundled desktop engine refreshed** so headless and REPL runs inherit the
  latest desktop-release agent-reliability fixes. (Released without a detailed
  changelog entry at the time; recorded here for completeness.)

## [0.1.2] - 2026-07-09

### Added
- **`bodega loops` is now a full scheduling surface.** The backend's scheduled
  automations (cron/interval agentic tasks, QEL-gated apply, run in isolated
  worktrees) were already there; the CLI now drives all of them:
  - `loops list` shows each loop's trigger, enabled state, last run
    (status + QEL score) and next fire time. `--output json` emits the same
    with the full last-run row embedded for scripts.
  - `loops create --name N --prompt "..." --cron "0 3 * * *"` defines a loop.
    Cron is sanity-checked client-side (field count) before the round-trip; the
    backend stays authoritative. `--disabled` defines a loop without arming it;
    `--every`, `--on-comment`, and `--on-map-stale` cover the other triggers.
  - `loops enable|disable|delete <name-or-id>` manage a loop by name or id.
    `delete` asks for confirmation on an interactive terminal (`--yes` skips it;
    a non-interactive shell without `--yes` is refused rather than run blind).
  - `loops runs [name-or-id]` lists recent runs; `--follow` streams the live
    `loop_run_status` channel until you press Ctrl-C, optionally filtered to one
    loop. Every subcommand works non-interactively with clean tabular or JSON
    output, and fails soft against an older backend that lacks loops.
- **Verification moat is now visible.** The CLI surfaces three QEL verification
  lanes it already inherits from the backend but never showed:
  - **Answer grounding.** On question turns that used tools, the score card and
    the human / headless run summary now note when an answer was NOT grounded
    (for example, it cited a file it never read). A grounded answer prints
    nothing to avoid noise; `/proof` shows the full verdict either way. It is
    advisory - it never changes the verdict or the exit code.
  - **Modification proofs.** The project's own lint/test results (such as
    `npm run lint --silent` and `npm test --silent`) now appear as pass/fail
    proof-gate lines on fix/modification tasks, as fix evidence.
  - **Contract review.** Code-review findings surface on the card and in `/proof`
    when the backend's contract-review lane is on.

  In `--output stream-json`, the `qel_score` envelope carries three new optional
  fields - `proof_gates`, `answer_grounding`, and `code_review` - documented in
  `docs/schemas/bodega-run-v1.json`. They are additive and omitted when absent, so
  existing consumers and older backends are unaffected. No exit-code semantics
  change; this is visibility only.
- **Drift badge in the status bar.** When the connected backend has a recorded
  Drift Radar scan showing regressions in the current project's contracts, the
  status bar now shows a `drift: N` indicator and drops a one-time note into the
  scrollback pointing you at the desktop app's Drift Radar for the full report.
  It is read-only (the CLI never runs a scan itself) and fails silent against an
  older backend that predates Drift Radar, so it never breaks or warns spuriously.
- **Keyless first run lands you on a local model.** First launch with no cloud
  key configured now offers a local option (Ollama or bundled llama.cpp) right
  in the onboarding flow, driving the backend's own hardware-fit and install
  logic. Pick it and onboarding installs the runtime if needed, pulls a
  recommended model for your machine, and verifies it responds, so you can
  start coding with no API key at all. Onboarding never touches licensing.

### Changed
- **`bodega run` splits transport death into its own exit code.** Exit `3`
  previously meant two different things: a task that finished with nothing to
  verify, and the backend connection dropping mid-run. Scripts could not tell an
  unverifiable success from a lost connection. Exit `3` now means only
  "finished, nothing to verify"; a mid-run connection drop is a new exit `8`. A
  retry is pointless on a `3` but may help on an `8`. Pre-run spawn/connect
  failures are still exit `2`, and `no_verification_is: pass` still maps `3` to
  `0` but never masks an `8`.

### Fixed
- **Piped stdin can no longer hang or exhaust memory on a runaway pipe.**
  Headless prompt reads from stdin are now bounded to 10 MiB, so a
  misconfigured pipe (or a mistaken `cat /dev/zero |`) fails fast instead of
  reading forever.
- **Em-dashes normalized to hyphens across CLI output.** All user-facing
  messages (TUI notices, headless summaries, error text, help copy) now use
  plain hyphens for consistency; comments and internal identifiers are
  unaffected.

## [0.1.1] - 2026-07-07

### Added
- **`--model` is validated at launch.** A mistyped model id (`bodega --model
  qwen3.6:27` for an installed `qwen3.6:27b`) used to boot a normal-looking
  session whose first message errored with "model not found". The REPL now
  checks an explicitly passed `--model` against the provider's installed list
  at boot and fails fast with a did-you-mean suggestion plus the installed
  models. Best-effort by design: offline boots, disconnected providers, and
  empty lists skip the check - the pre-flight can never block a boot.

### Fixed
- **Linux builds now run on mainstream distributions.** The bundled SQLite
  native module required GLIBC 2.38, so the CLI crashed on startup on every
  Linux older than Ubuntu 24.04 - that includes Ubuntu 22.04 LTS, Debian
  11/12, RHEL 8/9, and Amazon Linux. Because every command loads the database
  first, the whole CLI was affected, not one feature. The release now compiles
  the Linux native modules from source against an older GLIBC (2.35), and the
  pipeline fails the build if the result still needs a too-new GLIBC, so a
  Linux binary can never silently ship broken again.
- **Switching models before your first message no longer blanks the screen.**
  A pre-first-turn `/model` switch dropped the welcome pane, leaving only a
  lone "switched to …" line on an empty screen. The switch notice is now a
  quiet note: the welcome pane stays up (its header already shows the new
  model) and the notice renders in the scrollback once real conversation
  exists. Error and command-reply notes still surface immediately.

## [0.1.0] - 2026-07-06

The first public release of Bodega One Code (CLI): the terminal surface of the
Bodega One family. Local-first agentic coding with BYOK cloud models or fully
local models (Ollama / llama.cpp), a hard air-gap mode, MCP management,
parallel fleet runs with QEL-verified winner selection, scheduled QEL-gated
loops, and a data dir shared with the Bodega One desktop app and agent.

### Added
- **`bodega loops` - scheduled agentic tasks from the terminal** (gap C4). Create,
  list, run, inspect the history of, import, and delete Bodega Loops: recurring
  agent tasks that run on your machine in an isolated git worktree, gated by QEL
  verification. `bodega loops create --name nightly-docs --task "Sync inline docs"
  --every "0 2 * * *"` (cron; or `--every 30` interval minutes, `--on-comment`,
  `--on-map-stale N`); `--apply park|auto-if-pass|dry-run`. The CLI implements no
  scheduler of its own - loops live in the shared backend, so a loop created here
  is the same one the desktop app manages, and vice-versa. `run` waits for the
  verified result and derives its exit code from it (QEL-failed → 1) so a script
  can gate on the outcome. The scheduler fires while loops are enabled (Settings
  → AI Behavior → Loops).
- **Family-shared canonical data dir** (gap S1, CLI half). The three Bodega apps
  (desktop IDE, CLI, agent) historically resolved *three different* data dirs, so
  a provider key / MCP server / session set in one was invisible to the others.
  The CLI now resolves a neutral canonical location - `%LOCALAPPDATA%\BodegaOne`
  (Windows), `~/Library/Application Support/BodegaOne` (macOS),
  `${XDG_DATA_HOME:-~/.local/share}/bodegaone` (Linux) - shared with the other
  surfaces, with `BODEGA_USER_DATA_DIR` as the family override. Migration is
  strictly safe: **copy-not-move** (a legacy dir is never touched, so no data can
  be lost), auto-migrating only the unambiguous single-legacy-source case;
  when multiple populated legacy dirs exist it does **not** guess which is
  authoritative - it keeps using the app's own historical dir (zero regression)
  and drops a `.consolidation-pending.json` marker for a deliberate consolidation
  step. A hot source (a legacy app actively writing its DB) defers migration to a
  later launch. Verified live on a 3-dir machine: the CLI kept its own dir and the
  48MB desktop DB was byte-identical afterward. Agent + desktop halves land next.
- **`bodega fleet`** - run one task N ways in parallel and pick the best (gap
  C2a). `bodega fleet "<task>" --split 3` forks 2–5 isolated git worktrees
  (branch `fleet/<id>/m<n>`), runs each as an independent detached background
  run with its own backend process and its own QEL verdict, then
  `fleet attach` aggregates the results (status, exit, QEL score, diffstat per
  member) and RECOMMENDS the highest-QEL passing member - apply stays explicit
  (`fleet apply <id> --member <n> [--strategy squash|merge]`, clean-tree-only),
  matching the backend GUI's own manual-winner posture. `fleet
  status|list|discard` complete the lifecycle. The 2–5 split cap is the C2a
  concurrency gate (mirrors the backend's VRAM ceiling policy - unbounded
  concurrent heavy runs have taken machines down in live testing). Run flags
  after the prompt (`--model`, `--deny-tools`, `--qel-threshold`, …) forward to
  every member; `--cwd/--bg/--task-id/--output` are fleet-owned and rejected.
- **`bodega models`** - local runtimes and models from the terminal (gap C3). A
  keyless user on a bare machine no longer needs the desktop app to get a local
  model: `bodega models` shows runtime status (Ollama + bundled llama.cpp) plus
  hardware-fit model recommendations for THIS machine; `models install-runtime
  [--runtime ollama|llamacpp] [--flavor <gpu>]` drives the backend's existing
  checksum-pinned installers with live progress; `models pull <name>` /
  `models rm <name>` download and delete models via the backend's model hub
  (idempotent pulls - attaching to an in-flight download is not an error). The
  CLI re-implements zero install logic: platform installers, SHA256 manifests,
  path sandboxing, and the air-gap gate all stay in the shared backend (installs
  and pulls are refused under air-gap; read-only status still works). Onboarding-
  wizard integration ("set up a local model" as a first-run choice) is queued as
  the follow-up slice.
- **MCP management from the CLI** - `bodega mcp add / edit / remove / credential`
  give a CLI-only user full parity with the desktop app's MCP settings, by driving
  the backend's existing per-server CRUD routes (duplicate-name rejection,
  cmd validation, auto-connect on add, and live reconcile on edit all come from the
  shared backend for free). `add --from <file>` imports a portable
  `{"mcpServers": {...}}` JSON config (stdio entries only - remote url/SSE/HTTP
  entries are rejected loudly since the backend is stdio-only; `--url` is reserved
  and errors with the same explanation). Credentials go through the encrypted
  credential route: `credential set <server> --stdin` keeps secrets out of shell
  history; multi-var `env` blocks import their first key and warn about the rest.
- **Project-local `.bodega/mcp.json`** - a second, tool-portable declarative source
  next to the `.bodega.yml` `mcp:` block (YAML wins on any id collision, with a
  warning). Embedded credential values are never auto-stored at startup - a warning
  points at the one-time `bodega mcp add --from` import instead.
- **Headless MCP reconcile** - `bodega run` now applies the project's MCP config
  before streaming, exactly like the REPL always did. Previously the reconcile was
  REPL-only, so scripted/CI runs silently saw none of the project's MCP servers.
- The `.bodega.yml` `mcp:` block (and the reconcile diff) now supports
  `enabled_tools` / `disabled_tools` / `required` - these backend config fields were
  previously unreachable from the CLI's write path entirely.
- **`bodega self-update`** (alias `bodega update`) - a non-destructive check of the
  release channel that reports whether a newer version is published and how to get
  it. `--channel <url>` (or `$BODEGA_UPDATE_URL`) points at a release manifest;
  `--json` emits the raw status. This closes a shipped broken promise: the
  CLI-vs-backend version-skew warning already told users to "run: bodega self-update",
  but no such command existed. In-place binary replacement is deliberately deferred
  until signed releases are published and the swap can be verified per platform
  (the Windows running-exe lock is the tricky case); today the command checks and
  instructs. Dev/un-stamped builds and an unreachable channel both fail soft - a
  version check never breaks the CLI or a wrapper script.
- **Portable sessions** - `bodega session export <id> --out <file>` writes a session's
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
  prompt text) - a headless run you believed was sandboxed had no tool restriction
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
  under the input - a constant float that grew as an open `/`-command palette
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
  exceeds the model's resolved context window (common on local models - the
  backend caps `num_ctx` to a VRAM-safe size, so a big model on a modest GPU can
  resolve to as little as 4096), the model quietly loses the task, does no
  verifiable work, and the run exits 3 with an empty-looking result - no clue
  why. The human card now warns with the actual numbers ("the prompt (8190
  tokens) exceeded this model's context window (4096) on your hardware…") and
  suggests a smaller/larger-VRAM model; `--output json` carries a
  `contextOverflow` object. Found + fixed during in-house battle-testing of
  `bodega fleet`, where two members reproduced the failure identically.
- **`bodega models` reported a running Ollama as "not installed."** The status
  decoder used `installed`/`version` but the route emits
  `alreadyInstalled`/`pinnedVersion` - so the fields silently defaulted to
  false/empty. Now decodes the routes' real keys, with fixture-shaped regression
  tests so a future rename fails the test instead of the UI. (Battle-test find.)
- **Battle suite is hermetic against the host environment.** With
  `BODEGA_INSTALL_ROOT` (or `BODEGA_NODE`) set in the shell, the suite would drive
  the binary against the *real* backend instead of the in-repo mock - leaking a
  process and (on Windows) hanging ~60s then failing on temp-dir cleanup. The gate
  now neutralizes both for every battle test.

### Added
- **Image input** - attach images to a prompt for vision models: `@path/to/image.png`
  in the REPL composer, or repeatable `--image <path>` on `bodega run`
  (png/jpg/jpeg/gif/webp). Paths go through the same sandbox as `@file` refs. If the
  resolved model can't see images, the CLI warns you up front instead of silently
  dropping them.
- **Full-turn checkpoints** - `/undo` now reverts both the files *and* the
  conversation for the last turn (previously files only), and a new **`/rewind`** steps
  back N turns at once: `/rewind` lists available checkpoints, `/rewind <n>` rolls back
  to that point.
- **Per-session cost budget** - cap BYOK/cloud spend with `bodega run --max-cost <usd>`
  or `budget.max_cost_usd` in config. A `/budget` command shows spend against the
  ceiling in the REPL; the CLI warns at 80% and hard-stops at 100%. An over-budget
  headless run exits with a new distinct code **7**, so CI can tell "too expensive"
  apart from a QEL failure.
- **Two-phase backend readiness** - headless runs now wait (by default) for the
  backend's full toolset (MCP servers + skills) to finish loading after the health
  gate, so a run dispatched at startup doesn't execute with a partial toolset.
  `--no-wait-settle` opts out and dispatches at first ready; if the backend never
  reports settled within the timeout, the run proceeds with a note rather than failing.
- **`bodega run --bg`** - detach a headless run and get a task id back immediately,
  with `bodega attach <id>` to wait for the result, plus `logs` and `runs` to inspect
  background tasks.
- **`bodega run --mode ask|plan|act`** + **`--profile <name>`** - set the permission
  mode explicitly in headless runs, or apply a named permission profile stored in the
  backend's settings.
- **Versioned `stream-json` output** - `--output stream-json` events now carry a v1
  envelope, so scripts can pin against a stable event taxonomy.
- **Session lifecycle tracking** - the CLI now tracks session state transitions
  explicitly, so resumed, replaced, and ended sessions can't be written to after the
  fact.
- **Richer diff rendering in the REPL** - word-level inline highlighting for modified
  lines, and unchanged context collapses behind a summary line instead of filling the
  scrollback.
- **Permission-mode pill** - the REPL status bar shows the current permission mode in
  color, cycles it with Shift+Tab, and adds a prompt-cache hit-rate segment. - a GitHub webhook ingress that turns a labeled (or
  opened-with-label) issue into a verified, QEL-gated automation run. Every delivery is
  HMAC-SHA256-verified over the raw body against the shared secret
  (`--webhook-secret` or `$BODEGA_WEBHOOK_SECRET`) *before* its body is interpreted -
  an unsigned/mis-signed request is rejected `401` and never dispatches. A matching
  delivery (default trigger label `bodega`, set with `--webhook-label`) runs through the
  backend's verified-automation pipeline (isolated worktree → headless run → PR) via
  `POST /api/github/run-task`; the GitHub PAT stays server-side and never crosses the
  CLI boundary. Runs dispatch serially (the backend is single-flight) and asynchronously
  (within GitHub's ~10s delivery window). Binds to `127.0.0.1:8787` by default
  (`--webhook-addr`); front it with a tunnel/reverse-proxy for external delivery.
  Exactly one of `--acp` or `--webhook` is required.
- **`bodega run --tier fast|smart|code`** + **`/tier`** - force the routing tier for a
  turn. The CLI resolves the tier to a concrete model via the backend router
  (`llm.fast_model` / `smart_model` / `code_model`) and hard-pins it (`modelPinned=true`,
  so routing rules are skipped - an explicit tier outranks rules). `--tier` is mutually
  exclusive with `--model` and `--dual`; `/tier off` reverts to the pre-force model.
  Single-active presets (llama.cpp / LM Studio) resolve every tier to the default model
  and the CLI surfaces that verbatim. Reuses `POST /api/routing/classify`; no backend
  change.
- **Shell-fix offer in the REPL** - when a `!<cmd>` exits non-zero, a one-key
  "press y to ask the agent to fix it" appears; `y` sends the failing command +
  its output to the agent as the next prompt. Any other key dismisses it.
- **`bodega run --output-schema <file>`** - validate the run's final answer against a
  JSON Schema. On a mismatch (or a non-JSON answer) the run exits **6** - a distinct
  code so CI/jq pipelines can gate on a typed answer shape, separate from the QEL
  verdict. It's the last gate (only turns a passing run into a failure) and is skipped
  under `--dry-run`. A surrounding ```json fence is tolerated.
- **`bodega review`** + **`/review`** - AI code review of your uncommitted changes
  (working-tree diff, or branch-vs-base), printing structured Critical / High /
  Suggestions findings. `--full` / `/review full` ignores the per-project delta cache.
  Reuses the backend reviewer; honors air-gap (needs a local model when on).
- **`bodega init --analyze`** - scan the project and draft `BODEGA_PROJECT.md`
  (stack, build/lint/test commands, structure) in one command, instead of running
  the onboarding wizard. Reuses the backend ProjectAnalyzer.
- **`bodega run --fallback <csv>`** + **`/fallback`** - set the failover model chain
  (`llm.fallback_models`). On a retriable primary failure (rate-limit/5xx/timeout) the
  backend fails over to the next healthy model. `/fallback off` clears it.
- **`bodega run --dual architect=<id>,editor=<id>`** + **`/dual`** - enable the
  sequential dual-model pipeline (`agent.dual_model`): a planner model drafts, an editor
  model executes. Distinct from `/model architect …` (which writes concurrent routing
  rules). `/dual off` disables it.
- **`@file` line-ranges** - attach a precise slice of a file: `@path/file.go:10-40`
  (also `:N` for one line, `:N-` to EOF, `:-M` from the top). Strictly additive - a ref
  with no range reads the whole file as before; the path sandbox is unchanged.
- **`.gitignore` / `.bodegaignore`-aware `@`-picker** - the file autocomplete now reads
  the project-root `.gitignore` (and a `.bodegaignore` supplement) and prunes those
  paths, with glob support (`*.snap`, `gen/*`). Build output and generated files no
  longer clutter the picker.
- **`bodega run --effort`** - set the reasoning tier (off|low|medium|high|xhigh|max) in
  headless/CI runs (previously REPL-only). Out-of-range tiers clamp per model.
- **`bodega --bare`** - start the REPL with no project customizations (hooks, custom
  commands, MCP reconcile) for a clean, reproducible session and config bisecting.
- Interactive REPL (Bubble Tea): streaming, tool/plan approvals, unified diffs, the QEL
  score card, model + dual-model picker, slash commands, `@`-file / `!`-shell composer.
- Headless `bodega run` with the QEL-derived exit-code contract (0–5), `--output`
  human|json|quiet|stream-json, `--trace`, `--deny-tools`, `--qel-threshold`,
  `--dry-run`, `--answers`, `--air-gap`, and stdin-piped prompts.
- `bodega serve --acp` - ACP stdio relay for editor integration (`--check` to diagnose).
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
- **Faster interactive rendering** - the REPL scrollback re-renders incrementally from
  a cached prefix instead of rebuilding the whole transcript every frame, which keeps
  large sessions responsive.
- **Faster startup** - independent post-onboarding boot round-trips now run in
  parallel instead of sequentially.
- **Background file indexing** - the `@`-autocomplete file index is built off the UI
  thread at startup, so the first `@` no longer stalls the composer in large projects.
- The REPL usage text now documents `--repair` and `--effort`.
- The reasoning-tier valid set + validation are now a single source of truth in
  `internal/core/model` (shared by `bodega run --effort` and the REPL `/effort`).
- A pre-release battle-test suite now gates the first release (internal; no user-facing
  behavior change).

### Fixed
- `/undo` after switching sessions (`/resume` or `/new`) no longer restores files from
  the previous session - checkpoints are invalidated on any state-replacing transition.
- The REPL renders on the alternate screen buffer, so quitting restores your terminal
  scrollback instead of leaving the session transcript behind.
- Stacked overlays (pickers, approvals) no longer collapse the scrollback when the
  terminal is short - chrome height is clamped.
- `bodega review` sends the resolved project path instead of the raw `--cwd` flag
  value.

### Notes
- The release mirror is `BodegaoneAI/bodegaone-releases`.
