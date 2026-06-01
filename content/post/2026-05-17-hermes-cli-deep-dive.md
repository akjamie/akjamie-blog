---
layout: post
title: "Building Production Agentic CLIs: A Deep Dive into Hermes"
date: "2026-05-17"
author: "Jamie Zhang"
tags: ["Hermes", "CLI", "Agentic AI", "TUI", "Plugin System"]
categories: ["AI"]
description: "A Staff-Engineer-level study of the Hermes Agent command-line interface — from filesystem layout to REPL internals, model selection to plugin extensibility, and the design patterns that make it production-grade."
image: "/img/ai-01.jpg"
keywords: ["Hermes Agent", "CLI", "TUI", "command line interface", "React Ink", "argparse", "slash commands"]
---

# Building Production Agentic CLIs: A Deep Dive into Hermes

> *A Staff-Engineer-level study of the Hermes Agent command-line interface — from filesystem layout to REPL internals, model selection to plugin extensibility, and the design patterns that make it production-grade.*



---

> [!NOTE]
> ### ⚡ Executive TL;DR
> This document is a comprehensive, Staff-Engineer level deep dive into the architecture, execution pipeline, and design patterns of the **Hermes Agent CLI**. If you are building production-grade agentic command-line interfaces, these are your key takeaways:
> - **Unified Command Registry:** A single source of truth (`CommandDef` dataclass) dynamically derives help systems, autocompleters, and platform manifests (Telegram, Slack, Discord) to guarantee consistency across surfaces.
> - **Pre-Import Bootstrapping:** Intercepting CLI profiles and modifying paths *before* heavy modules are imported prevents stale module-level cache issues.
> - **Process Boundary for TUI:** Spawning the TypeScript/Ink TUI as a separate process communicating via JSON-RPC keeps React's rendering reconciler and Python's synchronous loop isolated, scaling cleanly.
> - **Atomic Configuration Management:** Utilizing temp-file-replace writes ensures file integrity and leverages filesystem `mtime/size` hooks for zero-overhead, cache-invalidation-free configuration lookups.
> - **Graceful Offline-First Degradation:** Diagnostic systems and model registries degrade sequentially (e.g., Live API → Local Cache → Manual Input) rather than failing outright.

---

## Part I — Foundations

## Chapter 1: What Hermes CLI Is (and What It Isn't)

The **Hermes CLI** (`hermes_cli/`) is the user-facing command-line interface for the Hermes Agent, an open-source agentic AI system built by Nous Research. At its surface it looks like a chat interface. Under the hood it is a multi-modal, multi-platform orchestration layer composed of roughly 80 Python modules and a full React/Ink terminal UI.

What Hermes provides:

| Surface | Implementation |
|:---------|:---------------|
| Interactive REPL | `prompt_toolkit` with autocomplete, history, and Rich formatting |
| Full-screen TUI | Ink (React) terminal UI with streaming, tool panels, session management |
| Setup & configuration wizard | Modular, section-re-runnable curses prompts |
| Gateway management | Long-running asyncio adapters for Telegram, Discord, Slack, WhatsApp, and 15+ other platforms |
| Session management | SQLite + FTS5 full-text search; resume by ID, title, or interactive browser |
| Skill & tool management | Install, enable, configure, and curate reusable agent capabilities |
| Plugin system | Third-party tools, hooks, slash commands, and CLI subcommands — no core changes needed |

What Hermes is **not**: it is not a thin wrapper around an LLM API. The interactive chat is a small fraction of the codebase. The majority is infrastructure — configuration, security, session persistence, multi-platform delivery, and extensibility machinery.

> **Key insight for architects:** The `cli.py` file alone is ~14,000 lines. `hermes_cli/main.py` is ~486 KB. This is not unusual for a mature, multi-surface agent harness. The complexity is earned by the breadth of concerns it handles correctly.

The architectural layers are:

```
User Terminal
    │
hermes_cli/main.py          ← entry point, bootstrap, dispatch
    │
    ├─ cli.py::HermesCLI    ← interactive REPL (~14k LOC)
    ├─ hermes_cli/setup.py  ← first-run wizard
    ├─ hermes_cli/main.py   ← subcommand handlers (cmd_model, cmd_status, …)
    └─ ui-tui/              ← Ink TUI subprocess
         │
         └─ tui_gateway/    ← Python JSON-RPC backend for the TUI
```

The remainder of this book traces each layer in depth, starting with the most fundamental: the filesystem.

---

## Chapter 2: The `~/.hermes/` Home: Your Agent's Filesystem

Before a single line of Python executes in a user context, Hermes needs a home. The `~/.hermes/` directory — or whatever path `HERMES_HOME` resolves to — is the agent's entire persistent state. Understanding it is a prerequisite for understanding everything else.

### 2.1 Directory Layout

The home directory is created by `ensure_hermes_home()` in `hermes_cli/config.py` on first run:

```
~/.hermes/
├── config.yaml           ← All behavioural configuration
├── .env                  ← API keys and secrets ONLY
├── SOUL.md               ← Base system prompt (editable)
├── state.db              ← SQLite: sessions, messages, FTS5 index
│
├── sessions/             ← Legacy session JSON (pre-SQLite, still read)
├── skills/               ← Agent-created and hub-installed skills
│   ├── .archive/         ← Curator-archived stale skills (recoverable)
│   └── .curator_backups/ ← Pre-run snapshots (tar.gz)
├── skins/                ← User YAML skin files
├── logs/
│   ├── agent.log         ← INFO+ (all agent activity)
│   ├── errors.log        ← WARNING+
│   ├── gateway.log       ← Gateway platform traffic
│   └── curator/          ← Per-run curator reports
├── cron/
│   └── jobs.json         ← Scheduled job definitions
├── hooks/                ← Shell-script event hooks
├── memories/             ← Persistent cross-session memory store
├── pairing/              ← Device-pairing state
├── image_cache/          ← Downloaded image cache
├── audio_cache/          ← TTS output cache
├── checkpoints/          ← Git-backed filesystem snapshots (/rollback)
├── lsp/bin/              ← Auto-installed LSP server binaries
├── profiles/             ← Named profiles, each a full clone of this layout
│   └── <name>/
└── active_profile        ← Sticky default profile name (single line)
```

### 2.2 Security Model

On POSIX systems, every file is `chmod 600` and every directory `chmod 700`. This is enforced by `_secure_file()` and `_secure_dir()` at creation time. Three opt-outs exist:

- `HERMES_SKIP_CHMOD=1` — skip all permission changes (Docker volume mounts)
- `HERMES_MANAGED=homebrew|nixos` — defer to the package manager's own permission model
- `HERMES_HOME_MODE=0701` — custom directory mode (e.g., to allow nginx traversal)

> **Warning:** Never hardcode `~/.hermes` in paths. Always call `get_hermes_home()` from `hermes_constants`. The profile system re-points `HERMES_HOME` before any module imports, so a module-level `Path.home() / ".hermes"` will silently ignore the active profile.

### 2.3 The Four Core Files

Every other file in `~/.hermes/` is supplementary. Four files govern everything meaningful:

**`config.yaml`** is the agent's brain — 30+ top-level sections controlling model selection, tool behaviour, compression, display, platform-specific settings, and more. Covered in depth in Chapter 13.

**`.env`** holds secrets exclusively: API keys, tokens, and passwords. Nothing else. The separation is intentional — config YAML gets committed to dotfiles repos; secrets must not follow. Covered in Chapter 14.

**`SOUL.md`** is the base system prompt, seeded from a default template on first run and freely editable by the user. It is the single file that defines the agent's persona and communication style. Covered in Chapter 14.

