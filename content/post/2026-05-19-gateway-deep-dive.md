---
layout: post
title: "Building a Multi-Platform Agentic Gateway: A Deep Dive into Hermes Gateway"
date: "2026-05-19"
author: "Jamie Zhang"
tags: ["Hermes", "Gateway", "Agentic AI", "Messaging Platforms", "asyncio"]
categories: ["AI"]
description: "A Staff-Engineer-level study of the Hermes Agent messaging gateway — from process lifecycle to platform adapters, session management to streaming delivery, and the design patterns that make it production-grade across 15+ messaging platforms."
image: "/img/ai-01.jpg"
keywords: ["Hermes Gateway", "asyncio", "Telegram adapter", "Discord adapter", "Slack adapter", "messaging platform", "stream consumer", "graceful shutdown"]
---

# Building a Multi-Platform Agentic Gateway: A Deep Dive into Hermes Gateway

> *A Staff-Engineer-level study of the Hermes Agent messaging gateway — from process lifecycle to platform adapters, session management to streaming delivery, and the design patterns that make it production-grade across 15+ messaging platforms.*


---

> [!NOTE]
> ### Executive TL;DR
> This document is a comprehensive deep dive into the **Hermes Gateway** — ~75,000+ lines of Python across 50+ modules connecting an AI agent to 15+ messaging platforms (Telegram, Discord, Slack, WhatsApp, Signal, Matrix, and more). Key architectural patterns:
> - **Abstract Platform Adapter Pattern:** A `BasePlatformAdapter` ABC normalizes wildly different platform APIs into a single `MessageEvent` dataclass. Adding a new platform requires ~5 required methods.
> - **Process-Lifetime Agent Cache:** AIAgent instances cached per session (LRU, max 128, 1h idle TTL) preserve prompt caching across turns — ~10x cost savings on Anthropic.
> - **Two-Layer Authorization:** Static allowlists gate who can talk to the bot; slash command access control gates which commands allowed users can run. Code-based pairing handles new-user onboarding.
> - **Graceful Shutdown with Resume:** Draining waits for in-flight runs (180s default). If a run times out, the session is marked `resume_pending` so the next message auto-continues from where it left off.
> - **Dual Session Storage:** SQLite (FTS5 full-text search) for message transcripts + JSONL for legacy compatibility. Metadata persists via atomic temp-file-rename writes.
> - **Streaming with Three Transports:** `GatewayStreamConsumer` bridges sync agent callbacks to async platform delivery with progressive edit, native draft streaming, and fallback chunked send.
> - **Plugin Architecture:** `PlatformRegistry` and `PairingStore` are module-level singletons. Any platform adapter can be registered via plugin with zero core changes.

---

## 1. Overview: What the Gateway Is

The **Hermes Gateway** (`gateway/`) is a long-running **asyncio** process that sits between messaging platforms and the AI agent. It is the bridge between the synchronous, single-user CLI experience and the asynchronous, multi-user, multi-platform world of Telegram, Discord, Slack, WhatsApp, and 15+ other platforms.

### 1.1 What It Provides

| Concern | Implementation |
|:---------|:---------------|
| Multi-platform ingress | Telegram, Discord, Slack, WhatsApp, Signal, Matrix, Mattermost, HomeAssistant, Email, SMS, DingTalk, Feishu, WeCom, Weixin, BlueBubbles, QQ Bot, Yuanbao, webhooks, API server |
| Session management | Per-chat, per-thread, per-user conversation isolation with configurable reset policies (daily, idle, none) |
| Authorization | Static allowlists, slash command access tiers, and code-based pairing for new users |
| Streaming delivery | Real-time token-by-token response streaming (edit-based, draft-based, or chunked fallback) |
| Cron delivery | Route scheduled job outputs to specific platforms, channels, or threads |
| Cross-session persistence | SQLite + JSONL dual persistence with FTS5 full-text search |
| Agent caching | LRU-cached AIAgent instances preserving prompt caching across turns |
| Graceful lifecycle | Draining, restart with resume, PID-based liveness, cross-process lock contention |
| Extensibility | Plugin platform adapters via `PlatformRegistry` — no core changes needed |

What the Gateway is **not**: it is not a web framework, not a message queue, and not an RPC server. It is a **process-boundary orchestrator** that translates platform-native events into a uniform dataclass, routes them through the agent loop, and delivers responses back to the originating (or any other) platform.

> **Critical architectural distinction:** The gateway does **not** expose a general-purpose API to the outside world. The only HTTP listeners are platform-required webhooks (Telegram webhook mode, WhatsApp Cloud API callbacks) and a lightweight internal REST API (`api_server.py`) used exclusively by the `send_message` tool and cron delivery when the gateway is running. You cannot call the gateway via REST to "run an agent turn" — the only way a message enters the pipeline is through a platform adapter (polling loop, WebSocket, webhook receiver, or socket mode). It is fundamentally an **event-driven process with background schedulers** (expiry watcher at 60s, channel directory refresh at 5min, platform reconnection with backoff, cron tick loop), not a service you integrate with via API calls.

### 1.2 Scale (as of May 2026)

```
~75,000+ lines of Python across 50+ files
  ├── gateway/run.py            — 17,073 lines (GatewayRunner + bootstrap)
  ├── gateway/platforms/base.py —  3,730 lines (BasePlatformAdapter ABC)
  ├── gateway/config.py         —  1,873 lines (GatewayConfig, Platform enum, env bridge)
  ├── gateway/session.py        —  2,026 lines (SessionStore, SessionEntry, SessionContext)
  ├── gateway/status.py         —    971 lines (PID file, scoped locks, takeover markers)
  ├── gateway/stream_consumer.py—  1,286 lines (GatewayStreamConsumer)
  └── gateway/platforms/*.py    — ~50,000 lines (Telegram, Discord, Slack, etc.)
```

### 1.3 High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     Gateway Process                           │
│                                                               │
│   asyncio event loop                                          │
│     │                                                         │
│     ├─ GatewayRunner                                          │
│     │   ├─ Adapter: Telegram  ─── Polling/Webhook  ⇄ Telegram API │
│     │   ├─ Adapter: Discord   ─── Gateway WebSocket ⇄ Discord API  │
│     │   ├─ Adapter: Slack     ─── Socket Mode      ⇄ Slack API     │
│     │   ├─ Adapter: WhatsApp  ─── Webhook Server   ⇄ WhatsApp API  │
│     │   ├─ ... (15+ more)                                   │
│     │   └─ Adapter: Webhook   ─── HTTP Server       ⇄ HTTP  │
│     │                                                       │
│     ├─ SessionStore (per-platform session states)            │
│     ├─ DeliveryRouter (cron output routing)                  │
│     ├─ AIAgent Cache (LRU, 128 max)                         │
│     └─ PairingStore (pending codes + approved users)         │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### 1.4 Module Map — File-by-File

```
gateway/
├── __init__.py              ← Public API exports
├── run.py                   ← GatewayRunner: lifecycle, message dispatch, agent integration (17,073 LOC)
│
├── config.py                ← GatewayConfig, Platform enum (dynamic plugin members),
│                               SessionResetPolicy, StreamingConfig, PlatformConfig (~1,873 LOC)
├── session.py               ← SessionStore, SessionEntry, SessionSource, SessionContext,
│                               build_session_key(), build_session_context_prompt(), reset policies (~2,026 LOC)
├── delivery.py              ← DeliveryRouter, DeliveryTarget — cron output routing (258 LOC)
├── hooks.py                 ← HookRegistry — event hook system (gateway:startup, agent:start, ...)
│
├── platform_registry.py     ← PlatformRegistry singleton + PlatformEntry dataclass
├── platforms/
│   ├── base.py              ← BasePlatformAdapter ABC, MessageEvent, SendResult, EphemeralReply,
│   │                          Image/audio/video/document cache, URL safety, SSRF guard, proxy (3,730 LOC)
│   ├── ADDING_A_PLATFORM.md ← 16-step guide for new platforms
│   ├── telegram.py / discord.py / slack.py / whatsapp.py / signal.py / matrix.py /
│   │   mattermost.py / homeassistant.py / email.py / sms.py / dingtalk.py / feishu.py /
│   │   feishu_comment.py / wecom.py / weixin.py / bluebubbles.py / qqbot/ / yuanbao.py /
│   │   api_server.py / webhook.py / msgraph_webhook.py — platform adapters (~50K LOC total)
│   └── helpers.py / telegram_network.py / signal_rate_limit.py / ... — shared utilities
│
├── status.py                ← PID file management, scoped token locks, runtime health,
│                               takeover markers, process discovery (971 LOC)
├── restart.py               ← Restart constants (drain timeout, exit codes)
├── stream_consumer.py       ← GatewayStreamConsumer — sync→async streaming bridge (1,286 LOC)
├── session_context.py       ← Session context injection helpers
├── mirror.py                ← Cross-session message mirroring for transcript context
├── channel_directory.py     ← Cached channel/contact map for send_message tool
├── pairing.py               ← Code-based DM pairing (OWASP + NIST SP 800-63-4)
├── slash_access.py          ← Per-platform slash command access control tiers
├── shutdown_forensics.py    ← Gateway shutdown diagnostics
├── display_config.py        ← Platform display configuration helpers
├── runtime_footer.py        ← Runtime footer injection
├── whatsapp_identity.py     ← WhatsApp ID canonicalization (JID/LID)
└── sticker_cache.py         ← Telegram sticker cache utilities
```

---

## 2. Gateway Lifecycle: From Boot to Shutdown

The gateway's lifecycle is the story of how a process starts, connects to platforms, handles messages, and cleanly shuts down. This section traces that journey from `python -m gateway.run` to the final exit.

### 2.1 Boot Sequence (Module-Level Initialization)

Before `GatewayRunner` is even constructed, a carefully sequenced module-level bootstrap runs inside `gateway/run.py`.

#### Phase 1: UTF-8 Bootstrap (line 18-25)

```python
try:
    import hermes_bootstrap  # noqa: F401
except ModuleNotFoundError:
    pass
```

`hermes_bootstrap` replaces `sys.stdout`/`sys.stderr` with UTF-8 wrappers on Windows. On POSIX it's a no-op. The `try/except` is defensive — during `hermes update`, git-reset may land new code before `uv pip install -e .` finishes, leaving `hermes_bootstrap` temporarily unregistered.

#### Phase 2: SSL Certificate Detection (line 307-342)

```python
_ensure_ssl_certs()
```

Attempts to set `SSL_CERT_FILE` from up to 4 sources: Python's compiled-in defaults, certifi, common distro paths (Debian, RHEL, Fedora, Alpine, macOS Homebrew). This runs **before** any HTTP library imports because the Discord SDK, aiohttp, and other dependencies silently fail on NixOS and other systems without standard CA cert paths.

The SSL check runs at **module import time** (line 373), not inside `GatewayRunner.__init__`, because by the time `GatewayRunner` is constructed, `import discord` has already happened and its `aiohttp.ClientSession` was created without `SSL_CERT_FILE`.

#### Phase 3: Env Var Bridging (config.yaml → os.environ, lines 424-583)

This is the most critical phase. The gateway bridges config.yaml values into environment variables before any adapter or agent is constructed:

| Section | Env Vars Bridged | Priority |
|:---------|:-----------------|:----------|
| `terminal.*` | `TERMINAL_ENV`, `TERMINAL_CWD`, `TERMINAL_TIMEOUT`, etc. | Config overrides `.env` |
| `agent.*` | `HERMES_MAX_ITERATIONS`, `HERMES_AGENT_TIMEOUT`, etc. | Config overrides `.env` (unconditional) |
| `auxiliary.*` | `AUXILIARY_VISION_PROVIDER`, `AUXILIARY_WEB_EXTRACT_MODEL`, etc. | Set when non-default |
| `display.*` | `HERMES_GATEWAY_BUSY_INPUT_MODE`, `BUSY_ACK_ENABLED` | Config overrides `.env` |
| Platform-specific | `SLACK_REQUIRE_MENTION`, `DISCORD_REQUIRE_MENTION`, `TELEGRAM_ALLOWED_CHATS` | Config overrides `.env` |
| Timezone | `HERMES_TIMEZONE` | Config only |
| Security | `HERMES_REDACT_SECRETS` | Config overrides `.env` |

