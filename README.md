# Bodega One Code

Binary distribution for **Bodega One Code**, the terminal surface of [Bodega One](https://bodegaone.ai) — a local-first AI coding agent that runs on your machine, against your models, and checks its own work. The source lives in a separate private repo. This repo carries the built bundles, checksums, install scripts, and the self-update manifests.

## What is Bodega One Code?

An agentic coding CLI with an interactive terminal session and a headless mode for scripts and CI. It spawns its own bundled backend (Node runtime included — nothing to install first) and works two ways:

- **Your keys**: OpenAI, Anthropic, Google, and 20+ other providers, using API keys you already have. No account with us, no metering, no monthly token anxiety.
- **No keys at all**: point it at local models. `bodega models` installs Ollama, recommends models that actually fit your GPU, and pulls them — from zero to a working local coding agent in one command.

What it does that other coding CLIs don't:

- **Verified output.** Every agentic run is scored by the Quality Enforcement Layer — contract extracted from your request, code compiled, tests run, result probed. `bodega run` derives its exit code from the verification, so your scripts can trust it.
- **A real air-gap.** Flip air-gap mode and zero bytes leave your machine — enforced in the engine, not promised in a privacy policy.
- **Fleet runs.** `bodega fleet "task" --split 3` races the same task in parallel isolated worktrees and recommends the winner by verification score.
- **One shared brain.** Keys, settings, sessions, and MCP servers are shared with the Bodega One desktop app and agent — configure once, use everywhere.
- **Windows-first.** Native. No WSL required, ever.

## Install

### Windows
```powershell
irm https://github.com/BodegaoneAI/bodegaone-cli-releases/releases/latest/download/install.ps1 | iex
```
Or with Scoop (the clean path — no SmartScreen prompt):
```powershell
scoop install https://github.com/BodegaoneAI/bodegaone-cli-releases/releases/latest/download/bodega.json
```

### macOS / Linux
```sh
curl -fsSL https://github.com/BodegaoneAI/bodegaone-cli-releases/releases/latest/download/install.sh | sh
```
macOS bundles are signed and notarized (Developer ID). Homebrew users: the stamped formula ships as a release asset (`bodega.rb`).

### Manual download

Grab your platform's archive from the [latest release](https://github.com/BodegaoneAI/bodegaone-cli-releases/releases/latest):

| Platform | Asset |
|---|---|
| Windows x64 | `bodega-windows-amd64.zip` |
| macOS Apple Silicon | `bodega-darwin-arm64.tar.gz` |
| macOS Intel | `bodega-darwin-amd64.tar.gz` |
| Linux x64 | `bodega-linux-amd64.tar.gz` |
| Linux arm64 | `bodega-linux-arm64.tar.gz` |

Unpack anywhere and run `bin/bodega` (`bin\bodega.exe` on Windows). First run walks you through setup; `bodega doctor` verifies the install.

### Verify a download

Every release carries `checksums.txt` plus a `.sha256` sidecar per asset:

```sh
sha256sum -c --ignore-missing checksums.txt
```

## Updating

`bodega self-update` checks this repo's release channel and tells you when a newer version is out (it never replaces the binary behind your back — re-run the installer to upgrade). Scoop and Homebrew update through their own flows.

## Getting started

```sh
bodega                 # interactive session (first run opens the setup wizard)
bodega models          # install a local runtime + pull a model that fits your hardware
bodega run "fix the failing test in ./pkg"   # headless: exit code follows verification
bodega doctor          # check your install
bodega help            # the full command tour
```

## License & source

Proprietary. The source is developed in a private repository; this repo exists so downloads, checksums, and update manifests live at stable public URLs. Issues and feedback: open an issue here.