**`state.db`** is the SQLite session store. Every conversation, every message, every tool call is logged here, with a FTS5 full-text index for instant search. Covered in Chapter 14.

---

## Part II — The Boot Sequence

---

## Chapter 3: Six Phases from Shell to REPL

Typing `hermes chat` triggers a carefully ordered six-phase initialization. The order is not arbitrary — each phase depends on side-effects established by the previous one. Understanding it prevents a class of subtle bugs where environment setup arrives too late.

### Phase 1: Pre-Import Profile Override

The very first code that runs is `_apply_profile_override()`, called at **module level** in `hermes_cli/main.py` — not inside any function:

```python
# hermes_cli/main.py — runs at import time, before argparse, before logging
_apply_profile_override()
```

This function scans `sys.argv` for `--profile` / `-p`, sets `os.environ["HERMES_HOME"]` to the matching profile directory, then strips the flag so argparse never sees it. If no `--profile` flag is present, it falls back to the contents of `~/.hermes/active_profile`.

> **Why module level?** Python caches module-level constants at import time. Many modules call `get_hermes_home()` at the top of their file to compute default paths. If the profile override ran inside `main()` — after those modules were already imported — `HERMES_HOME` would be set too late to matter. Moving it to module level ensures the environment variable is in place before any other `hermes_cli` module is imported.

### Phase 2: Environment Bootstrap

After the profile is resolved, a fixed bootstrap sequence runs:

```python
import hermes_bootstrap          # Force UTF-8 stdio on Windows (replaces sys.stdout)
load_hermes_dotenv(project_env=PROJECT_ROOT / ".env")  # Load ~/.hermes/.env
_setup_logging(mode="cli")       # Rotating file handlers: agent.log + errors.log
if config.get("network", {}).get("force_ipv4"):
    _apply_ipv4(force=True)      # Monkey-patch socket.getaddrinfo to prefer IPv4
```

The `.env` load happens before logging so that any env-var-gated log levels take effect immediately. The `force_ipv4` option addresses a common Linux server problem where Python tries AAAA records first and stalls for the full TCP timeout on broken IPv6 stacks.

### Phase 3: Argparse Construction

The argument parser is built by `build_top_level_parser()` in `hermes_cli/_parser.py` — a **separate module** from `main.py`. This separation is deliberate: `relaunch.py` needs to introspect which flags carry `inherit_on_relaunch=True` without executing any of `main()`'s side-effects.

Two constraints worth noting:
- `--profile` / `-p` is absent from the parser definition — it was consumed pre-argparse
- Flags tagged `inherit_on_relaunch=True` are forwarded verbatim when the CLI re-execs itself (e.g., after `hermes update`)

### Phase 4: Subcommand Dispatch

After `args = parser.parse_args()`, `main()` dispatches to the appropriate `cmd_*` function:

```python
dispatch = {
    "chat":      cmd_chat,
    "setup":     cmd_setup,
    "model":     cmd_model,
    "status":    cmd_status,
    "doctor":    cmd_doctor,
    "gateway":   cmd_gateway,
    "profile":   cmd_profile,
    # … 14 more
}
fn = dispatch.get(args.command or "chat")
fn(args)
```

Each `cmd_*` function is a thin coordinator. Heavy logic lives in dedicated modules (`setup.py`, `models.py`, `status.py`, etc.) that are imported lazily inside the function — avoiding startup cost for commands that won't be used.

### Phase 5: Chat Pre-flight

`cmd_chat()` is the most complex dispatcher. Before handing control to the REPL, it runs a mandatory pre-flight sequence:

1. **Session resolution** — `--continue` / `-c` maps to the most recent session ID; `--resume <title>` fuzzy-matches past sessions by title
2. **First-run guard** — calls `_has_any_provider_configured()`; if no provider is set, prints guidance to run `hermes setup` and exits
3. **Background update check** — `prefetch_update_check()` runs in a daemon thread so it doesn't block startup
4. **Bundled skill sync** — `sync_skills(quiet=True)` checks for new bundled skills; fast, skips unchanged files
5. **Environment flags** — `--yolo`, `--ignore-user-config`, `--ignore-rules` are set as env vars *before* any config imports
6. **Fork** — `--tui` → `_launch_tui()`; default → lazy-import `cli.py::main()`

### Phase 6: TUI Launch

When `--tui` is specified, `_launch_tui()` takes over:

1. Verifies Node.js is available (offers auto-install via `node-bootstrap.sh`)
2. Checks whether `npm install` is needed by comparing **content** of `package-lock.json` against `node_modules/.package-lock.json` — not mtimes
3. Builds the bundle via esbuild (or uses `tsx` in `--dev` mode for watch-rebuild)
4. Passes all configuration as **environment variables** to the subprocess — avoiding shell-escaping issues and argument-length limits
5. Sets `NODE_OPTIONS=--max-old-space-size=8192 --expose-gc`
6. Calls `os.execvp()` to replace the Python process entirely with the Node.js process

The `os.execvp` choice is significant: it means the TUI process is a direct child of the shell, not a grandchild. Signals (Ctrl-C, SIGTERM) reach it directly without a Python intermediary that might swallow them.

---

## Part III — The Interactive Layer

---

## Chapter 4: HermesCLI and the REPL Loop

`HermesCLI`, defined at `cli.py:2502`, is the heart of the classic interactive interface. Its constructor (~200 lines) initializes five categories of state:

| Category | Key Attributes |
|:----------|:---------------|
| Display | `compact`, `streaming_enabled`, `show_reasoning`, `show_timestamps`, `final_response_markdown`, `inline_diffs` |
| Model | `model`, `provider`, `api_mode`, `base_url`, `api_key` |
| Agent | `max_turns` (90 default), `enabled_toolsets`, `disabled_toolsets` |
| Session | `session_id`, `checkpoints_enabled`, `_session_db` |
| UI | `console` (Rich), `_app` (prompt_toolkit Application), `_spinner` (KawaiiSpinner) |

### 4.1 The REPL Loop

`HermesCLI.run()` is a `while True` loop around four operations:

```
1. Render prompt (prompt_toolkit Application)
2. Get user input
3. Is it a slash command?
   ├── Yes → process_command(input)
   └── No  → agent.run_conversation(input)
4. Display result
```

While the agent is working, three things happen in parallel:
- The `KawaiiSpinner` animates in the status bar
- Tool results stream as `┊`-prefixed activity lines
- Reasoning tokens (if enabled) appear in fenced `[thinking]` blocks

### 4.2 process_command() Dispatcher

The slash command handler at `cli.py:7634` follows a strict pattern:

```python
def process_command(self, command: str) -> bool:
    cmd_lower = command.lower().strip()
    base_word = cmd_lower.split()[0].lstrip("/")
    cmd_def = resolve_command(base_word)           # → CommandDef or None
    canonical = cmd_def.name if cmd_def else base_word

    if canonical in {"quit", "exit"}:
        return False                               # signals REPL to exit
    elif canonical == "help":
        self.show_help()
    elif canonical == "model":
        self._handle_model_switch(cmd_original)
    # … ~80 more elif branches
```

Two key points: input is lowercased only for dispatch — the original case is preserved for argument parsing. And all dispatch goes through `resolve_command()`, which maps aliases to canonical names, meaning the `if/elif` chain only contains canonical names.

### 4.3 Agent Integration

When the input is not a slash command, it is sent to `AIAgent.run_conversation()`. The agent loop is fully synchronous (no asyncio in the CLI path), with interrupt detection on each iteration:

```python
while api_call_count < max_iterations and budget.remaining > 0:
    if interrupt_requested:
        break
    response = client.chat.completions.create(model=model, messages=messages, tools=schemas)
    if response.tool_calls:
        for call in response.tool_calls:
            result = handle_function_call(call.name, call.args, task_id)
            messages.append(tool_result_message(result))
        api_call_count += 1
    else:
        return response.content    # final text response
```