> **Key insight:** The bridge uses **unconditional writes** (not `if X not in os.environ` guards) for `agent.*`, `display.*`, `timezone`, and `security.*` settings. This was a hard lesson from PR #18413 where a stale `.env` entry (`HERMES_MAX_ITERATIONS=60`) silently shadowed the user's current `config.yaml` (`max_turns: 500`), causing a 60-iteration cap that took days to diagnose.

The bridge failure path: previously silent (`except: pass`), now it prints to stderr because at module-import time `logger` is not yet initialized.

#### Phase 4: Config Validation (lines 594-606)

```python
try:
    from hermes_cli.config import print_config_warnings
    print_config_warnings()
except Exception as _bootstrap_exc:
    print(f"  Warning: config validation failed: {_bootstrap_exc}", file=sys.stderr)
```

Three checks run:
1. `print_config_warnings()` — catches malformed sections
2. `warn_deprecated_cwd_env_vars()` — tells users to migrate from `MESSAGING_CWD` / `TERMINAL_CWD` in `.env` to `terminal.cwd` in config.yaml
3. Network config — `apply_ipv4_preference()` if `network.force_ipv4` is set

#### Phase 5: Gateway-Specific Env Vars

These markers are set at module-import time, spread across the file:

- `_HERMES_GATEWAY = "1"` at **line 371** (right after `_ensure_ssl_certs()`), because `cli.py`'s module-level `load_cli_config()` could be triggered by any subsequent import — the marker must be in place before any lazy import of CLI code. Without this marker, the CLI config loader would overwrite `TERMINAL_CWD` with the CLI-specific default, breaking gateway terminal operations.
- `HERMES_QUIET = "1"` and `HERMES_EXEC_ASK = "1"` at **lines 609-612** (after config validation), ensuring they don't suppress startup warnings.

### 2.2 GatewayRunner: The Central Controller

`GatewayRunner` (defined at `gateway/run.py:1175`) manages the entire gateway runtime. Its constructor (~120 lines) initializes six categories of state:

| Category | Key Attributes |
|:----------|:---------------|
| Adapters | `adapters: Dict[Platform, BasePlatformAdapter]`, `_failed_platforms` |
| Sessions | `session_store: SessionStore`, `_running_agents`, `_pending_messages`, `_queued_events`, `_session_sources` |
| Agent Cache | `_agent_cache` (OrderedDict, 128 max, 1h idle TTL), `_agent_cache_lock` |
| Model | `_session_model_overrides`, `_session_reasoning_overrides`, `_prefill_messages`, `_reasoning_config`, `_service_tier` |
| Lifecycle | `_shutdown_event`, `_draining`, `_restart_requested`, `_exit_code`, `_stop_task` |
| Other | `pairing_store`, `hooks` (HookRegistry), `_voice_mode`, `_background_tasks`, `_pending_approvals`, `_session_db` (SQLite) |

The constructor also wires in:
- **Process registry** — `process_registry.has_active_for_session()` prevents resetting sessions with live background processes
- **SQLite auto-prune** — opportunistic maintenance of ended sessions older than `retention_days` (default 90), throttled to once per day
- **Tirith security scanner** — ensures the security scanner binary is available

#### Lifecycle Methods

```
GatewayRunner.start()
  ├── _connect_platforms()         ← Connect all configured adapters
  ├── _schedule_periodic_tasks()   ← Session expiry watcher, channel directory refresh (5 min)
  ├── write_pid_file()             ← PID file + runtime lock (O_CREAT|O_EXCL)
  └── hooks.emit("gateway:startup")

GatewayRunner.stop()
  ├── _drain_active_sessions()     ← Give in-flight agents time to finish (180s default)
  ├── _disconnect_platforms()      ← Disconnect all adapters
  ├── remove_pid_file()            ← Cleanup PID + lock files
  └── hooks.emit("gateway:shutdown")

GatewayRunner.restart()
  ├── Same as stop() + spawn replacement process
  └── Exit with GATEWAY_SERVICE_RESTART_EXIT_CODE (75)
```

### 2.3 Platform Connection & Reconnection

#### Adapter Factory

Platform adapters are created in `_create_adapter()`. The resolution order is:

```
1. Plugin platforms checked first via platform_registry.create_adapter()
2. If None returned → built-in if/elif chain:
     Platform.TELEGRAM → TelegramAdapter(config)
     Platform.DISCORD  → DiscordAdapter(config)
     ...
```

This means a plugin can override any built-in platform by registering the same name.

#### Connection with Timeout

```python
async def _connect_adapter_with_timeout(self, adapter, platform) -> bool:
    timeout = self._platform_connect_timeout_secs()  # default 30s
    if timeout <= 0:
        return await adapter.connect()
    try:
        return await asyncio.wait_for(adapter.connect(), timeout=timeout)
    except asyncio.TimeoutError:
        raise TimeoutError(f"{platform.value} connect timed out after {timeout:g}s")
```

Each platform has its own timeout so a single hung connection (e.g., Discord WebSocket stuck in handshake) doesn't block all other platforms.

#### Reconnection with Backoff

Failed platforms are tracked in `_failed_platforms: Dict[Platform, Dict]`:

```python
self._failed_platforms = {
    Platform.TELEGRAM: {
        "config": platform_config,
        "attempts": 3,
        "next_retry": 1735680000.0,  # epoch seconds
    }
}
```

A periodic background task (`_retry_failed_platforms()`) attempts reconnection with exponential backoff + jitter. On successful connection, the adapter's `_mark_connected()` writes runtime status. On fatal error, `_set_fatal_error()` marks the adapter as disconnected and fires the error handler.

### 2.4 Graceful Shutdown with Drain

#### The Drain Flow

```
1. Gateway receives SIGTERM (or /restart or internal restart_requested)
2. GatewayRunner._initiate_shutdown():
   ├── Set _draining = True
   ├── Stop accepting new messages
   ├── Clear adapter._pending_messages
   └── Wait for in-flight agent runs:
        ├── For each _running_agents:
        │     Wait up to restart_drain_timeout (default 180s)
        │     If timeout → set session resume_pending = True
        └── Continue after timeout or all agents complete
3. Disconnect all platform adapters
4. Clean up PID file
5. Exit with 0 (clean) or GATEWAY_SERVICE_RESTART_EXIT_CODE (75)
```

#### Service Restart Exit Code

```python
# EX_TEMPFAIL from sysexits.h
GATEWAY_SERVICE_RESTART_EXIT_CODE = 75
```

When the gateway exits with 75, systemd interprets `Restart=on-failure` and revives the process. When it exits with 0, systemd considers it a clean shutdown and does NOT restart — this distinction is critical for distinguishing planned restarts from unexpected deaths.

### 2.5 PID Files, Locks & Takeover Markers

The gateway uses a **three-layer process identity system** to prevent duplicate instances, handle `--replace` handoffs, and distinguish planned shutdowns from crashes. Understanding this system is key to how the gateway maintains single-instance guarantees across restarts.

#### Layer 1: PID File (`status.py:479`)

```python
def write_pid_file():
    path = _get_pid_path()  # {HERMES_HOME}/gateway.pid
    record = json.dumps({
        "pid": os.getpid(),
        "kind": "hermes-gateway",
        "argv": list(sys.argv),
        "start_time": _get_process_start_time(os.getpid()),
    })
    fd = os.open(path, os.O_CREAT | os.O_EXCL | os.O_WRONLY)
    with os.fdopen(fd, "w") as f:
        f.write(record)
```

The `O_CREAT | O_EXCL` pattern ensures exactly one process wins the race during concurrent `--replace` invocations — if two processes try to write the PID file simultaneously, only one succeeds and the other gets `FileExistsError`.

The PID record stores four fields:
- **pid** — the OS process ID
- **kind** — always `"hermes-gateway"` (used for identity validation when cmdline is unavailable)
- **argv** — the full command line for identity checks
- **start_time** — the process start timestamp, critical for PID-reuse detection

#### Layer 2: Runtime Lock (`status.py:421`)

```python
def acquire_gateway_runtime_lock() -> bool:
    path = _get_gateway_lock_path()  # {HERMES_HOME}/gateway.lock
    handle = open(path, "a+")
    if not _try_acquire_file_lock(handle):
        return False
    _write_gateway_lock_record(handle)
    return True
```

The lock is an **OS-managed advisory file lock**:

```python
def _try_acquire_file_lock(handle) -> bool:
    try:
        if _IS_WINDOWS:
            # Use byte-range locking at a high offset so runtime status readers
            # can still read the file while the lock is held.
            handle.seek(_WINDOWS_LOCK_OFFSET)  # 1 MB offset
            msvcrt.locking(handle.fileno(), msvcrt.LK_NBLCK, 1)
        else:
            fcntl.flock(handle.fileno(), fcntl.LOCK_EX | fcntl.LOCK_NB)
        return True
    except (BlockingIOError, OSError):
        return False
```

Key property: if the process dies (SIGKILL, segfault, OOM), the OS **automatically releases the file lock**. This means there's no stale-lock problem — the lock dies with the process. This is fundamentally different from the PID file, which must be explicitly cleaned up.

#### Layer 3: Process Identity Validation (`status.py:925`)

`get_running_pid()` combines all three layers to answer "is the gateway running?":

```python
def get_running_pid(pid_path, *, cleanup_stale=True):
    # Step 1: Check the OS-level file lock first
    lock_active = is_gateway_runtime_lock_active(lock_path)
    if not lock_active:
        cleanup_invalid_pid_path(pid_path)  # Delete dead PID file + lock
        return None

    # Step 2: Read the PID file metadata
    record = _read_pid_record(pid_path)
    if not record:
        return None
    pid = record.get("pid")

    # Step 3: Verify the PID is alive
    if not _pid_exists(pid):
        return None

    # Step 4: Verify it's actually a gateway process (not a reused PID)
    if not _looks_like_gateway_process(pid):
        return None

    return pid
```

The validation is deliberately conservative — all four checks must pass.

##### `_pid_exists()` — the cross-platform landmine (`status.py:335`)

```python
def _pid_exists(pid: int) -> bool:
    try:
        import psutil
        return bool(psutil.pid_exists(int(pid)))
    except ImportError:
        if _IS_WINDOWS:
            # ctypes: OpenProcess + WaitForSingleObject
            ...
        else:
            try:
                os.kill(pid, 0)  # Safe on POSIX — signal 0 is a no-op
                return True
            except OSError:
                return False
```

> **Critical Windows quirk (bpo-14484):** Python's `os.kill(pid, 0)` on Windows is **NOT** a no-op like it is on POSIX. CPython's Windows implementation treats `sig=0` as `CTRL_C_EVENT` because the two values collide at the C level. Calling `os.kill(pid, 0)` on Windows silently sends a Ctrl+C to the **entire console process group** containing that PID, often killing unrelated processes. The gateway avoids this by using `psutil` (which uses `OpenProcess + GetExitCodeProcess` on Windows) and falling back to ctypes `OpenProcess` if psutil is unavailable.

##### `_looks_like_gateway_process()` (`status.py:167`)

```python
def _looks_like_gateway_process(pid: int) -> bool:
    cmdline = _read_process_cmdline(pid)
    if not cmdline:
        return False

    patterns = (
        "hermes_cli.main gateway",
        "hermes_cli/main.py gateway",
        "hermes gateway",
        "hermes-gateway",
        "gateway/run.py",
    )
    return any(pattern in cmdline for pattern in patterns)
```

`_read_process_cmdline()` (`status.py:126`) uses a three-tier fallback:

```python
def _read_process_cmdline(pid: int) -> Optional[str]:
    # Tier 1: Linux — /proc/{pid}/cmdline (null-separated, fastest)
    cmdline_path = Path(f"/proc/{pid}/cmdline")
    try:
        raw = cmdline_path.read_bytes()
        if raw:
            return raw.replace(b"\x00", b" ").decode("utf-8", errors="ignore").strip()
    except OSError:
        pass

    # Tier 2: macOS/BSD — ps command
    try:
        result = subprocess.run(
            ["ps", "-p", str(pid), "-o", "command="],
            capture_output=True, text=True, timeout=5,
        )
        if result.returncode == 0 and result.stdout.strip():
            return result.stdout.strip()
    except OSError:
        pass

    # Tier 3: Windows — psutil (already loaded by _pid_exists)
    try:
        import psutil
        return " ".join(psutil.Process(pid).cmdline())
    except Exception:
        return None
```

Without this check, a PID-reused-by-chrome.exe would be falsely identified as a running gateway.

There's also `_record_looks_like_gateway()` that validates from the PID file's `argv` metadata when the live process's cmdline is unavailable (e.g., permissions error on `/proc`).

##### `_get_process_start_time()` — Linux-only (`status.py:111`)

```python
def _get_process_start_time(pid: int) -> Optional[int]:
    stat_path = Path(f"/proc/{pid}/stat")
    try:
        # Field 22 in /proc/<pid>/stat is process start time (clock ticks).
        return int(stat_path.read_text(encoding="utf-8").split()[21])
    except OSError:
        return None
```

This reads field 22 from `/proc/{pid}/stat`, which is the process start time in kernel clock ticks. **Linux-only** — on macOS and Windows, `_get_process_start_time()` always returns `None`. This means the PID + start_time matching in takeover markers is best-effort: on Linux it provides robust protection against PID reuse; on other platforms the marker still works for the TTL window (60s) but without the start_time guard.

#### Takeover Markers: Solving the systemd Restart Loop (`--replace`, `status.py:834`)

The most interesting piece: **takeover markers** enable zero-downtime updates by letting the old process distinguish "I'm being replaced" from "I crashed."

**The problem** `--replace` originally had:

```
1. New process SIGTERMs old process
2. Old process exits with code 1 (because SIGTERM looks like a crash)
3. systemd sees Restart=on-failure, exit code 1 → revives old process
4. Now both gateways try to connect to the same bot token → flap loop
```

**The solution — a handoff marker file:**

```
1. New process calls write_takeover_marker(target_pid):
   → Writes {HERMES_HOME}/.gateway-takeover.json
     {
       "target_pid": 12345,
       "target_start_time": 284792034,  # Prevent PID-reuse false matches
       "replacer_pid": 12346,
       "written_at": "2026-03-15T14:12:00+00:00"
     }

2. New process sends SIGTERM to old process

3. Old process's signal handler calls consume_takeover_marker_for_self():
   def consume_takeover_marker_for_self() -> bool:
       return _consume_pid_marker_for_self(
           _get_takeover_marker_path(),
           pid_field="target_pid",
           start_time_field="target_start_time",
           ttl_s=_TAKEOVER_MARKER_TTL_S,  # 60 seconds
       )

   _consume_pid_marker_for_self():
   ┌─ Read marker file
   ├─ Check TTL: marker older than 60s → stale, return False
   ├─ Match: target_pid == our_pid AND start_time matches?
   ├─ Always unlink marker (one-shot)
   └─ Return match result

4. If matched → exit(0)  — clean exit, systemd does NOT restart
   If not matched → exit(75) — EX_TEMPFAIL, systemd Restart=on-failure revives
```

**Why the start_time check matters:** PIDs get reused by the OS. If process A (PID 12345) wrote a marker, then exited, and 30 seconds later a new unrelated process B got PID 12345, we don't want B to consume A's marker. The `target_start_time` field prevents this — it's compared against `_get_process_start_time(os.getpid())`.

**Why the 60-second TTL:** If the marker write succeeds but the SIGTERM never arrives (network issue, bug), the marker would sit forever and a future gateway startup on the same `HERMES_HOME` could falsely match it. The TTL ensures markers older than 60s are treated as stale and silently cleaned up.

**Same pattern: planned stop markers** (`write_planned_stop_marker()` at `status.py:886`). Identical mechanism, different filename (`.gateway-planned-stop.json`), used for `hermes gateway stop`. This ensures `hermes gateway stop` exits with code 0 (no systemd restart) while an unexpected crash still exits with code 75 (systemd revives it).

### 2.6 Resume-Pending Recovery

When an in-flight agent run is interrupted by gateway shutdown (drain timeout), the session is marked `resume_pending`:

```python
def mark_resume_pending(self, session_key, reason="restart_timeout"):
    entry = self._entries.get(session_key)
    if entry and not entry.suspended:
        entry.resume_pending = True
        entry.resume_reason = reason
        self._save()
        return True
    return False
```

On startup, `_recover_interrupted_sessions()` marks recently active sessions as resumable. When the user sends the next message:
- `get_or_create_session()` sees `resume_pending=True` and returns the existing entry (same session_id)
- The transcript is reloaded intact
- `_handle_message_with_agent()` runs the auto-continue path
- If the last transcript entry was a tool result, a system note is prepended: *"[System note: Your previous turn was interrupted before you could process the last tool result(s)..."*

#### Freshness Gate

```python
_AUTO_CONTINUE_FRESHNESS_SECS_DEFAULT = 60 * 60  # 1 hour

def _is_fresh_gateway_interruption(timestamp, *, now=None, window_secs=None):
    window = window_secs or _AUTO_CONTINUE_FRESHNESS_SECS_DEFAULT
    if window <= 0:
        return True
    timestamp = _coerce_gateway_timestamp(timestamp)
    if timestamp is None:
        return True  # Legacy transcript: treat as fresh
    return (now or time.time()) - timestamp <= window
```

This prevents a stale `resume_pending` marker (from a gateway restart that interrupted a session days ago) from reviving itself when a user sends a new message.

#### Stuck-Loop Detection (#7536)

A persistent problem: the gateway restarts mid-turn, the session resumes, the same code path triggers the same error, and the gateway restarts again — creating a spin loop. The gateway maintains a **stuck-loop file** (`{HERMES_HOME}/.stuck_loop.json`):

```python
def _increment_restart_failure_counts(self, active_session_keys: set):
    path = self._hermes_home / self._STUCK_LOOP_FILE
    counts = json.loads(path.read_text()) if path.exists() else {}
    for key in active_session_keys:
        counts[key] = counts.get(key, 0) + 1
    for key in list(counts.keys()):
        if key not in active_session_keys:
            counts[key] = max(0, counts[key] - 1)
    counts = {k: v for k, v in counts.items() if v > 0}
    path.write_text(json.dumps(counts))
```

Key invariant: only **sessions that were active at restart time** get their counter incremented. Sessions that completed normally get decremented. After 3 consecutive failures, the session is auto-suspended:

```python
if attempts >= 3:
    entry.suspended = True          # Force clean slate
    entry.resume_pending = False
```

#### Fallback Eviction Gating (#7130)

When an agent run fails (invalid model, provider outage, etc.), the gateway must **not** evict the cached agent:

```python
_run_failed = result.get("failed") if result else False
if fallback_activated and not _run_failed:
    self._evict_cached_agent(session_key)
```

The gate condition ensures the cache entry survives failures so the next message can retry instantly without rebuilding tool schemas or MCP connections.

---

## 3. The Platform Adapter Pattern

The gateway connects to 15+ messaging platforms, each with wildly different APIs (polling vs webhook, markdown vs no markdown, message length limits, media handling). The `BasePlatformAdapter` ABC normalizes all of them into a single interface.

### 3.1 BasePlatformAdapter ABC (`gateway/platforms/base.py:1265`)

#### Required Abstract Methods

```python
class BasePlatformAdapter(ABC):
    @abstractmethod
    async def connect(self) -> bool:
        """Connect to the platform, start listeners."""

    @abstractmethod
    async def disconnect(self):
        """Stop listeners, close connections, cancel tasks."""

    @abstractmethod
    async def send(self, chat_id: str, content: str, ...) -> SendResult:
        """Send a text message."""

    @abstractmethod
    async def send_typing(self, chat_id: str):
        """Send typing indicator."""

    @abstractmethod
    async def send_image(self, chat_id, image_url, caption) -> SendResult:
        """Send an image."""

    @abstractmethod
    async def get_chat_info(self, chat_id) -> dict:
        """Return {name, type, chat_id} for a chat."""
```

#### Optional Methods (with Default Stubs)

| Method | Purpose | Default |
|:--------|:---------|:---------|
| `send_document` | Send a file attachment | Not implemented |
| `send_voice` | Send a voice message | Not implemented |
| `send_video` | Send a video | Not implemented |
| `send_media_group` | Send an album/group of media | Not implemented |
| `edit_message` | Edit an existing message | Raises NotImplementedError |
| `delete_message` | Delete a message | No-op |
| `upload_file` | Upload a file to platform CDN | Not implemented |
| `send_draft` | Native streaming draft preview | Raises NotImplementedError |
| `supports_draft_streaming` | Whether native drafts are supported | Returns False |

#### State Tracking

```python
class BasePlatformAdapter:
    _active_sessions: Dict[str, asyncio.Event]  # Per-session interrupt signals
    _pending_messages: Dict[str, MessageEvent]    # Queued follow-ups
    _session_tasks: Dict[str, asyncio.Task]       # Owner task per session
    _background_tasks: set[asyncio.Task]           # Background processing
    _running: bool                                 # Connection state
    _fatal_error_code/_message/_retryable          # Error state
```

#### Message Handler Registration (base.py:1515)

```python
MessageHandler = Callable[[MessageEvent], Awaitable[Optional[Union[str, "EphemeralReply"]]]]

def set_message_handler(self, handler: MessageHandler):
    self._message_handler = handler
```

The handler returns a string (normal reply), an `EphemeralReply` (auto-deleting system notice), or `None` (response already delivered via streaming).

#### The handle_message Dispatch (base.py:2812)

When a platform receives an inbound event, the adapter calls `self.handle_message(event)`:

1. **Pre-gateway dispatch plugin hook** — before any authorization runs, plugins can `skip` or `rewrite` the event
2. **Interrupt policy check** — if the session has an active agent, either queue (`busy_input_mode="queue"`) or interrupt (`"interrupt"`, default)
3. **Message type switch** — `PHOTO` → vision pipeline, `VOICE` → STT, `DOCUMENT` → document handling, `COMMAND` → slash dispatch, `TEXT` → agent
4. **Pending message merge** — photo bursts merged into a single event
5. **Fire handler** — call `self._message_handler(event)` (the callback set by GatewayRunner)

The interrupt logic:

```python
if session_key in self._active_sessions:
    mode = os.environ.get("HERMES_GATEWAY_BUSY_INPUT_MODE", "interrupt")
    if mode == "queue":
        merge_pending_message_event(self._pending_messages, session_key, event)
        return
    elif mode == "interrupt":
        merge_pending_message_event(self._pending_messages, session_key, event)
        if event.is_command() and event.get_command() in {"stop", "new", "reset"}:
            self._interrupt_session(session_key, reason=f"Command: /{event.get_command()}")
        else:
            self._interrupt_session(session_key, reason="New message")
```

### 3.2 MessageEvent: The Universal Exchange Format

Every platform adapter normalizes inbound messages into a `MessageEvent` dataclass:

```python
@dataclass
class MessageEvent:
    text: str                                    # Message content
    message_type: MessageType = MessageType.TEXT  # TEXT, PHOTO, VOICE, etc.
    source: SessionSource = None                  # Where it came from
    raw_message: Any = None                       # Original platform object
    message_id: Optional[str] = None              # Platform message ID
    platform_update_id: Optional[int] = None      # For offset tracking (Telegram)
    media_urls: List[str] = field(default_factory=list)    # Local cached paths
    media_types: List[str] = field(default_factory=list)    # MIME types
    reply_to_message_id: Optional[str] = None     # Reply context
    reply_to_text: Optional[str] = None           # Replied-to message text
    auto_skill: Optional[str | list[str]] = None  # Channel-bound skill
    channel_prompt: Optional[str] = None          # Per-channel ephemeral prompt
    channel_context: Optional[str] = None         # History backfill context
    internal: bool = False                        # Synthetic events bypass auth
    timestamp: datetime = field(default_factory=datetime.now)
```