Messages follow the OpenAI wire format (`role: system/user/assistant/tool`). Reasoning content is stored in `assistant_msg["reasoning"]` and surfaced to the UI separately from the main content.

---

## Chapter 5: The Slash Command Registry

The command registry in `hermes_cli/commands.py` is one of the most instructive design choices in the codebase: a single list of `CommandDef` dataclasses serves as the authoritative definition of every slash command across every platform.

### 5.1 CommandDef

```python
@dataclass(frozen=True)
class CommandDef:
    name: str                              # canonical name: "background"
    description: str
    category: str                          # "Session" | "Configuration" | "Tools & Skills" | "Info" | "Exit"
    aliases: tuple[str, ...] = ()          # ("bg", "btw")
    args_hint: str = ""                    # "<prompt>", "[name]"
    subcommands: tuple[str, ...] = ()      # tab-completable sub-words
    cli_only: bool = False
    gateway_only: bool = False
    gateway_config_gate: str | None = None # dotpath into config.yaml
```

The `gateway_config_gate` field is worth highlighting. A command like `/verbose` is `cli_only=True` but can be exposed in the gateway if `display.tool_progress_command` is truthy in `config.yaml`. The command is always present in `GATEWAY_KNOWN_COMMANDS` (so dispatch never fails), but help text and Telegram/Slack menus only show it when the gate is open.

### 5.2 Downstream Derivations

At import time, `commands.py` rebuilds a set of derived mappings from the single `COMMAND_REGISTRY` list:

| Derived Mapping | Purpose |
|:-----------------|:---------|
| `_COMMAND_LOOKUP` | `{name/alias → CommandDef}` for fast dispatch |
| `COMMANDS` | Flat `{"/cmd" → description}` for backward compatibility |
| `COMMANDS_BY_CATEGORY` | `{category → {"/cmd" → description}}` for `show_help()` |
| `GATEWAY_KNOWN_COMMANDS` | `frozenset` of gateway-dispatchable names |
| `ACTIVE_SESSION_BYPASS_COMMANDS` | Commands that reach the agent mid-run |

And four platform-specific functions derived from the same registry:

| Platform | Function | Output |
|:----------|:----------|:--------|
| Telegram | `telegram_bot_commands()` | `[(name, desc)]` for `setMyCommands` API |
| Slack | `slack_native_slashes()` | `[(name, desc, usage)]` for app manifest |
| Slack subcommands | `slack_subcommand_map()` | `{subcommand → "/cmd"}` for `/hermes` dispatch |
| Gateway help | `gateway_help_lines()` | Formatted help text for all platforms |

Each sanitizes names per platform constraints: Telegram requires lowercase with underscores, max 32 characters; Slack forbids certain reserved words; Discord clamps to 32 characters with collision-aware suffix truncation.

> **The payoff:** Adding a new slash command requires exactly two steps — one `CommandDef` in `COMMAND_REGISTRY`, and one `elif` branch in `process_command()`. Help menus, Telegram menus, Slack manifests, autocomplete, and gateway dispatch update automatically. Adding an alias requires only editing the `aliases` tuple on the existing `CommandDef`.

---

## Chapter 6: The TUI — React Meets Python

The TUI is a full Node.js application using [Ink](https://github.com/vadimdemedes/ink) — React for the terminal — running as a subprocess of the Python CLI. This is a process-boundary architecture, not a widget library.

### 6.1 Process Model

```
hermes --tui
  └─ Node.js (Ink + React)           ← owns the screen
       │  newline-delimited JSON-RPC over stdio
       └─ Python (tui_gateway/entry.py)  ← owns state
            └─ AIAgent + tools + SessionDB
```

TypeScript handles rendering: the transcript, the composer, tool activity panels, approval prompts, and session pickers. Python handles everything stateful: the agent loop, tool execution, session persistence, and slash command dispatch.

### 6.2 The GatewayClient

`ui-tui/src/gatewayClient.ts` (701 lines) is the bridge. It spawns `python -m tui_gateway.entry` as a child process, writes JSON-RPC requests to its stdin, and reads responses from its stdout line-by-line:

```typescript
// GatewayClient.ts (simplified)
proc = spawn("python", ["-m", "tui_gateway.entry"], { env: { ...process.env } })
proc.stdout.on("data", chunk => {
    buffer += chunk.toString()
    for (const line of buffer.split("\n")) {
        const msg = JSON.parse(line)
        dispatch(msg.method, msg.params)
    }
})

send(method: string, params: object) {
    proc.stdin.write(JSON.stringify({ method, params }) + "\n")
}
```

The client supports an "attach mode" where instead of spawning a subprocess it connects to an external gateway via WebSocket — this is how the web dashboard embeds the TUI.

### 6.3 Key UI Surfaces

| Surface | Ink Component | JSON-RPC Flow |
|:---------|:--------------|:---------------|
| Chat streaming | `app.tsx` + `messageLine.tsx` | `prompt.submit` → `message.delta` → `message.complete` |
| Tool activity | `thinking.tsx` | `tool.start` → `tool.progress` → `tool.complete` |
| Approval prompts | `prompts.tsx` | `approval.request` ← user → `approval.respond` |
| Clarify / sudo / secret | `prompts.tsx`, `maskedPrompt.tsx` | `clarify.respond` |
| Session picker | `sessionPicker.tsx` | `session.list` → `session.resume` |
| Slash commands | Local handler + `slash.exec` fallthrough | `command.dispatch` |
| Theming | `theme.ts` + `branding.tsx` | `gateway.ready` event carries skin data |

### 6.4 The Custom Ink Fork

Hermes maintains `packages/hermes-ink/` — a fork of the upstream Ink library. This provides `onFrame` hooks for the activity spinner and `onHyperlinkClick` for clickable terminal links. The fork is a local package referenced in `package.json` as `"@hermes/ink": "file:./packages/hermes-ink"`.

### 6.5 Dashboard Integration

`hermes dashboard` serves a web UI that embeds the real `hermes --tui` — not a reimplementation. The browser loads `ChatPage.tsx`, which mounts an xterm.js `Terminal` with the WebGL renderer. The server (`hermes_cli/web_server.py`) spawns a PTY via `ptyprocess`, connects it to `hermes --tui`, and forwards bytes over a WebSocket authenticated with a session token.

> **Design rule enforced in `AGENTS.md`:** Never rebuild the primary chat surface in React for the dashboard. Anything new added to Ink shows up in the dashboard automatically. Structured React UI (sidebars, inspectors, model pickers) is allowed when it complements the embedded TUI rather than replacing the transcript or composer.

---

## Part IV — Core Subcommands

---

## Chapter 7: `hermes setup` — The First-Run Wizard

`hermes setup` is defined in `hermes_cli/setup.py` (3,559 lines) and is the canonical way to configure Hermes from scratch. Unlike a monolithic wizard that must be run end-to-end, each section can be invoked independently:

```bash
hermes setup            # full wizard (all sections)
hermes setup model      # re-run provider and model selection only
hermes setup terminal   # reconfigure execution backend only
hermes setup tools      # add or update tool API keys only
```

### 7.1 Wizard Architecture

All interactive prompts funnel through three wrappers — `prompt()`, `prompt_choice()`, and `prompt_checklist()` — each of which delegates to `curses_ui.py` for arrow-key navigation. When curses is unavailable (CI, piped output, non-TTY), these wrappers fall back to numbered plain-text input. The non-interactive detection is explicit:

```python
if not is_interactive_stdin():
    print("Run 'hermes config set <key> <value>' to configure headlessly.")
    sys.exit(0)
```

Two additional input hardening steps are worth noting. API keys pasted from the clipboard often carry terminal bracketed-paste escape sequences (`ESC[200~` / `ESC[201~`). The wizard strips these via `_BRACKETED_PASTE_PATTERN` before saving — a small detail that prevents mysterious authentication failures from invisible control characters.

### 7.2 Shared Code Path with `hermes model`

The setup wizard does not have its own provider selection logic. It calls `select_provider_and_model()` directly — the same function `cmd_model` invokes. This means there is exactly one code path for all provider configuration in the system, exercised by both the setup wizard and the standalone `hermes model` command. Changes to provider handling propagate to both surfaces automatically.

### 7.3 Post-Setup Health Summary

After the wizard completes, `_print_setup_summary()` iterates every tool category (vision, web, browser, image generation, TTS, Modal), calls the real runtime resolver for each, and renders a `✓ / ✗` health matrix. This is not a cosmetic recap — it invokes the same resolution code the CLI uses at startup, so it reflects the actual state the agent will see on next launch.

---

## Chapter 8: `hermes model` — Provider and Model Selection

The `hermes model` command surfaces one of the most interesting design challenges in the codebase: presenting a curated, user-friendly model picker that works offline, handles 15+ providers with different auth flows, and stays correct as the model landscape evolves weekly.

### 8.1 The Full Selection Flow

`cmd_model()` calls `select_provider_and_model()`, a 200+ line function that walks the user through six stages:

```
Stage 1: Load config.yaml → read current model.provider
Stage 2: Resolve effective provider
          ├── config.yaml model.provider
          ├── HERMES_INFERENCE_PROVIDER env var (override)
          └── resolve_provider("auto") → scans available API keys

Stage 3: Build provider menu
          ├── CANONICAL_PROVIDERS (static list from models.py)
          └── custom_providers from config.yaml (user-defined, env-ref preserved)

Stage 4: _prompt_provider_choice() → curses arrow-key picker

Stage 5: Provider-specific flow dispatch:
          ├── openrouter     → _model_flow_openrouter()
          ├── anthropic      → _model_flow_anthropic()
          ├── copilot        → _model_flow_copilot()         ← OAuth device code flow
          ├── google-gemini  → _model_flow_google_gemini_cli()
          ├── bedrock        → _model_flow_bedrock()         ← AWS credential validation
          ├── custom         → _model_flow_custom()          ← URL entry + test call
          └── api_key_type   → _model_flow_api_key_provider() ← generic key + list

Stage 6: Inside each flow:
          a. Prompt for API key if missing → save to ~/.hermes/.env
          b. Fetch models from provider /v1/models → live, authoritative list
             └── Falls back to _DEFAULT_PROVIDER_MODELS on timeout or failure
          c. Present model picker (curses radiolist)
          d. save_config() → atomic YAML write
```

### 8.2 Three-Tier Model List Resolution

The model picker never fails completely. It degrades gracefully through three tiers:

| Tier | Source | When Used |
|:------|:--------|:-----------|
| 1 | Live `/v1/models` API call | Network available, credentials valid |
| 2 | `_DEFAULT_PROVIDER_MODELS` static catalog | API timeout or network failure |
| 3 | Free-text input field | Provider not in static catalog |

This means a user on a plane can still run `hermes model` and type in a model name they know by heart. The system degrades to functional, not to broken.

### 8.3 Auxiliary Model Routing

Below the primary model picker sits a secondary menu — `_aux_config_menu()` — that routes nine background tasks to separate, independently configured providers:

| Task | Config Key | Typical Use |
|:------|:-----------|:-------------|
| Vision | `auxiliary.vision` | Screenshot and image analysis |
| Compression | `auxiliary.compression` | Context window summarization |
| Web extract | `auxiliary.web_extract` | Web page summarization |
| Session search | `auxiliary.session_search` | Past conversation recall |
| Approval | `auxiliary.approval` | Smart command risk assessment |
| MCP | `auxiliary.mcp` | MCP tool routing and reasoning |
| Title generation | `auxiliary.title_generation` | Automatic session naming |
| Skills hub | `auxiliary.skills_hub` | Skill search and install |
| Curator | `auxiliary.curator` | Background skill maintenance |

The pattern enables a split-model architecture: a powerful (expensive) model handles the primary conversation while cheap flash models handle compression, title generation, and session search. A concrete example:

```yaml
# config.yaml: route compression to a cheap model, use main model for everything else
auxiliary:
  compression:
    provider: openrouter
    model: google/gemini-3-flash-preview
```

### 8.4 Stale Endpoint Cleanup

After every provider switch, `_clear_stale_openai_base_url()` removes any `OPENAI_BASE_URL` entry from `~/.hermes/.env`. This addresses a real user-facing issue (#5161): users who configure a custom endpoint, then switch to a named provider, were left with a dangling `OPENAI_BASE_URL` that silently overrode the new provider's URL. The cleanup runs automatically on every `hermes model` invocation.

---

## Chapter 9: `hermes status` and `hermes doctor` — Diagnostics

Hermes ships two diagnostic commands with deliberately different scopes. `hermes status` is a fast, read-only health snapshot. `hermes doctor` is a deep scan with auto-fix capabilities.

### 9.1 `hermes status`

Defined in `hermes_cli/status.py` (550 lines), `status` is designed to be lightweight — no LLM calls, no heavy imports. It checks:

```
◆ Environment       — project root, Python version, .env presence, active model
◆ API Keys          — 18+ providers, displayed redacted via mask_secret()
◆ Auth Providers    — Nous, Codex, Gemini, MiniMax (OAuth token validity + expiry)
◆ Terminal Backend  — local / docker / ssh / daytona / vercel_sandbox + backend details
◆ Messaging         — 15 platforms: Telegram, Discord, Slack, WhatsApp, Signal, …
◆ Gateway Service   — running PIDs, service manager (systemd / launchd / manual)
◆ Scheduled Jobs    — count from ~/.hermes/cron/jobs.json
◆ Sessions          — total session count from SessionDB
```

The `--deep` flag adds live network probes: tests the OpenRouter `/models` endpoint and checks whether the gateway port (18789) is reachable. `--all` shows unredacted API key values.

The critical design choice: `_effective_provider_label()` resolves the provider through `resolve_requested_provider()` → `resolve_provider()` — the exact same code path the CLI uses at startup. The status output shows what the agent *will* use, not just what is configured.

### 9.2 `hermes doctor`

Defined in `hermes_cli/doctor.py` (1,836 lines), doctor runs ordered check categories and collects results into two lists: auto-fixable issues and manual-only issues.

**Check ordering is intentional — security first:**

```
1. Security Advisories    ← FIRST: detect compromised packages (litellm worm, etc.)
2. Python Environment     ← version check, venv detection
3. Required Packages      ← openai, rich, yaml, httpx
4. Configuration Files    ← .env existence, config.yaml presence, version migration
5. Config Structure       ← validate_config_structure() for malformed custom_providers
6. Auth Providers         ← OAuth token status for each configured provider
7. Provider Connectivity  ← live HTTP /models checks
8. System Dependencies    ← Node.js, ripgrep, ffmpeg, browser binaries
9. Tool Availability      ← per-toolset check_fn() results
10. Gateway Service       ← systemd linger check on Linux
```

> **Design principle:** The security advisory check runs first because a compromised package is more urgent than a missing config key. When writing diagnostic pipelines, order checks by user impact, not by implementation convenience.

**Auto-fix mode** (`hermes doctor --fix`) handles three classes of repair: creating a missing `~/.hermes/.env`, running `migrate_config()` for outdated schema versions, and promoting stale root-level `provider`/`base_url` keys into the correct `model:` section.

**Plugin-augmented discovery:** `_build_apikey_providers_list()` starts with a static list, then extends it by iterating `list_providers()` from the plugin registry. Any provider plugin dropped into `~/.hermes/plugins/` automatically appears in `hermes doctor` health checks without touching core files.

---

## Chapter 10: `hermes auth`, `config`, `gateway`, and `profile`

### 10.1 `hermes auth` — Credential Pools

The auth command manages multi-key pools for rate-limit avoidance. The storage convention is straightforward: numbered environment variables per provider.

```bash
# ~/.hermes/.env
OPENROUTER_API_KEY=sk-or-primary
OPENROUTER_API_KEY_2=sk-or-secondary
OPENROUTER_API_KEY_3=sk-or-tertiary
```

`AIAgent` reads the pool at startup and cycles through keys transparently on `429 RateLimitError`. The `hermes auth` subcommands manage this pool:

```bash
hermes auth add openrouter --key sk-or-...   # append to pool
hermes auth list                             # show pool (keys masked)
hermes auth remove openrouter 1              # remove by index
hermes auth test openrouter                  # verify all keys with a live call
```

### 10.2 `hermes config` — Configuration Editor

The config command is a thin delegator to `config_command()` in `hermes_cli/config.py`. It supports dotpath-notation access and atomic writes:

```bash
hermes config                              # display full config (redacted secrets)
hermes config get model.provider           # read a single key
hermes config set agent.max_turns 120      # write a key (atomic YAML replace)
hermes config edit                         # open $EDITOR on config.yaml directly
hermes config reset                        # restore all defaults (with confirmation)
```

The `set` operation navigates nested YAML using dotpath notation, then writes through `atomic_yaml_write()` — a temp-file-then-rename pattern that prevents partial-write corruption and naturally invalidates the in-process config cache by producing a fresh inode.

### 10.3 `hermes gateway` — Messaging Platform Manager

The gateway is a long-running asyncio process running platform adapters in parallel. `hermes_cli/gateway.py` provides the OS-level management wrapper:

```bash
hermes gateway            # start in foreground
hermes gateway install    # register as systemd unit (Linux) or launchd plist (macOS)
hermes gateway status     # show running PIDs and service state
hermes gateway stop       # graceful shutdown (drains in-flight agent runs)
hermes gateway logs       # tail gateway.log
```

Platform adapters are discovered via `gateway/platform_registry.py`. Plugins register new platforms by calling `platform_registry.register(...)` in their `register(ctx)` function — no changes to gateway core required.

The graceful shutdown flow respects `agent.restart_drain_timeout` (default 180s): the gateway stops accepting new work, waits for in-flight agent runs to complete, then sends SIGTERM to any remaining runs after the timeout. This prevents cutting off a mid-reasoning model call that has been running for two minutes.

### 10.4 `hermes profile` — Multi-Instance Isolation

Each profile is a complete, self-contained `~/.hermes/`-equivalent directory:

```bash
hermes profile create work    # creates ~/.hermes/profiles/work/
hermes profile list           # shows all profiles + active indicator
hermes profile use work       # writes "work" to ~/.hermes/active_profile (sticky)
hermes profile delete work    # removes profile directory
hermes profile export work    # zip profile for backup or transfer
```

Using a profile for a single invocation without changing the sticky default: `hermes -p work chat`. The flag is consumed before argparse runs, so all imports inside `cmd_chat()` see the correct `HERMES_HOME` from the moment they load.

---

## Part V — Extensibility

---

## Chapter 11: The Plugin System

The plugin system in `hermes_cli/plugins.py` (1,561 lines) is the primary extension point for Hermes. It allows third parties to register tools, hooks, slash commands, and CLI subcommands without modifying any core file.

### 11.1 Discovery Sources

Plugins are discovered from four locations in priority order:

| Source | Path | Priority |
|:--------|:------|:----------|
| Pip-installed | Entry point `hermes_agent.plugins` | Highest (last writer wins) |
| Project-local | `./.hermes/plugins/<name>/` | Medium (requires `HERMES_ENABLE_PROJECT_PLUGINS=1`) |
| User-installed | `~/.hermes/plugins/<name>/` | Medium |
| Bundled | `<repo>/plugins/<name>/` | Lowest |

Discovery is triggered as a side-effect of importing `model_tools.py`. Code paths that need plugin state without going through `model_tools` must call `discover_plugins()` explicitly — it is idempotent.

### 11.2 Plugin Manifest and Entry Point

Every plugin requires two files:

```
~/.hermes/plugins/my-plugin/
├── plugin.yaml     ← manifest
└── __init__.py     ← must define register(ctx)
```

Minimal `plugin.yaml`:

```yaml
name: my-plugin
description: "A custom tool plugin"
version: "1.0"
kind: tool   # tool | memory | context_engine | model-provider | platform
```

The `register(ctx)` function receives a `PluginContext`:

```python
def register(ctx: PluginContext) -> None:
    ctx.register_tool(
        name="my_tool",
        toolset="my-plugin",
        schema={"name": "my_tool", "description": "…", "parameters": {…}},
        handler=lambda args, **kw: json.dumps({"result": do_work(args)}),
    )
    ctx.register_hook("post_tool_call", my_audit_hook)
    ctx.register_command("mycommand", "Does X", args_hint="<arg>", handler=my_handler)
```

### 11.3 Valid Hook Points

The hook system covers the entire agent lifecycle:

```python
VALID_HOOKS = {
    "pre_tool_call", "post_tool_call",
    "transform_terminal_output", "transform_tool_result", "transform_llm_output",
    "pre_llm_call", "post_llm_call",
    "pre_api_request", "post_api_request",
    "on_session_start", "on_session_end", "on_session_finalize", "on_session_reset",
    "subagent_stop",
    "pre_gateway_dispatch",
    "pre_approval_request", "post_approval_response",
}
```

> **Non-negotiable rule from `AGENTS.md`:** Plugins must not modify core files. If a plugin needs a capability the framework does not expose, the correct fix is to expand the generic plugin surface — never add plugin-specific logic to `run_agent.py`, `cli.py`, or `gateway/run.py`.

### 11.4 Shell-Script Hooks

For simpler integration needs, `config.yaml` supports declarative shell-script hooks that do not require a Python plugin:

```yaml
hooks:
  post_tool_call:
    - matcher: "terminal"           # tool name glob pattern
      command: "~/.hermes/hooks/audit.sh"
      timeout: 5
  pre_llm_call:
    - matcher: "*"
      command: "~/.hermes/hooks/log-call.sh"
      timeout: 2
```

First registration triggers a one-time consent prompt stored in `~/.hermes/shell-hooks-allowlist.json`. Set `hooks_auto_accept: true` for headless gateway or cron deployments.

---

## Chapter 12: Skins and Themes

The skin engine in `hermes_cli/skin_engine.py` (921 lines) provides data-driven visual customization. No code changes are required to add a new skin — only data.

### 12.1 What a Skin Controls

```
SkinConfig
├── colors             ← banner border, title, accent, response box
├── spinner
│   ├── waiting_faces  ← idle animation frames
│   ├── thinking_faces ← active animation frames
│   ├── thinking_verbs ← "thinking", "reasoning", "computing"
│   └── wings          ← optional left/right wing pairs: ["⟨⚡", "⚡⟩"]
├── branding
│   ├── agent_name     ← displayed in banner
│   ├── welcome        ← banner subtitle
│   ├── response_label ← label on the response box
│   └── prompt_symbol  ← "❯ " or "⚡ " before user input
├── tool_prefix        ← character before tool output lines (default "┊")
└── tool_emojis        ← {tool_name → emoji} mapping
```

### 12.2 Built-in Skins

| Skin | Character |
|:------|:-----------|
| `default` | Classic Hermes gold and kawaii |
| `ares` | War-god — crimson and bronze |
| `mono` | Clean grayscale |
| `slate` | Cool blue, developer-focused |
| `daylight` | Light-background-friendly |
| `poseidon` | Ocean-god — deep blue and seafoam |
| `charizard` | Volcanic — burnt orange and ember |

### 12.3 User Skins

Create `~/.hermes/skins/<name>.yaml` with any subset of the skin schema. Missing fields fall back to the `default` skin — no error for partial definitions:

```yaml
name: cyberpunk
description: Neon-soaked terminal theme
colors:
  banner_border: "#FF00FF"
  banner_title: "#00FFFF"
spinner:
  thinking_verbs: ["jacking in", "decrypting", "uplink established"]
  wings:
    - ["⟨⚡", "⚡⟩"]
branding:
  agent_name: "Cyber Agent"
  prompt_symbol: "⚡ "
tool_prefix: "▏"
```

Activate at runtime with `/skin cyberpunk` or permanently with `hermes config set display.skin cyberpunk`.


---

## Part VI — Configuration Reference

---

## Chapter 13: `config.yaml` — The Agent's Brain

`~/.hermes/config.yaml` is the single most important file in the Hermes ecosystem. Every CLI mode loads it. Every tool reads from it. Every platform adapter interprets it. The file is deep-merged with `DEFAULT_CONFIG` at load time, so a user's file only needs to contain overrides — a minimal two-line `config.yaml` is perfectly valid.

### 13.1 Version Tracking and Migration

The top-level key `_config_version: 23` (as of May 2026) enables schema migration. On every startup, `migrate_config()` compares the user's version to `DEFAULT_CONFIG`'s version and runs any needed migration functions. Bumping the version is only required for structural changes — renaming keys, moving sections, changing value types. Adding a new key to an existing section does not require a version bump; the deep merge handles it transparently.

### 13.2 Top-Level Sections Reference

| Section | Key Default | What It Controls |
|:---------|:------------|:-----------------|
| `model` | `""` | Active provider slug + model name |
| `providers` | `{}` | Named custom provider endpoints |
| `fallback_providers` | `[]` | Ordered failover chain |
| `credential_pool_strategies` | `{}` | Per-provider key-cycling |
| `toolsets` | `["hermes-cli"]` | Active toolset bundle names |
| `agent` | (see below) | Core loop parameters |
| `terminal` | (see below) | Execution backend |
| `web` | `backend: ""` | Search + extract routing |
| `browser` | `inactivity_timeout: 120` | Browser engine, CDP, sessions |
| `checkpoints` | `enabled: false` | Git-backed snapshots |
| `compression` | `enabled: true, threshold: 0.50` | Context window compression |
| `prompt_caching` | `cache_ttl: "5m"` | Anthropic prompt cache TTL |
| `openrouter` | `response_cache: true` | OpenRouter response caching |
| `bedrock` | `region: ""` | AWS Bedrock + guardrails |
| `auxiliary` | (9 tasks) | Per-task auxiliary model routing |
| `display` | (see below) | UI/UX appearance |
| `dashboard` | `theme: "default"` | Web dashboard theme |
| `privacy` | `redact_pii: false` | PII redaction in LLM context |
| `tts` | `provider: "edge"` | Text-to-speech |
| `stt` | `provider: "local"` | Speech-to-text |
| `voice` | `record_key: "ctrl+b"` | Push-to-talk settings |
| `memory` | `memory_enabled: true` | Persistent memory injection |
| `context` | `engine: "compressor"` | Context window manager plugin |
| `delegation` | `max_iterations: 50` | Subagent routing + concurrency |
| `goals` | `max_turns: 20` | Persistent goal loop budget |
| `skills` | `external_dirs: []` | Skill directories + template vars |
| `curator` | `interval_hours: 168` | Background skill maintenance |
| `approvals` | `mode: "manual"` | Command approval gate |
| `security` | `tirith_enabled: true` | Pre-exec scanning + advisories |
| `slack` / `discord` / `telegram` / … | platform-specific | Per-platform gateway behaviour |
| `cron` | `wrap_response: true` | Scheduled task delivery |
| `kanban` | `dispatch_interval_seconds: 60` | Multi-agent board |
| `lsp` | `enabled: true` | Language server integration |
| `logging` | `level: "INFO"` | Log rotation + retention |
| `sessions` | `auto_prune: false` | state.db retention + VACUUM |
| `model_catalog` | `ttl_hours: 24` | Remote model list cache |
| `_config_version` | `23` | Migration marker (do not edit) |

### 13.3 Critical Sub-Sections

**`agent` — the loop contract:**

```yaml
agent:
  max_turns: 90                   # Tool-calling iteration cap per conversation
  gateway_timeout: 1800           # Idle timeout for gateway runs (seconds)
  restart_drain_timeout: 180      # Graceful drain before hard kill on /restart
  api_max_retries: 3              # App-level retry loop (above the SDK's own)
  tool_use_enforcement: "auto"    # Force tool calls on models that prefer prose
  clarify_timeout: 600            # Max wait for user response to clarify prompt
  gateway_notify_interval: 180    # "Still working" keepalive interval
  image_input_mode: "auto"        # native | text — how images reach the model
  disabled_toolsets: []           # Global toolset disable list
```

**`terminal` — the execution harness:**

```yaml
terminal:
  backend: "local"                # local | docker | ssh | daytona | modal | vercel_sandbox
  cwd: "."                        # Working directory for gateway mode
  timeout: 180                    # Per-command execution timeout (seconds)
  docker_image: "nikolaik/python-nodejs:python3.11-nodejs20"
  persistent_shell: true          # Reuse a long-lived bash across execute() calls
  container_memory: 5120          # MB — container memory limit (5 GB default)
  container_disk: 51200           # MB — container disk limit (50 GB default)
  docker_volumes: []              # ["host_path:container_path"] mount pairs
```

**`display` — the UI contract:**

```yaml
display:
  skin: "default"                 # Skin name (built-in or ~/.hermes/skins/<name>.yaml)
  language: "en"                  # UI language for static strings (en/zh/ja/de/…)
  streaming: false                # Token-by-token streaming in the CLI
  show_reasoning: false           # Show model reasoning tokens
  final_response_markdown: "strip" # render | strip | raw
  inline_diffs: true              # Show diffs for write_file/patch operations
  show_cost: false                # Show $ cost in status bar
  tool_progress_command: false    # Enable /verbose command in the gateway
  tui_status_indicator: "kaomoji" # kaomoji | emoji | unicode | ascii
```

**`tool_output` — context budget guardrails:**

```yaml
tool_output:
  max_bytes: 50000      # Terminal output cap (~12-15K tokens)
  max_lines: 2000       # read_file pagination cap
  max_line_length: 2000 # Per-line character cap in line-numbered view
```

**`tool_loop_guardrails` — preventing runaway loops:**

```yaml
tool_loop_guardrails:
  warnings_enabled: true
  hard_stop_enabled: false    # Opt-in: terminates the loop on repeated failure
  warn_after:
    exact_failure: 2
    same_tool_failure: 3
    idempotent_no_progress: 2
  hard_stop_after:
    exact_failure: 5
    same_tool_failure: 8
    idempotent_no_progress: 5
```

### 13.4 The Three Config Loaders

Hermes uses three distinct loading paths depending on the execution context. Mixing them up is a common source of "why does the gateway not see my config change?" bugs.

| Loader | Used By | Behaviour |
|:--------|:---------|:-----------|
| `load_cli_config()` | `cli.py` interactive REPL | Merges CLI defaults + `load_config()` output |
| `load_config()` | All subcommands | Deep-merges `DEFAULT_CONFIG` → `config.yaml` |
| Direct YAML | `gateway/run.py` | Reads raw YAML, no `DEFAULT_CONFIG` merge |

All three are protected against concurrent access by `_CONFIG_LOCK` (an `RLock`). `load_config()` additionally caches results keyed by `(mtime_ns, file_size)` — a cache hit costs ~13µs (a `deepcopy`); a miss costs ~13ms (YAML parse + deep merge). The `atomic_yaml_write()` function produces a fresh inode on every write, which naturally invalidates this cache without any explicit invalidation call.

> **Minimal valid config.yaml:** Because of the deep merge, the user's file only needs to express divergence from defaults. A file containing only the model settings is complete:
> ```yaml
> model:
>   provider: openrouter
>   default: google/gemini-3-flash-preview
> ```

---

## Chapter 14: `.env`, `SOUL.md`, and `state.db`

### 14.1 `.env` — Secrets Only

`~/.hermes/.env` holds API keys, tokens, and passwords. Non-secret settings — timeouts, flags, paths, thresholds — always go in `config.yaml`. The separation is not aesthetic: the `.env` file receives an independent `chmod 600`, and `hermes config` output redacts it. A `config.yaml` committed to a dotfiles repository will never expose secrets if this boundary is respected.

**Loading order:** `get_env_value(key)` checks `os.environ` first (for runtime overrides via the shell), then falls back to `~/.hermes/.env`. The loader strips surrounding whitespace and terminal bracketed-paste control sequences automatically.

**Credential pool numbering:**

```bash
OPENROUTER_API_KEY=sk-or-v1-primary
OPENROUTER_API_KEY_2=sk-or-v1-secondary
OPENROUTER_API_KEY_3=sk-or-v1-tertiary
```

`AIAgent` reads the pool at startup and rotates through keys on `429 RateLimitError` with no user intervention.

**Key categories:**

| Category | Examples |
|:----------|:---------|
| Primary providers | `OPENROUTER_API_KEY`, `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GOOGLE_API_KEY`, `XAI_API_KEY`, `NVIDIA_API_KEY`, `KIMI_API_KEY` |
| Provider base URLs | `LM_BASE_URL`, `GEMINI_BASE_URL`, `XAI_BASE_URL` |
| Tool services | `FIRECRAWL_API_KEY`, `TAVILY_API_KEY`, `BROWSERBASE_API_KEY`, `FAL_KEY`, `ELEVENLABS_API_KEY` |
| Messaging platforms | `TELEGRAM_BOT_TOKEN`, `DISCORD_BOT_TOKEN`, `SLACK_BOT_TOKEN`, `GITHUB_TOKEN` |
| System | `SUDO_PASSWORD`, `HERMES_HOME`, `HERMES_PYTHON` |

### 14.2 `SOUL.md` — The Agent's Identity

Seeded on first run from `hermes_cli/default_soul.py`, `SOUL.md` is the base system prompt — the text that defines who the agent is before any skill, memory, or session context is added. The default content:

```
You are Hermes Agent, an intelligent AI assistant created by Nous Research.
You are helpful, knowledgeable, and direct. You assist users with a wide
range of tasks including answering questions, writing and editing code,
analyzing information, creative work, and executing actions via your tools.
You communicate clearly, admit uncertainty when appropriate, and prioritize
being genuinely useful over being verbose unless otherwise directed below.
Be targeted and efficient in your exploration and investigations.
```

Users edit this file directly to change persona, tone, domain focus, or output constraints. The system prompt the model sees is assembled in layers:

```
Layer 1: SOUL.md                    ← Identity (always present)
Layer 2: Active skill SKILL.md      ← Skill-specific instructions (optional)
Layer 3: Persistent memory          ← From ~/.hermes/memories/ (~800-token budget)
Layer 4: User profile memory        ← From memory provider (~500-token budget)
Layer 5: Session context files      ← .context files in the working directory
```

### 14.3 `state.db` — The Session Store

`~/.hermes/state.db` is a SQLite database managed by `hermes_state.py::SessionDB`. It is the permanent record of every conversation Hermes has ever had on this system.

| Table | Contents |
|:-------|:----------|
| `sessions` | Session metadata: id, title, platform, created_at, ended_at |
| `messages` | All conversation messages: role, content, tool_calls, reasoning |
| `tool_calls` | Structured tool log: tool_name, args, result, duration_ms |
| `state_meta` | Key-value store: pruning timestamps, onboarding state |
| FTS5 virtual table | Full-text search index over session titles and message content |

The FTS5 virtual table powers `hermes sessions search "my query"` and the `/search` slash command — sub-millisecond full-text search across tens of thousands of messages.

Session pruning is opt-in, governed by `sessions.auto_prune` and `sessions.retention_days` in `config.yaml`. Heavy gateway deployments (continuous messaging across Telegram, Discord, etc.) should enable `auto_prune: true` with a sensible `retention_days` — users have reported 384MB+ databases with 68K+ messages when pruning is disabled.

### 14.4 `cron/jobs.json` — Scheduled Tasks

```json
{
  "jobs": [
    {
      "id": "daily-standup",
      "enabled": true,
      "schedule": "0 9 * * 1-5",
      "task": "Generate a standup summary and post it to #engineering",
      "delivery": "slack",
      "model": "",
      "max_turns": 10
    }
  ]
}
```

The gateway reads this file at startup and on `SIGHUP`. The dispatcher evaluates due jobs on the `kanban.dispatch_interval_seconds` tick (default 60s) and spawns `AIAgent` instances in threads, with the `task` string as the opening user message.

### 14.5 `skills/` — Reusable Agent Capabilities

```
skills/my-skill/
├── SKILL.md        ← Instructions injected into system prompt when active
├── plugin.yaml     ← name, description, version, requires
└── *.py / *.sh     ← Optional implementation files
```

Minimal `plugin.yaml`:

```yaml
name: my-skill
description: Does something useful
version: "1.0"
requires:
  executables: [ripgrep]
  python: [httpx]
```

The curator — running on the configurable `curator.interval_hours` schedule — reviews agent-created skills, marks unused ones as stale after `stale_after_days`, and archives them to `skills/.archive/` after `archive_after_days`. Archives are recoverable; nothing is deleted automatically.


---

## Part VII — Staff-Engineer Patterns

---

## Chapter 15: Ten Design Patterns Worth Stealing

The Hermes codebase is a working example of a production-grade Python CLI under sustained development. Ten patterns appear repeatedly and are worth understanding as design primitives for any similar system.

### Pattern 1: Data-Driven Command Registries

`COMMAND_REGISTRY` in `commands.py` is a list of `CommandDef` dataclasses — the sole definition of every slash command in the system. Every downstream consumer derives from it automatically at import time: CLI dispatch, help menus, Telegram bot menus, Slack app manifests, autocomplete, and gateway routing.

```python
@dataclass(frozen=True)
class CommandDef:
    name: str
    description: str
    category: str
    aliases: tuple[str, ...] = ()
    args_hint: str = ""
    cli_only: bool = False
    gateway_config_gate: str | None = None
```

The `if/elif` dispatch chain in `process_command()` is the *only* place containing implementation logic. It stays minimal because the registry handles all metadata. Adding a command requires one `CommandDef` entry and one `elif` branch. Adding an alias requires editing only the `aliases` tuple.

**Applicable when:** you have a command surface that needs to be consistent across multiple delivery channels (CLI, API, messaging bots, help text, autocomplete).

### Pattern 2: Pre-Import Environment Bootstrapping

Python caches module-level expressions at import time. Setting `os.environ` inside `main()` — after other modules have already been imported — misses any values those modules cached at the module level.

Hermes solves this by calling `_apply_profile_override()` at module level in `main.py`:

```python
# hermes_cli/main.py — module level, NOT inside a function
_apply_profile_override()   # sets HERMES_HOME before any other import
```

This scans `sys.argv` for `--profile`, mutates the environment, and strips the flag before argparse sees it. All subsequent `get_hermes_home()` calls in any module return the correct value regardless of import order.

**Applicable when:** you have an environment variable that must be set before any module-level code that reads it executes (profile systems, workspace isolation, multi-tenant CLIs).

### Pattern 3: TTY Guard on Every Interactive Command

Every command that calls `input()` begins with a TTY check:

```python
def _require_tty(cmd: str) -> None:
    if not sys.stdout.isatty():
        print(f"Error: '{cmd}' requires an interactive terminal.", file=sys.stderr)
        sys.exit(1)
```

Non-interactive flows expose `hermes config set` as the programmatic alternative. This prevents wizard prompts from hanging indefinitely in CI pipelines, cron jobs, or piped output.

**Applicable when:** any command that uses interactive prompts. The guard should be the first line of the function, not buried after setup.

### Pattern 4: Decouple Parser from Executor

`build_top_level_parser()` lives in `hermes_cli/_parser.py`, not `main.py`. This separation allows `relaunch.py` to introspect flag metadata — specifically which flags carry `inherit_on_relaunch=True` — without executing any of `main()`'s initialization side-effects.

**Applicable when:** you have code that needs to reflect on the CLI's argument schema (re-exec with inherited flags, documentation generation, shell completion generation) without triggering side-effects.

### Pattern 5: Atomic Config Writes with Implicit Cache Invalidation

```python
def atomic_yaml_write(path: Path, data: dict) -> None:
    tmp = path.with_suffix(".yaml.tmp")
    tmp.write_text(yaml.dump(data), encoding="utf-8")
    tmp.replace(path)   # atomic on POSIX; near-atomic on Windows
```

Two invariants are maintained simultaneously. First, a crash mid-write leaves the old file intact — there is never a moment where `path` contains partial content. Second, `os.replace()` produces a new inode with updated `mtime`, which naturally invalidates the `(mtime_ns, file_size)` cache key in `load_config()` without any explicit cache invalidation call.

**Applicable when:** any file that is both written by one code path and cached by another. The temp-rename pattern gives you crash safety and cache invalidation for free.

### Pattern 6: Three-Tier Graceful Degradation for External Data

The model picker degrades through three tiers so it is never completely broken:

```
Tier 1: Live /v1/models API call    → network available, authoritative
  ↓  (timeout or failure)
Tier 2: Static offline catalog      → _DEFAULT_PROVIDER_MODELS dict
  ↓  (provider not in catalog)
Tier 3: Free-text input             → user types any model name
```

**Applicable when:** any feature that depends on external data (model lists, registry lookups, remote configs). Design for degraded-but-functional rather than all-or-nothing. The user should always be able to proceed, even if the experience is less convenient.

### Pattern 7: Static Base + Plugin Extension

`_build_apikey_providers_list()` in `doctor.py` starts with a static list, then extends it by iterating `list_providers()` from the plugin registry:

```python
for provider in list_providers():
    if provider.auth_type != "api_key" or provider.name in known_canonical:
        continue
    static_list.append((provider.display_name, provider.env_vars, …))
```

`hermes doctor` automatically includes any provider added via a plugin. The pattern — **static base + open plugin extension** — is applicable to any registry that needs to work out of the box without configuration but also be extensible without touching core code.

### Pattern 8: Security-First Diagnostic Ordering

`hermes doctor` runs security advisory checks first, before any other diagnostic category. This is a deliberate editorial choice: ordering communicates priority. A compromised dependency is more urgent than a missing config key, and a user who scrolls past content sees the most important issue first.

**Applicable when:** writing any diagnostic or health-check pipeline. Order checks by user impact, not by implementation convenience or alphabetical name.

### Pattern 9: Process Boundary Escalation

The TUI is a separate Node.js process, not a Python widget library. Configuration crosses the process boundary via environment variables (not CLI args — which have length limits and escaping issues). The Python process becomes a pure JSON-RPC backend.

This pattern is broadly applicable: when a subsystem has fundamentally different runtime requirements from its host — React's reconciler versus Python's asyncio, for example — a narrow RPC interface over a process boundary is cleaner than forcing both into a single process and managing their event loops.

### Pattern 10: Content-Based Cache Invalidation

`_tui_need_npm_install()` compares `package-lock.json` content against `node_modules/.package-lock.json`, not file mtimes. Git checkouts update mtimes without changing content, which would trigger spurious `npm install` runs on every `git pull`.

```python
def _tui_need_npm_install() -> bool:
    lockfile = TUI_DIR / "package-lock.json"
    installed = TUI_DIR / "node_modules" / ".package-lock.json"
    if not installed.exists():
        return True
    return lockfile.read_bytes() != installed.read_bytes()
```

**Rule:** use content equality, not timestamps, as the cache key for version-controlled files. Timestamps are an unreliable proxy for content in any system where files can be restored from version control.

---

## Appendix: Module Reference

| Module | Location | Primary Responsibility |
|:--------|:----------|:----------------------|
| Entry point | `hermes_cli/main.py` | Profile override, bootstrap, argparse dispatch, all `cmd_*` handlers |
| Interactive REPL | `cli.py` | `HermesCLI` class (~14k LOC) |
| Argument parser | `hermes_cli/_parser.py` | `build_top_level_parser()` |
| Command registry | `hermes_cli/commands.py` | `COMMAND_REGISTRY`, `CommandDef`, platform derivations |
| Configuration | `hermes_cli/config.py` | `DEFAULT_CONFIG`, `load_config()`, `atomic_yaml_write()`, migration |
| Setup wizard | `hermes_cli/setup.py` | `run_setup_wizard()`, modular sections, curses prompts |
| Model selection | `hermes_cli/main.py` | `select_provider_and_model()`, `_model_flow_*()` |
| Status | `hermes_cli/status.py` | `show_status()`, component health matrix |
| Doctor | `hermes_cli/doctor.py` | `run_doctor()`, `_build_apikey_providers_list()` |
| Auth | `hermes_cli/auth_commands.py` | Credential pool management |
| Gateway wrapper | `hermes_cli/gateway.py` | OS-level service management (systemd/launchd) |
| Profiles | `hermes_cli/profiles.py` | Profile CRUD, `_apply_profile_override()` |
| Plugin system | `hermes_cli/plugins.py` | `PluginManager`, `PluginContext`, discovery |
| Skin engine | `hermes_cli/skin_engine.py` | `SkinConfig`, built-in skins, YAML loader |
| Session store | `hermes_state.py` | `SessionDB`, SQLite + FTS5 |
| Agent core | `run_agent.py` | `AIAgent`, conversation loop (~12k LOC) |
| Tool registry | `tools/registry.py` | `registry.register()`, schema collection, dispatch |
| Tool orchestration | `model_tools.py` | `discover_builtin_tools()`, `handle_function_call()` |
| TUI entry | `ui-tui/src/entry.tsx` | Node.js bootstrap, `GatewayClient` init, `ink.render()` |
| TUI gateway bridge | `ui-tui/src/gatewayClient.ts` | JSON-RPC client (701 lines) |
| TUI Python backend | `tui_gateway/entry.py` | JSON-RPC server, `AIAgent` + sessions |
| Dashboard server | `hermes_cli/web_server.py` | FastAPI + PTY bridge + WebSocket |

---

*Hermes Agent — May 2026*