Key design decisions:
- **media_urls are local cache paths**, not platform URLs — because platform URLs (especially Telegram's) expire after ~1 hour
- **reply_to_text** is included so the agent sees what the user was replying to without fetching history
- **auto_skill** enables channel-skill bindings (e.g., `#code-review` → `code-review` skill)
- **channel_prompt** supports per-channel ephemeral system prompts
- **channel_context** is set by Discord's history backfill when `require_mention` is active

### 3.3 PlatformRegistry & Plugin Discovery (`gateway/platform_registry.py:162`)

The `PlatformRegistry` is a module-level singleton that allows platform adapters to self-register without hardcoded if/elif chains.

#### PlatformEntry Dataclass

```python
@dataclass
class PlatformEntry:
    name: str                                          # "irc", "teams", etc.
    label: str                                         # Human-readable "IRC"
    adapter_factory: Callable[[Any], Any]              # Receives PlatformConfig
    check_fn: Callable[[], bool]                       # Deps available?
    validate_config: Optional[Callable[[Any], bool]]   # Properly configured?
    required_env: list[str]                            # Env vars needed
    install_hint: str = ""
    setup_fn: Optional[Callable[[], None]] = None
    source: str = "plugin"                             # "builtin" or "plugin"
    plugin_name: str = ""
    # ... env_enablement_fn, apply_yaml_config_fn, cron_deliver_env_var,
    #     standalone_sender_fn, pii_safety, emoji, platform_hint, ...
```

The `adapter_factory` pattern (vs. a bare class reference) lets plugins do custom init — wrapping in try/except, passing extra kwargs, or conditionally selecting an adapter subclass.

#### Dynamic Platform Enum

The `Platform` enum at `gateway/config.py:100` supports dynamic members through `_missing_()`:

```python
class Platform(Enum):
    TELEGRAM = "telegram"
    DISCORD = "discord"
    # ... 18+ built-in members ...

    @classmethod
    def _missing_(cls, value):
        """Create pseudo-members for plugin platforms on-demand."""
        if value in _Platform__bundled_plugin_names:
            pseudo = object.__new__(cls)
            pseudo._value_ = value
            pseudo._name_ = value.upper().replace("-", "_")
            cls._value2member_map_[value] = pseudo
            cls._member_map_[pseudo._name_] = pseudo
            return pseudo
        if platform_registry.is_registered(value):
            ...  # Same for runtime-registered plugins
        return None
```

`Platform("irc")` works without modifying the enum, and `Platform("irc") is Platform("irc")` holds identity stability through the cached pseudo-member.

### 3.4 Adding a New Platform

#### Plugin Path (Recommended, Zero Core Changes)

Create `~/.hermes/plugins/<name>/plugin.yaml` + `__init__.py`:

```yaml
name: irc
description: "IRC Platform Adapter"
version: "1.0"
kind: platform
```

```python
def register(ctx):
    ctx.register_platform(PlatformEntry(
        name="irc", label="IRC",
        adapter_factory=lambda cfg: IRCAdapter(cfg),
        check_fn=lambda: bool(shutil.which("irssi")),
        validate_config=lambda cfg: bool(cfg.extra.get("server")),
        required_env=["IRC_SERVER"],
    ))
```

The plugin system handles: adapter creation, config parsing, user authorization, cron delivery, send_message routing, system prompt hints, and status display.

#### Plugin Hook Points

| Hook | Purpose | Called |
|:------|:---------|:--------|
| `env_enablement_fn` | Seed `PlatformConfig.extra` from env vars | Before adapter construction |
| `apply_yaml_config_fn` | Translate config.yaml keys → env vars | During `load_gateway_config()` |
| `cron_deliver_env_var` | Name of `*_HOME_CHANNEL` env var | Cron delivery routing |
| `standalone_sender_fn` | Out-of-process message delivery | Cron jobs outside gateway |
| `setup_fn` | Interactive configuration | `hermes gateway setup` |

#### Built-in Path (16 Integration Points)

Requires changes across: platform adapter file, config.py (Platform enum), run.py (adapter creation + auth maps), session.py (identity fields), toolsets.py, cron/scheduler.py, send_message_tool.py, status display, and setup wizard.

---

## 4. The Message Pipeline

This is the core journey of a message: from the platform's WebSocket or HTTP endpoint, through authorization and session resolution, into the agent loop, and back out as a streamed response.

### 4.1 End-to-End Flow

```
Platform WebSocket/HTTP Request
        │
   adapter.handle_message(event)
        │          ┌─ pre_gateway_dispatch plugin hook (skip/rewrite)
        │          │
   BasePlatformAdapter._on_message()
        │          ┌─ Interrupt policy (queue vs interrupt)
        │          ├─ Message type switch (PHOTO→vision, VOICE→STT)
        │          └─ Pending message merge (photo bursts)
        │
   ┌────┴────────────────────────────────────────────┐
   │  GatewayRunner._handle_message_with_agent()     │
   │                                                 │
   │  1. Authorization check (Phase 1)               │
   │     ├─ Is user in allow_from / group_allow_from? │
   │     ├─ Check ALLOW_ALL_USERS env var            │
   │     ├─ Internal event bypass?                   │
   │     └─ unauthorized_dm_behavior (pair / ignore) │
   │                                                 │
   │  2. Session resolution                          │
   │     ├─ get_or_create_session(source) → session_key │
   │     ├─ Check reset policy (idle/daily)          │
   │     ├─ Handle resume_pending / suspended        │
   │     └─ Stuck-loop detection (3 failures → suspend)│
   │                                                 │
   │  3. Pre-processing                              │
   │     ├─ Convert media to local cache paths       │
   │     ├─ STT transcription (voice messages)       │
   │     ├─ Vision enrichment (photo messages)       │
   │     ├─ Build sender prefix (multi-user groups)  │
   │     └─ Inject channel_context (Discord backfill)│
   │                                                 │
   │  4. Agent cache lookup                          │
   │     ├─ Cache hit (session_key + sig match)      │
   │     │   └─ move_to_end(), _init_cached_agent    │
   │     ├─ Cache miss → new AIAgent(...)            │
   │     └─ _enforce_agent_cache_cap() (evict LRU)   │
   │                                                 │
   │  5. Slash command dispatch                      │
   │     ├─ /stop, /new, /reset → interrupt          │
   │     ├─ /model, /reasoning → session override    │
   │     ├─ /help, /status → direct response         │
   │     ├─ Skill command → load skill + agent       │
   │     └─ Other → check slash access policy        │
   │                                                 │
   │  6. Agent turn (run_conversation)               │
   │     ├─ Stream consumer setup (if enabled)       │
   │     ├─ Thread pool executor (sync agent)        │
   │     └─ Completion handlers                      │
   │                                                 │
   │  7. Post-processing                             │
   │     ├─ Normalize empty responses                │
   │     ├─ Handle MEDIA: tags (image delivery)      │
   │     ├─ Clear resume_pending flag                │
   │     ├─ Persist transcript (SQLite + JSONL)      │
   │     └─ Drain queued follow-ups                  │
   │                                                 │
   └─────────────────────────────────────────────────┘
```

### 4.2 Authorization Gates

#### Phase 1: Message-Level Auth (`_is_user_authorized()`, run.py:5475)

```python
platform_env_map = {
    Platform.DISCORD: "DISCORD_ALLOWED_USERS",
    Platform.TELEGRAM: "TELEGRAM_ALLOWED_USERS",
    ...
}
platform_allow_all_map = {
    Platform.DISCORD: "DISCORD_ALLOW_ALL_USERS",
    ...
}
```

Check order:
1. Is the user in the platform's `allow_from` or `group_allow_from` list?
2. Is the `ALLOW_ALL_USERS` env var set?
3. Is the message internal? (synthetic events bypass auth)
4. If `unauthorized_dm_behavior == "pair"` → return a pairing code
5. If `unauthorized_dm_behavior == "ignore"` → silently drop

#### Two-Level Message Guard

The gateway has **two** sequential message guards that control/approval commands must bypass:

1. **Base adapter** (`BasePlatformAdapter.handle_message`) — queues messages in `_pending_messages` when `session_key in self._active_sessions`
2. **Gateway runner** (`GatewayRunner._handle_message_with_agent`) — intercepts `/stop`, `/new`, `/queue`, `/status`, `/approve`, `/deny` before they reach the agent

Commands like approval responses MUST bypass both guards and be dispatched inline, not via `_process_message_background()` (which races session lifecycle).

### 4.3 Session Resolution

```python
session_entry = self.session_store.get_or_create_session(source)
session_key = session_entry.session_key
```

The session key is a deterministic string: `agent:main:{platform}:{chat_type}:{chat_id}[:{thread_id}][:{user_id}]`

Resolution handles:
- **Reset policy evaluation** — check if idle or daily expiry has triggered
- **Resume-pending recovery** — if the last turn was interrupted by restart, auto-continue
- **Suspended sessions** — `/stop` marks `suspended=True`, next access creates a fresh session
- **Background process protection** — sessions with active processes are never reset

### 4.4 Preprocessing

Before the agent sees the message:

- **Media download** — platform URLs (especially Telegram's) expire after ~1 hour; all media is downloaded to `{HERMES_HOME}/cache/images/` before creating the event
- **STT transcription** — voice messages are transcribed via speech-to-text
- **Vision enrichment** — photos are analyzed for the agent
- **Sender prefix** — multi-user groups get a prefix like "`[alice]: hello`" so the agent knows who's talking
- **Channel context backfill** — when `require_mention` is active on Discord, the gateway fetches messages between bot turns and prepends them so the agent doesn't miss context

### 4.5 Agent Cache: Getting (or Reusing) the AIAgent

#### Cache Architecture

```python
self._agent_cache: OrderedDict[str, tuple] = OrderedDict()
# Key: session_key
# Value: (AIAgent, config_signature_str)
# LRU order: most recently used at end
self._agent_cache_lock = threading.Lock()
```

On each message, the lookup is **inlined** directly in `_handle_message_with_agent()`:

```python
# ~run.py:15240
_sig = self._agent_config_signature(model, runtime, enabled_toolsets, ...)
agent = None
_cache_lock = getattr(self, "_agent_cache_lock", None)
_cache = getattr(self, "_agent_cache", None)
if _cache_lock and _cache is not None:
    with _cache_lock:
        cached = _cache.get(session_key)
        if cached and cached[1] == _sig:
            agent = cached[0]                         # Cache hit
            if hasattr(_cache, "move_to_end"):
                _cache.move_to_end(session_key)       # Refresh LRU
            self._init_cached_agent_for_turn(agent, ...)

if agent is None:
    # Config changed or first message — create fresh
    agent = AIAgent(model=..., ...)
    if _cache_lock and _cache is not None:
        with _cache_lock:
            _cache[session_key] = (agent, _sig)
            self._enforce_agent_cache_cap()
```

#### Cache Invalidation Triggers

1. **Config signature changes** — model, provider, enabled toolsets, etc.
2. **Session expires** — `_session_expiry_watcher()` evicts idle agents
3. **LRU cap reached** — `_enforce_agent_cache_cap()` pops the least-recently-used
4. **Session is suspended** — `/stop` marks `suspended=True`, next access creates fresh

#### Why This Matters

Without the cache, every message would:
- Re-load config.yaml, re-discover plugins, re-build tool schemas
- Re-read SOUL.md, re-load memory, re-initialize LLM client SDKs
- On Anthropic, this would break prompt caching entirely — the entire system prompt would be re-sent on every turn instead of only sending the new user message (roughly 10x cost increase)

#### Eviction Behavior

`_enforce_agent_cache_cap()` is sophisticated about mid-turn agents:

```python
# Agents currently in _running_agents are SKIPPED — their clients,
# terminal sandboxes, and child subagents are all in active use.
# If every candidate in LRU order is active, the cache stays over cap
# temporarily; it will re-check on the next insert.
running_ids = {id(a) for a in self._running_agents.values() if a}
excess = max(0, len(_cache) - _AGENT_CACHE_MAX_SIZE)
for key in ordered_keys[:excess]:
    agent = _cache[key][0]
    if agent is not None and id(agent) in running_ids:
        continue  # Don't evict active agents
    evict_plan.append((key, agent))
```

Evicted agents get **soft cleanup** (`_release_evicted_agent_soft`) rather than full teardown — terminal sandboxes, browser daemons, and background processes must survive the Python `AIAgent` instance being garbage collected, because the next turn may resume the same task.

### 4.6 Slash Command Dispatch

Slash commands are intercepted at step 5 of the pipeline:

| Command | Action |
|:---------|:--------|
| `/stop`, `/new`, `/reset` | Interrupt current run, clear/reset session |
| `/model`, `/reasoning` | Store per-session model/reasoning overrides (cache-invalidating) |
| `/help`, `/status` | Respond directly without agent |
| Skill command (e.g., `/code-review`) | Load skill into agent context |
| All others | Check slash access policy → allow/deny |

#### Two-Level Pending Queue

The gateway has two pending-message buffers with different semantics:

| Buffer | Where | Purpose | Merging |
|:--------|:-------|:---------|:---------|
| `adapter._pending_messages` | `BasePlatformAdapter` | Single-slot "next-turn" follow-up | Collapses repeated sends |
| `self._queued_events` | `GatewayRunner` | FIFO `/queue` command buffer | Never merges — each produces its own turn |

When `_pending_messages` slot is occupied, additional `/queue` items land in `_queued_events` and are promoted one-at-a-time after each run's drain.

#### Queued Event Drain

```python
# After run_conversation() returns:
while True:
    pending = _dequeue_pending_event(adapter, session_key)
    if not pending:
        break
    result = await _run_agent(session_key, pending.text, ...)
    # Chain results via _preserve_queued_followup_history_offset
```

This ensures rapid follow-ups (e.g., "fix that bug" → "actually in the second function" → "the one named validate") are all handled within a single "user turn" instead of each getting its own full agent cycle.

### 4.7 Streaming Delivery (GatewayStreamConsumer)

The `GatewayStreamConsumer` (`gateway/stream_consumer.py:77`) bridges synchronous agent callbacks to async platform delivery.

#### Architecture

```
Agent Worker Thread                asyncio Task
┌──────────────┐                 ┌──────────────────┐
│ step: model  │                 │ GatewayStreamCon- │
│ generates    │                 │ sumer.run()       │
│ token stream │                 │                   │
│              │  queue.Queue    │ drains queue,     │
│ on_delta(t) ─┼────────────────>│ rate-limits,      │
│ on_delta(t) ─┼────────────────>│ edits message     │
│ ...          │                 │                   │
│ finish()    ─┼────────────────>│ got_done → final  │
└──────────────┘                 └──────────────────┘
                                        │
                                   adapter.edit_message()
                                   (or send_draft, or send)
```

#### Three Streaming Transports

| Transport | Mechanism | When Used |
|:-----------|:-----------|:-----------|
| **edit** | `send()` first → `edit_message()` progressively | Universal fallback |
| **draft** | `send_draft()` native animated preview | Telegram DM (Bot API 9.5+) |
| **off** | No streaming, deliver final response | Platforms without edit support (webhook, Signal) |

Resolution logic (`_resolve_draft_streaming()`):

```python
transport = cfg.transport  # "auto" | "draft" | "edit" | "off"
if transport == "edit":
    return False
if transport == "draft":
    return adapter.supports_draft_streaming(chat_type=...)
if transport == "auto":
    return adapter.supports_draft_streaming(chat_type=...)
```

#### Think-Block Filtering

Models like MiniMax emit `<think>...</think>` tags inline. The consumer implements a state machine to strip these:

```python
def _filter_and_accumulate(self, text: str):
    # State machine: _in_think_block bool
    # Opening tags at block boundaries (start of text or after newline)
    # Closing tags anywhere
    # Partial tags at buffer boundaries held in _think_buffer
```

The block-boundary check prevents false positives when the model mentions `<think>` in prose.

#### Adaptive Rate-Limiting

```python
_max_flood_strikes = 3
_current_edit_interval = cfg.edit_interval  # starts at 0.8s

# On flood control error:
self._flood_strikes += 1
self._current_edit_interval *= 2  # exponential backoff
if self._flood_strikes >= self._max_flood_strikes:
    self._edit_supported = False  # fall back to chunked sends
```

The `edit_interval` (0.8s) is tuned for Telegram's ~1 edit/s flood envelope. The `buffer_threshold` (24 chars) makes short replies feel near-instant in DMs.

#### Fresh-Final Delivery

For long-running responses (port from openclaw/openclaw#72038):

```python
_fresh_final_after_seconds = 60.0

async def _try_fresh_final(self, text):
    new_message = await adapter.send(chat_id, text)  # fresh timestamp
    await adapter.delete_message(chat_id, old_preview_id)  # cleanup
```

This makes the platform's visible timestamp reflect completion time instead of first-token time — especially important for reasoning models that stream slowly.

### 4.8 Post-Processing

After `run_conversation()` returns:
- **Response normalization** — empty responses replaced with fallback text
- **Media delivery** — `MEDIA:` tags in the response trigger `send_image()`/`send_document()`
- **Resume-pending cleared** — successful completion clears the flag
- **Transcript persistence** — messages appended to both SQLite and JSONL
- **Queued follow-up drain** — check `_pending_messages` and `_queued_events` for more work
- **Token tracking** — costs are accumulated in the `SessionEntry`

---

## 5. Session & State Management

Sessions are the gateway's mechanism for isolating conversations. Each user/chat/thread combination maps to a deterministic session key, backed by dual persistence (SQLite + JSONL) and governed by configurable reset policies.

### 5.1 How Sessions Connect to the Pipeline

Sessions are the bridge between the **message pipeline** (Section 4) and **persistence**. Every step of the pipeline reads from and writes to the session:

| Pipeline Step | Session Interaction |
|:---------------|:-------------------|
| Authorization | Determines which session isolation policy applies |
| Session resolution | `get_or_create_session(source)` → returns or creates `SessionEntry` |
| Preprocessing | `channel_prompt` and `auto_skill` are per-session settings |
| Agent cache | Cache keyed by `session_key`, invalidated on session reset |
| Response delivery | Origin stored in `SessionSource` for reply routing |
| Post-processing | Transcript appended to session's SQLite + JSONL storage |
| Cron delivery | `DeliveryRouter` looks up origin/home channel from session |

### 5.2 Session Key Design

The session key is a deterministic string built from `SessionSource` fields:

```
agent:main:{platform}:{chat_type}:{chat_id}[:{thread_id}][:{user_id}]
```

Examples:
- `agent:main:telegram:dm:12345` — Telegram DM with user 12345
- `agent:main:discord:group:98765:thread42` — Discord thread 42 in group 98765
- `agent:main:discord:group:98765:thread42:user_abc` — with per-user isolation

The key is built by `build_session_key()` (`gateway/session.py:600`) which encodes isolation policies:

| Scenario | Behavior |
|:----------|:----------|
| DM | Per-user isolation always (key includes `chat_id`) |
| Group, `group_sessions_per_user=True` (default) | Per-user isolation (key includes `user_id`) |
| Group, `group_sessions_per_user=False` | Shared session (no `user_id` in key) |
| Thread, `thread_sessions_per_user=True` | Per-user isolation in threads |
| Thread, `thread_sessions_per_user=False` (default) | Shared thread session |

WhatsApp IDs are canonicalized before key construction to handle the JID/LID bridge alias problem (`whatsapp_identity.py`).

### 5.3 SessionEntry: What Gets Stored per Session

```python
@dataclass
class SessionEntry:
    session_key: str
    session_id: str                        # "20250315_143022_a1b2c3d4"
    created_at: datetime
    updated_at: datetime
    origin: Optional[SessionSource]        # For delivery routing
    display_name: Optional[str]
    platform: Optional[Platform]
    chat_type: str
    # Token tracking
    input_tokens, output_tokens, cache_read_tokens, cache_write_tokens, ...
    estimated_cost_usd: float
    # Reset lifecycle flags
    was_auto_reset: bool
    auto_reset_reason: Optional[str]       # "idle" or "daily"
    is_fresh_reset: bool
    reset_had_activity: bool
    expiry_finalized: bool                 # Memory flush completed
    # Recovery flags
    suspended: bool                        # /stop → hard wipe on next access
    resume_pending: bool                   # Restart-interrupted → preserve
    resume_reason: Optional[str]
    last_resume_marked_at: Optional[datetime]
```

### 5.4 Dual Persistence Strategy

The gateway uses **three** storage mechanisms, each serving a different purpose:

| Storage | Content | Purpose |
|:---------|:---------|:---------|
| `sessions.json` | Session metadata (key→id, tokens, flags) | Fast session lookup, temp-file atomic write |
| SQLite (SessionDB) | Full message transcript | FTS5 full-text search across sessions |
| JSONL per session | Full message transcript (legacy) | Backward compatibility, simple streaming reads |

#### Metadata Persistence (`sessions.json`)

```python
def _save(self):
    fd, tmp_path = tempfile.mkstemp(dir=str(self.sessions_dir), prefix=".sessions_")
    try:
        with os.fdopen(fd, "w", encoding="utf-8") as f:
            json.dump(data, f, indent=2)
            f.flush()
            os.fsync(f.fileno())
        atomic_replace(tmp_path, sessions_file)  # os.replace()
    except BaseException:
        os.unlink(tmp_path)
        raise
```

The temp-file-then-rename pattern prevents partial writes and naturally invalidates mtime-based caches.

#### SQLite Auto-Prune

On startup, if `sessions.auto_prune: true` in config.yaml, the gateway runs opportunistic maintenance:

```python
if self._session_db is not None:
    _sess_cfg = _load_full_config().get("sessions") or {}
    if _sess_cfg.get("auto_prune", False):
        self._session_db.maybe_auto_prune_and_vacuum(
            retention_days=int(_sess_cfg.get("retention_days", 90)),
            min_interval_hours=int(_sess_cfg.get("min_interval_hours", 24)),
            vacuum=bool(_sess_cfg.get("vacuum_after_prune", True)),
        )
```

Failures are logged but never raised — the gateway should not crash because SQLite maintenance failed.

#### SQLite FTS5 Table Architecture

The `SessionDB` creates two FTS5 virtual tables for full-text search on message transcripts:

```sql
CREATE VIRTUAL TABLE messages_fts USING fts5(content)
CREATE VIRTUAL TABLE messages_fts_trigram USING fts5(content, tokenize='trigram')
```

Each virtual table spawns an entire family of shadow tables automatically. What looks like 10+ tables in the schema is actually just two FTS5 indexes:

| Table Family | Files | Purpose |
|:-------------|:-------|:---------|
| `messages_fts` | `_config`, `_content`, `_data`, `_docsize`, `_idx` | Standard word/token search (fast exact + prefix match) |
| `messages_fts_trigram` | `_trigram_config`, `_trigram_content`, `_trigram_data`, `_trigram_docsize`, `_trigram_idx` | Trigram tokenizer for substring/fuzzy search |

Why two indexes:
- **`messages_fts`** — word-based. Fast for `"hello world"`, prefix searches. Uses the default tokenizer.
- **`messages_fts_trigram`** — breaks text into 3-character chunks. Enables substring matching: searching `"ello"` still finds `"hello"`.

The single column (`content`) is by design — only the message text needs indexing for search, not metadata fields like `role`, `timestamp`, or `session_id` (those are stored in regular columns outside FTS).

For application code, you only ever query `messages_fts` or `messages_fts_trigram` directly. The shadow tables (`_data`, `_content`, `_docsize`, `_idx`, `_config`) are SQLite's internal machinery — the inverted index B-tree, raw text storage, BM25 ranking sizes, and segment index respectively.

### 5.5 SessionSource & Context

Every message originates from a `SessionSource` — the gateway's representation of "who said what where":

```python
@dataclass
class SessionSource:
    platform: Platform
    chat_id: str
    chat_name: Optional[str]
    chat_type: str  # "dm", "group", "channel", "thread"
    user_id: Optional[str]
    user_name: Optional[str]
    thread_id: Optional[str]
    chat_topic: Optional[str]
    user_id_alt: Optional[str]     # Signal UUID, Feishu union_id
    chat_id_alt: Optional[str]     # Signal group internal ID
    is_bot: bool
    guild_id: Optional[str]        # Discord guild, Slack workspace
    parent_chat_id: Optional[str]  # Parent channel for threads
    message_id: Optional[str]      # Triggering message ID
```

#### PII Redaction

When `redact_pii=True` and the platform is in `_PII_SAFE_PLATFORMS` (WhatsApp, Signal, Telegram, BlueBubbles), identifiers are deterministically hashed:

```python
def _hash_id(value: str) -> str:
    return hashlib.sha256(value.encode("utf-8")).hexdigest()[:12]
```

Discord is intentionally excluded because mentions use `<@user_id>` and the LLM needs real IDs to tag users.

#### SessionContext Prompt

`build_session_context_prompt()` generates a system prompt section at runtime that tells the agent:
- Where it is (platform, DM vs group, channel topic)
- Who it's talking to (name/ID, redacted if applicable)
- What platforms are connected
- Where home channels are for cron delivery
- Platform-specific behavioral notes (e.g., Slack "you cannot search channel history")

### 5.6 Reset Policies & Expiry Watcher

#### Policy Configuration

```yaml
session_reset:
  mode: "both"            # "daily" | "idle" | "both" | "none"
  at_hour: 4              # Local time for daily reset
  idle_minutes: 1440      # 24 hours
  notify: true            # Send notification on auto-reset
```

Policy resolution follows a strict priority chain:

```
1. reset_by_platform[Platform.TELEGRAM]  → platform override
2. reset_by_type["dm"]                    → chat-type override
3. default_reset_policy                    → fallback
```

```python
def _should_reset(self, entry, source) -> Optional[str]:
    policy = self.config.get_reset_policy(platform=source.platform, session_type=source.chat_type)
    if policy.mode == "none":
        return None
    if policy.mode in {"idle", "both"}:
        idle_deadline = entry.updated_at + timedelta(minutes=policy.idle_minutes)
        if now > idle_deadline:
            return "idle"
    if policy.mode in {"daily", "both"}:
        today_reset = now.replace(hour=policy.at_hour, minute=0, ...)
        if entry.updated_at < today_reset:
            return "daily"
    return None
```

Sessions with active background processes are never reset.

#### Background Expiry Watcher

A periodic task (`_session_expiry_watcher()`) runs every 60 seconds:

```python
async def _session_expiry_watcher(self):
    while not self._draining:
        await asyncio.sleep(60)
        for entry in self.session_store._entries.values():
            if entry.suspended:
                continue
            if self.session_store._is_session_expired(entry):
                if not entry.expiry_finalized:
                    await self.hooks.emit("session:end", {...})
                    entry.expiry_finalized = True
                    self._evict_agent(entry.session_key)
```

The watcher also prunes old entries (`_prune_old_entries()`, every 5 minutes) and evicts idle agents from the cache (TTL > 3600s).

### 5.7 SessionSource LRU Cache

The `GatewayRunner` maintains an LRU cache of `SessionSource` objects keyed by `session_key` (max 512). This is used by fallback routing paths (shutdown notifications, synthetic background-process events) when the persisted origin is missing and `_parse_session_key` can't recover `thread_id`.

### 5.8 Delivery Router & Cron

The `DeliveryRouter` (`gateway/delivery.py:109`) routes cron job outputs to their targets.

#### Delivery Target Parsing

```
"origin"           → Back to the chat where the job was created
"local"            → Save to {HERMES_HOME}/cron/output/
"telegram"         → Telegram home channel (from config)
"telegram:12345"   → Specific Telegram chat
"telegram:12345:678" → Specific Telegram thread in chat 12345
```

#### Output Truncation

```python
MAX_PLATFORM_OUTPUT = 4000
TRUNCATED_VISIBLE = 3800

if len(content) > MAX_PLATFORM_OUTPUT:
    saved_path = _save_full_output(content, job_id)
    content = content[:TRUNCATED_VISIBLE] + f"\n\n... [truncated, full output saved to {saved_path}]"
```

The full output is always saved locally. The platform message shows a truncated preview.

#### Mirroring (Cross-Session Transcripts)

The `mirror.py` module appends "delivery-mirror" records to target session transcripts when messages are sent via `send_message` or cron delivery. This ensures that when the agent sends a message to a platform, the receiving session's transcript shows the context — so a user who resumes that session later sees the full conversation, including messages delivered by other means.

```python
def mirror_to_session(platform, chat_id, message_text, source_label="cli"):
    mirror_msg = {
        "role": "assistant",
        "content": message_text,
        "timestamp": datetime.now().isoformat(),
        "mirror": True,
        "mirror_source": source_label,
    }
    _append_to_jsonl(session_id, mirror_msg)
    _append_to_sqlite(session_id, mirror_msg)
```

---

## 6. Security & Authorization

The gateway implements a layered security model: message-level allowlists gate who can talk to the bot, slash command access control gates what authorized users can do, code-based pairing handles new-user onboarding, and scoped locks prevent credential conflicts.

### 6.1 Static Allowlists

Per-platform allowlists in config.yaml:

```yaml
telegram:
  allow_from: ["123456789", "987654321"]      # DM whitelist
  group_allow_from: ["111111111", "222222222"] # Group chat whitelist
```

Resolution in `_is_user_authorized()` at run.py:5475:

```python
platform_env_map = {
    Platform.DISCORD: "DISCORD_ALLOWED_USERS",
    Platform.TELEGRAM: "TELEGRAM_ALLOWED_USERS",
    Platform.SLACK: "SLACK_ALLOWED_USERS",
    ...
}
platform_allow_all_map = {
    Platform.DISCORD: "DISCORD_ALLOW_ALL_USERS",
    Platform.TELEGRAM: "TELEGRAM_ALLOW_ALL_USERS",
    ...
}
```

Check order:
1. Is the user in the platform's `allow_from` or `group_allow_from`?
2. Is the `*_ALLOW_ALL_USERS` env var set?
3. Is the message internal? (synthetic events bypass auth)
4. If `unauthorized_dm_behavior == "pair"` → return a pairing code
5. If `unauthorized_dm_behavior == "ignore"` → silently drop

### 6.2 Slash Access Control (`gateway/slash_access.py`)

Per-platform slash command access tiers:

```python
@dataclass(frozen=True)
class SlashAccessPolicy:
    enabled: bool
    admin_user_ids: FrozenSet[str]
    user_allowed_commands: FrozenSet[str]

    def is_admin(self, user_id) -> bool:
        if not self.enabled:
            return True  # gating off = everyone is admin
        return str(user_id) in self.admin_user_ids

    def can_run(self, user_id, canonical_cmd) -> bool:
        if not self.enabled:
            return True
        if self.is_admin(user_id):
            return True
        if canonical_cmd in _ALWAYS_ALLOWED_FOR_USERS:
            return True
        return canonical_cmd in self.user_allowed_commands
```

#### Scope Resolution

```python
def _scope_for_chat_type(chat_type) -> str:
    if chat_type in {"dm", "direct", "private", ""}:
        return "dm"
    return "group"
```

#### Always-Allowed Commands

Even when gating is enabled and the user has no allowed commands, these remain reachable:

```python
_ALWAYS_ALLOWED_FOR_USERS = frozenset({"help", "whoami"})
```

Only `help` and `whoami` — pure discovery commands so non-admin users can see what they can do without leaking operational data.

### 6.3 Pairing System (Code-Based Approval)

The `PairingStore` (`gateway/pairing.py:76`) implements a code-based authorization flow. Instead of requiring the operator to maintain static allowlists with raw user IDs, unknown users receive a one-time code that the operator approves via the CLI.

#### Security Design

```python
# Unambiguous alphabet — excludes 0/O, 1/I to prevent confusion
ALPHABET = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789"
CODE_LENGTH = 8

# Timing constants
CODE_TTL_SECONDS = 3600             # 1-hour expiry
RATE_LIMIT_SECONDS = 600            # 1 request/user/10min
LOCKOUT_SECONDS = 3600              # Lockout after failures
MAX_PENDING_PER_PLATFORM = 3        # Cap on pending codes
MAX_FAILED_ATTEMPTS = 5             # Failed approvals before lockout
```

#### Flow

```
1. Unknown user sends DM
2. Gateway: "You are not authorized. Use /pair <CODE> in the CLI."
3. PairingStore.generate_code() creates 8-char code from 32-char alphabet
   → Stores pending request, records rate limit entry
4. Operator runs: hermes pair approve <CODE>
5. PairingStore.approve_code() validates, adds user to approved list
6. User can now message the agent
```

#### File Security

All pairing data files get `chmod 0600` with temp-file-rename atomicity:

```python
def _secure_write(path, data):
    path.parent.mkdir(parents=True, exist_ok=True)
    fd, tmp_path = tempfile.mkstemp(...)
    try:
        with os.fdopen(fd, "w") as f:
            f.write(data)
            f.flush()
            os.fsync(f.fileno())
        atomic_replace(tmp_path, path)
        os.chmod(path, 0o600)
    except BaseException:
        os.unlink(tmp_path)
        raise
```

### 6.4 Scoped Token Locks (`status.py:578`)

Machine-local scoped locks prevent two gateway instances from using the same external credential:

```python
def acquire_scoped_lock(scope, identity, metadata=None):
    lock_path = _get_lock_dir() / f"{scope}-{sha256(identity)[:16]}.lock"
```

Lock files live at `{XDG_STATE_HOME}/hermes/gateway-locks/` (overridable via `HERMES_GATEWAY_LOCK_DIR`).

#### Stale Lock Detection

```python
stale = False
if not _pid_exists(existing_pid):
    stale = True
elif current_start != existing.get("start_time"):
    stale = True  # PID was reused
elif not _looks_like_gateway_process(existing_pid):
    stale = True  # process exists but isn't a gateway
```

#### Usage in Adapters

```python
# Telegram adapter connect():
def connect(self):
    if not self._acquire_platform_lock("telegram_bot", bot_token, "Telegram bot token"):
        return False
```

This prevents two profiles from using the same Telegram bot token simultaneously — a common misconfiguration causing message loss and duplicate delivery.

---

## 7. Configuration

### 7.1 GatewayConfig Multi-Layer Merge

`GatewayConfig` is built by `load_gateway_config()` at `gateway/config.py` from four layers:

| Layer | Source | Priority |
|:-------|:--------|:----------|
| 1 | Built-in defaults | Lowest |
| 2 | `~/.hermes/gateway.json` (legacy) | Medium-low |
| 3 | `~/.hermes/config.yaml` gateway section | Medium-high |
| 4 | Environment variables | Highest |

#### Config-YAML Mapping

```python
sr = yaml_cfg.get("session_reset")
if sr and isinstance(sr, dict):
    gw_data["default_reset_policy"] = sr

yaml_platforms = yaml_cfg.get("platforms")
for plat_name, plat_block in yaml_platforms.items():
    merged = {**existing, **plat_block}
    merged_extra = {**existing.get("extra", {}), **plat_block.get("extra", {})}
```

#### Platform-Section Bridging

Platform settings can be set at the top level (`telegram.require_mention`) or nested under `platforms.telegram.extra`. The bridge normalizes both:

```python
for plat in _shared_loop_targets:
    platform_cfg = yaml_cfg.get(plat.value)
    if "require_mention" in platform_cfg:
        bridged["require_mention"] = platform_cfg["require_mention"]
    if "allow_from" in platform_cfg:
        bridged["allow_from"] = platform_cfg["allow_from"]
    # ... 20+ more keys ...
    plat_data, extra = _ensure_platform_extra_dict(platforms_data, plat.value)
    extra.update(bridged)
```

### 7.2 StreamingConfig

```yaml
streaming:
  enabled: false
  transport: "auto"         # "auto" | "draft" | "edit" | "off"
  edit_interval: 0.8        # Seconds between edits
  buffer_threshold: 24      # Characters before flushing edit
  cursor: " ▉"              # Typing cursor indicator
  fresh_final_after_seconds: 60.0  # Replace long previews with fresh messages
```

```python
@dataclass
class StreamingConfig:
    enabled: bool = False
    transport: str = "auto"
    edit_interval: float = 0.8
    buffer_threshold: int = 24
    cursor: str = " ▉"
    fresh_final_after_seconds: float = 60.0
```

---

## 8. Operations & Extensibility

### 8.1 Channel Directory

The channel directory (`gateway/channel_directory.py`) is a cached map of reachable channels/contacts per platform, refreshed every 5 minutes.

#### Build Strategy

- **Discord:** Enumerates guild text channels + forum channels from the live client
- **Slack:** Calls `users.conversations` API with pagination (20 pages max)
- **Others:** Pulls from session history (`sessions.json` origin data)

#### Name Resolution

```python
def resolve_channel_name(platform_name, name):
    # 0. Exact ID match (raw platform IDs)
    # 1. Exact name match (case-insensitive)
    # 2. Guild-qualified match ("GuildName/channel")
    # 3. Unambiguous prefix match
```

Used by the `send_message` tool to resolve `#bot-commands` to numeric IDs.

### 8.2 Runtime Status & Health (`status.py`)

#### Runtime Status File

```python
def write_runtime_status(gateway_state=None, platform=None, platform_state=None, ...):
    path = _get_runtime_status_path()  # {HERMES_HOME}/gateway_state.json
    payload["gateway_state"] = gateway_state
    payload["platforms"][platform] = {"state": platform_state, ...}
    payload["updated_at"] = _utc_now_iso()
    _write_json_file(path, payload)
```

State values: `"starting"`, `"connected"`, `"disconnected"`, `"fatal"`, `"draining"`, `"stopped"`.

#### Platform Health Tracking

Each platform adapter calls `_mark_connected()` / `_mark_disconnected()` / `_set_fatal_error()` which in turn call `write_runtime_status()`. This provides real-time platform health visibility to `hermes gateway status` and `hermes doctor`.

### 8.3 Shutdown Forensics

The `shutdown_forensics.py` module captures diagnostics when the gateway exits unexpectedly:

```python
# On unexpected shutdown:
_shutdown_forensics.capture(
    signal=signal.SIGTERM,
    running_agents=list(self._running_agents.keys()),
    active_sessions=self.session_store.list_sessions(active_minutes=5),
    running=True,
)
```

Captures: active session list, running agents per session, platform connection states, last N lines of the event loop, and the signal that triggered shutdown. Written to `{HERMES_HOME}/logs/gateway_shutdown.json`.

### 8.4 Event Hook System

The `HookRegistry` at `gateway/hooks.py:35` provides a lightweight event system for lifecycle hooks.

#### Supported Events

```
gateway:startup    — Gateway process starts
session:start      — New session created
session:end        — Session ends (/new or /reset)
agent:start        — Agent begins processing a message
agent:step         — Each tool-calling iteration
agent:end          — Agent finishes processing
command:*          — Any slash command executed (wildcard match)
```

#### Hook Structure

Hooks live in `~/.hermes/hooks/<name>/`:

```yaml
# HOOK.yaml
name: my-audit
description: Audit all tool use
events:
  - agent:step
```

```python
# handler.py
async def handle(event_type, context):
    tool_name = context.get("tool_name")
    duration = context.get("duration_ms")
    print(f"[audit] Tool {tool_name} took {duration}ms")
```

#### emit vs emit_collect

```python
async def emit(self, event_type, context=None):
    """Fire handlers, discard return values. Supports wildcard matching."""

async def emit_collect(self, event_type, context=None):
    """Fire handlers and collect non-None return values.
    Used for decision-style hooks (e.g., command:<name> policies)."""
```

---

## 9. MCP in the Gateway

The gateway integrates with MCP (Model Context Protocol) servers to provide the agent with dynamically discovered tools. This section clarifies the architecture: MCP tools are **not** registered "on the gateway" — they register in the global process-level `ToolRegistry` — and explains why the gateway manages the MCP lifecycle.

### 9.1 Registration Is Global, Not Gateway-Specific

MCP tools register into `tools.registry.ToolRegistry`, the same singleton used by the CLI, TUI, cron, and batch runner. Each MCP tool gets a prefixed name:

```python
# From _register_server_tools() in tools/mcp_tool.py:
registry.register(
    name=tool_name_prefixed,       # e.g. "mcp_filesystem_list_directory"
    toolset=f"mcp-{name}",         # e.g. "mcp-filesystem"
    schema=schema,
    handler=_make_tool_handler(name, mcp_tool.name, server.tool_timeout),
    ...
)
```

Every entry point calls `discover_mcp_tools()` at its own startup — there is nothing gateway-specific about registration:

| Entry Point | How It Calls `discover_mcp_tools()` |
|:-------------|:--------------------------------------|
| **Gateway** (`gateway/run.py:16967`) | `await _loop.run_in_executor(None, discover_mcp_tools)` — runs in a thread so the asyncio loop doesn't freeze |
| **CLI** (`cli.py`) | Inline synchronous call at startup (no event loop to worry about) |
| **TUI** (`tui_gateway/server.py`) | Inline synchronous call at startup |
| **ACP** (`acp_adapter/server.py`) | `asyncio.to_thread` on session init |
| **Cron** (`cron/scheduler.py`) | Inline synchronous call at startup |

If Hermes ran CLI-only (no gateway), MCP would work identically — the CLI calls discovery at its own startup.

### 9.2 Why the Gateway Lifecycle-Manages MCP

The gateway adds three layers of lifecycle management that a CLI session doesn't need:

#### 9.2.1 Persistent Connections

MCP servers maintain **long-lived background connections** throughout the process lifetime. A dedicated asyncio loop (`_mcp_loop`) runs in a daemon thread. Each configured server gets an `MCPServerTask` that keeps the transport alive with automatic reconnection (exponential backoff):

```
Gateway Process
  └─ MCP Background Thread (asyncio loop)
       ├─ MCPServerTask("filesystem")   → stdio subprocess (npx ...)
       ├─ MCPServerTask("github")       → stdio subprocess (npx ...)
       └─ MCPServerTask("remote_api")   → HTTP/SSE connection
```

Three transport modes are supported:

| Transport | Config Pattern | SDK Client |
|:-----------|:---------------|:------------|
| **stdio** | `command: npx` + `args: [...]` | `mcp.client.stdio.stdio_client` |
| **Streamable HTTP** | `url: "https://..."` | `mcp.client.streamable_http` |
| **SSE** | `url: "..."` + `transport: sse` | `mcp.client.sse.sse_client` |

The gateway is the main long-running daemon, so it handles startup discovery (`gateway/run.py:16967`) and graceful shutdown (`gateway/run.py:17013`):

```python
async def start_gateway(...):
    # MCP discovery runs in executor to avoid blocking the asyncio loop
    try:
        from tools.mcp_tool import discover_mcp_tools
        _loop = asyncio.get_running_loop()
        await _loop.run_in_executor(None, discover_mcp_tools)
    except Exception as e:
        logger.debug("MCP tool discovery failed: %s", e)

    success = await runner.start()
    ...
    await runner.wait_for_shutdown()

    # Clean shutdown: close MCP connections
    try:
        from tools.mcp_tool import shutdown_mcp_servers
        shutdown_mcp_servers()
    except Exception:
        pass
```

#### 9.2.2 Dynamic Cache Invalidation

The gateway caches `AIAgent` instances between turns (LRU, max 128, 1h idle TTL). When MCP tools change, the cached agent's tool schemas become stale. The gateway detects this via `registry._generation`:

```python
# gateway/run.py:13770 — _extract_cache_busting_config()
def _extract_cache_busting_config(cls, user_config):
    ...
    try:
        from tools.registry import registry
        out["tools.registry_generation"] = getattr(registry, "_generation", None)
    except Exception:
        out["tools.registry_generation"] = None
    return out
```

This value is mixed into `_agent_config_signature()`. When MCP tools change (via `/reload-mcp` or a server-sent `notifications/tools/list_changed` notification), the registry generation increments, the cached agent signature no longer matches, and the gateway rebuilds the agent on the next turn with fresh schemas.

#### 9.2.3 Runtime Reload (`/reload-mcp`)

The `/reload-mcp` slash command (`gateway/run.py:12098`) lets users reconnect MCP servers without restarting the gateway:

```
/reload-mcp command flow:
  1. Check approvals.mcp_reload_confirm config gate
  2. If gate is on → route through slash-confirm (Approve Once / Always / Cancel)
  3. _execute_mcp_reload():
     a. Capture old server names from _servers dict
     b. shutdown_mcp_servers() — disconnect all cleanly
     c. discover_mcp_tools() — reconnect with fresh config.yaml
     d. Compute diff: added, removed, reconnected servers
     e. Inject a system notification message into the session transcript
        so the model knows tools changed on its next turn
     f. Return summary to the user
```

The injected notification ensures the model is aware of the tool change without losing prompt cache for the conversation prefix.

### 9.3 Dynamic Tool Discovery via Server Notifications

When an MCP server sends `notifications/tools/list_changed`, the `MCPServerTask._make_message_handler()` schedules a background refresh that:

1. Calls `session.list_tools()` again on the live connection
2. Re-registers tools in the global registry (adding new ones, removing stale ones, updating existing ones)
3. Increments `registry._generation`, which the gateway detects on the next turn's cache lookup

### 9.4 Per-Platform Tool Filtering

MCP servers are included on all platforms by default, unless:
- A platform explicitly lists MCP server names (acting as an allowlist)
- The special `no_mcp` sentinel is present in the platform's toolset list
- `include_default_mcp_servers=False` is passed (used by Discord session setup)

### 9.5 Agent Consumption Is Transparent

The `AIAgent` in `run_agent.py` does not know or care that some tools are MCP tools. It receives tool schemas through the normal `get_tool_definitions()` path from the global registry. The only MCP-aware code in the agent loop is **parallel execution** support: MCP servers can opt into parallel tool calls via `supports_parallel_tool_calls: true` in their config, and the agent checks `is_mcp_tool_parallel_safe()` to decide whether to run MCP tool calls concurrently alongside other tools.

---

## 10. Platform Deep Dive: Weixin

This section uses the **Weixin (WeChat Official Account / WeChat Work)** platform adapter as a concrete example to trace the full lifecycle of a platform integrator — from registration, through connection, to inbound and outbound message handling.

Source: `gateway/platforms/weixin.py` (2,170 lines)

### 10.1 Registration Path

Weixin is a **built-in** platform (not plugin-registered). The integration path has three parts:

#### Platform Enum (`gateway/config.py`)

```python
class Platform(Enum):
    WEIXIN = "weixin"
    # ... 18+ other built-in members ...
```

#### Adapter Factory (`gateway/run.py:_create_adapter`)

```python
elif platform == Platform.WEIXIN:
    from gateway.platforms.weixin import WeixinAdapter, check_weixin_requirements
    if not check_weixin_requirements():
        logger.warning("Weixin: aiohttp/cryptography not installed")
        return None
    return WeixinAdapter(config)
```

#### Environment Auto-Enable (`gateway/config.py:1658`)

If `WEIXIN_TOKEN` or `WEIXIN_ACCOUNT_ID` is set in the environment, the gateway automatically creates a `PlatformConfig` and bridges ~10 env vars into `config.platforms[WEIXIN].extra`:

| Env Var | Config Key |
|:---------|:-----------|
| `WEIXIN_ACCOUNT_ID` | `extra.account_id` |
| `WEIXIN_TOKEN` | `token` (or `extra.token`) |
| `WEIXIN_BASE_URL` | `extra.base_url` |
| `WEIXIN_CDN_BASE_URL` | `extra.cdn_base_url` |
| `WEIXIN_DM_POLICY` | `extra.dm_policy` |
| `WEIXIN_GROUP_POLICY` | `extra.group_policy` |
| `WEIXIN_ALLOWED_USERS` | `extra.allow_from` |
| `WEIXIN_HOME_CHANNEL` | `home_channel.chat_id` |

### 10.2 Connection Model: HTTP Long-Poll

Weixin does **not** use WebSockets or webhooks. It uses HTTP long-polling to the iLink Bot API (`https://ilinkai.weixin.qq.com`):

```python
# Core API endpoints
ILINK_BASE_URL = "https://ilinkai.weixin.qq.com"
EP_GET_UPDATES = "ilink/bot/getupdates"
EP_SEND_MESSAGE = "ilink/bot/sendmessage"
EP_SEND_TYPING = "ilink/bot/sendtyping"
EP_GET_CONFIG = "ilink/bot/getconfig"
EP_GET_UPLOAD_URL = "ilink/bot/getuploadurl"
```

#### Connect Sequence

```python
async def connect(self) -> bool:
    # 1. check_weixin_requirements() — aiohttp + cryptography installed?
    # 2. Requires token & account_id
    # 3. Acquires scoped lock (prevents two profiles using same credential)
    # 4. Creates poll_session + send_session (SSL via certifi)
    # 5. Restores context tokens from disk
    # 6. asyncio.create_task(_poll_loop())
    # 7. Registers self in _LIVE_ADAPTERS global dict
```

#### Long-Poll Loop

```python
async def _poll_loop(self):
    # Loads sync_buf from disk (cursor for get_updates offset tracking)
    while True:
        POST ilink/bot/getupdates?sync_buf=<cursor>  (35s long-poll timeout)
        ├─ Session expired (errcode=-14): pause 10 minutes
        ├─ Stale session (ret=-2 + "unknown error"): pause 10 minutes
        ├─ 3 consecutive failures: backoff 30s
        └─ For each message: asyncio.create_task(_process_message_safe())
```

The `sync_buf` is a disk-persisted cursor that survives gateway restarts, ensuring no messages are missed.

### 10.3 Inbound Message Pipeline

```
_process_message(event)
  ├─ Skip own messages (sender_id == account_id)
  ├─ Message-ID dedup (5min sliding window)
  ├─ Content-fingerprint dedup (MD5 of text per sender)
  ├─ _guess_chat_type() → "group" or "dm"
  ├─ Policy check:
  │   ├─ dm_policy: open / allowlist / disabled
  │   └─ group_policy: open / allowlist / disabled
  ├─ Save context_token from inbound message
  ├─ Typing indicator: fetch typing_ticket via getconfig API (cached 600s)
  ├─ Media processing: download + AES-128-ECB decrypt attachments
  └─ Build MessageEvent → self.handle_message(event)
```

#### Media Decryption

Weixin encrypts all media at rest on its CDN. The adapter downloads the ciphertext and decrypts locally:

```python
def _aes128_ecb_encrypt(plaintext: bytes, key: bytes) -> bytes: ...
def _aes128_ecb_decrypt(ciphertext: bytes, key: bytes) -> bytes: ...
def _parse_aes_key(aes_key_b64: str) -> bytes:
    # Handles both raw base64 (16 bytes) and hex-encoded (32 bytes of ASCII)
```

Media is dispatched by item type:
- `ITEM_IMAGE` → `_download_image()` → AES decrypt → `cache_image_from_bytes()`
- `ITEM_VIDEO` → `_download_video()` → AES decrypt → `cache_document_from_bytes()`
- `ITEM_FILE` → `_download_file()` → AES decrypt → `cache_document_from_bytes()`
- `ITEM_VOICE` → `_download_voice()` → AES decrypt → `cache_audio_from_bytes()`

All CDN URLs are validated against an SSRF allowlist:

```python
_WEIXIN_CDN_ALLOWLIST: frozenset[str] = frozenset({
    "novac2c.cdn.weixin.qq.com", "ilinkai.weixin.qq.com",
    "wx.qlogo.cn", "thirdwx.qlogo.cn", "res.wx.qq.com",
    "mmbiz.qpic.cn", "mmbiz.qlogo.cn",
})

def _assert_weixin_cdn_url(url: str) -> None:
    if parsed.host not in _WEIXIN_CDN_ALLOWLIST:
        raise ValueError(f"URL host not in allowlist: {parsed.host}")
```

### 10.4 Outbound Delivery

#### Send Flow

```python
async def send(self, chat_id, content, reply_to=None, metadata=None) -> SendResult:
    # 1. Extract MEDIA: tags and local file paths from content
    # 2. Deliver media first (voice → video → image → document)
    # 3. _split_text_for_weixin_delivery() — smart text chunking
    #    - Compact mode (default): keeps multi-line as one message
    #    - Legacy mode: one message per top-level unit
    # 4. Sequential delivery with 1.5s inter-chunk delay (configurable)
    # 5. Each chunk retries up to 4 times with backoff
    # 6. Session-expired: auto-strip context_token, retry once
    # 7. Rate-limited (-2 errcode): 3x backoff before retry
```

#### Media Upload Flow

```python
async def _send_file(self, chat_id, path, caption, force_file_attachment=False):
    # 1. Read file from local path
    # 2. Generate random AES-128 key
    # 3. Call _get_upload_url() → CDN upload URL
    # 4. AES-128-ECB encrypt + PKCS#7 pad
    # 5. POST ciphertext to CDN (SSRF-guarded host check)
    # 6. Send message with encrypted media reference
    # Note: aes_key is sent as base64(hex_string), not base64(raw_bytes)
```

#### Markdown → WeChat Rewriting

WeChat does not support standard markdown. The adapter rewrites common constructs:

```python
# Headers:     "# Title"    → "【Title】"
# Subheaders:  "## Sub"     → "**Sub**"
# Tables:      → key-value pair format
# Long lines:  wrap at 120 chars for copy-friendliness in WeChat
```

### 10.5 Session Continuity: Context Tokens

Weixin uses **context tokens** — opaque per-peer strings that the iLink API requires on every outbound send. The adapter persists these to disk:

```python
class ContextTokenStore:
    """Disk-backed context_token cache keyed by account + peer."""
    # Stores to: ~/.hermes/weixin/accounts/<account_id>.context-tokens.json
    # Methods: restore(), get(), set(), _persist()  (atomic_json_write)
```

Every inbound message updates the token; every outbound send echoes it back. This ensures session routing survives gateway restarts.

### 10.6 QR Login Flow

The adapter supports interactive login via QR code scanning:

```python
async def qr_login():
    # 1. GET ilink/bot/get_bot_qrcode?bot_type=3 → QR code image + URL
    # 2. Display QR in terminal (ASCII art via qrcode library, or URL fallback)
    # 3. Poll ilink/bot/get_qrcode_status every 1s (480s default timeout)
    # 4. States:
    #    - wait: still polling
    #    - scaned: phone scanned the QR
    #    - scaned_but_redirect: follow redirect_host (cross-account link)
    #    - expired: auto-refresh QR up to 3 times
    #    - confirmed: save credentials
    # 5. Credentials saved to ~/.hermes/weixin/accounts/<account_id>.json
```

### 10.7 One-Shot Send: `send_weixin_direct()`

Used by the `send_message` tool and cron delivery for outbound-only messages that don't go through the long-poll adapter lifecycle. If a live adapter exists with an open `_send_session`, it reuses that session; otherwise it creates an ephemeral `WeixinAdapter` with its own `aiohttp.ClientSession`.

### 10.8 What Makes Weixin Unique

| Feature | Weixin | Most Other Platforms |
|:---------|:--------|:---------------------|
| **Transport** | HTTP long-poll (35s timeout) | WebSocket / webhook |
| **Auth flow** | QR code scan (iLink Bot API) | Bot token / OAuth app |
| **Media crypto** | AES-128-ECB encrypted CDN | HTTPS direct download |
| **Message edits** | Not supported (`SUPPORTS_MESSAGE_EDITING = False`) | Supported by Telegram, Discord, Slack |
| **Session continuity** | Opaque per-peer context tokens (disk-backed) | Stateless or session IDs |
| **Sync cursor** | Disk-backed `get_updates_buf` (survives restarts) | Webhook offset / in-memory cursor |
| **Dependency gate** | `aiohttp` + `cryptography` | Varies (often just `aiohttp`) |
| **Rate limit signal** | `errcode=-2` → 3x inter-chunk backoff | 429 HTTP status |
| **Account persistence** | `~/.hermes/weixin/accounts/<id>.json` | Typically just token in `.env` |

---

## 11. Design Patterns Summary

| Pattern | Where Used | Why |
|:---------|:-----------|:-----|
| **Abstract Adapter** | `BasePlatformAdapter` ABC | Normalizes 15+ wildly different platform APIs into uniform interface |
| **Process-Boundary Architecture** | Gateway ↔ Agent (sync → async bridge) | Isolates sync Python agent loop from async I/O-heavy platform communication |
| **LRU + TTL Cache** | AIAgent cache (128 max, 1h idle) | Preserves prompt caching across turns, ~10x cost savings on Anthropic |
| **Active-Agent Eviction Protection** | `_enforce_agent_cache_cap()` skips mid-turn agents | Prevents tearing down resources in active use |
| **Temp-File-Replace Persistence** | SessionStore, PairingStore | Prevents partial writes, naturally invalidates mtime-based caches |
| **Double-Write Pattern** | SQLite + JSONL for transcripts | Backward compatibility + FTS5 search without migration risk |
| **Freshness Gating** | Auto-continue resume, lobby reminders, STT dedup | Prevents stale state from being revived after restarts/timeouts |
| **Takeover Markers** | `--replace` handoff | Breaks systemd restart loop during zero-downtime update |
| **Scoped Locking** | Token-scoped machine locks | Prevents two profiles from using the same credential simultaneously |
| **Multi-Layer Config** | gateway.json ↔ config.yaml ↔ env vars | Supports legacy configs while migrating to unified config.yaml |
| **SSRF Redirect Guard** | `_ssrf_redirect_guard` on httpx | Re-validates each redirect target after pre-flight URL safety check |
| **Adaptive Rate-Limiting** | GatewayStreamConsumer flood control | Exponential backoff on edit failures, fallback to chunked sends |
| **Dynamic Enum** | `Platform._missing_()` | Plugin platforms work without modifying the Platform enum |
| **Queued Event Drain** | `_dequeue_pending_event()` loop | Rapid follow-ups handled in a single agent turn |
| **Two-Level Queue** | `_pending_messages` + `_queued_events` | Separate semantics for interrupt follow-ups vs explicit `/queue` |
| **Deterministic Session Key** | `build_session_key()` | Same user always gets same session (until reset) regardless of connection |
| **Code-Based Pairing** | `PairingStore` with OWASP guidance | Onboarding without operator allowlist maintenance |
| **Stuck-Loop Auto-Suspend** | `_STUCK_LOOP_FILE` + 3-failure counter | Breaks infinite restart→resume→fail→restart cycles |
| **Fallback Eviction Gating** | `and not _run_failed` in eviction code | Prevents MCP reinit loop on failed runs |
| **Fatal Error Reporting** | `_set_fatal_error()` → runtime status → error handler | Propagates platform-level failures to operator with retry decision |
| **PII Redaction at Source** | `_hash_id()`, `_hash_sender_id()` | User IDs redacted before reaching LLM, routing uses original values |
| **Pre-Gateway Dispatch Hook** | `pre_gateway_dispatch` plugin hook | Plugin can skip/rewrite message before any authorization runs |
