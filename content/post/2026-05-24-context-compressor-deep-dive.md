---
layout: post
title: "Designing Context Compression for Production Agents: A Deep Dive into Hermes"
date: "2026-05-24"
author: "Jamie Zhang"
tags: ["Hermes", "Context Compression", "Agentic AI", "Summarization", "LLM Agents"]
categories: ["AI"]
description: "Staff-engineer-level notes on agent/context_compressor.py: how Hermes preserves task continuity when a long-running agent outgrows the model context window, and what the implementation teaches about summarization, compression, and failure-tolerant agent design."
image: "/img/ai-01.jpg"
keywords: ["Context Compressor", "context window", "compaction", "active task anchoring", "tool sanitization", "multimodal budgeting"]
---

# Designing Context Compression for Production Agents: A Deep Dive into Hermes

> Staff-engineer-level notes on `agent/context_compressor.py`: how Hermes
> preserves task continuity when a long-running agent outgrows the model context
> window, and what the implementation teaches about summarization, compression,
> and failure-tolerant agent design.


---

> [!NOTE]
> ### Executive TL;DR
> Hermes context compression is not "summarize the chat when it gets long." It is
> a transcript rewrite algorithm with strict invariants:
> - **Head / middle / tail partitioning:** keep the system prompt and first turns
>   intact, summarize the middle, and protect the recent tail by token budget.
> - **Active task anchoring:** the latest user message must stay outside the
>   summary. A summarized "pending ask" is reference material, not a live user
>   turn.
> - **Tool-aware compaction:** old tool outputs are deduplicated, summarized, and
>   pruned before any LLM call; tool call/result pairs are sanitized afterward so
>   providers never receive invalid message history.
> - **Iterative summaries:** second and later compactions update the existing
>   handoff instead of recursively summarizing summaries as ordinary turns.
> - **Multimodal budgeting:** images are charged a fixed token estimate so image
>   sessions do not accidentally preserve far more context than the model can fit.
> - **Failure visibility:** if the summary model fails, Hermes inserts an explicit
>   fallback marker and records dropped-turn metadata instead of silently losing
>   context.

---

## How to Use This Deep Dive

Read this document in four passes:

| Pass | Sections | What to learn |
|:------|:----------|:---------------|
| Architecture | 1-5 | Why compression is a runtime state transition; key parameters; the full algorithm |
| Algorithm | 6-11 | How Hermes chooses what to keep, summarize, and prune |
| Summarization | 12-15 | How the handoff prompt is shaped for continuity |
| Operations | 16-25 | How failures, tests, UX, and reusable patterns work |
| Expert Insights | 25 | Why the design choices were made and what they reveal about harder problems |

If you only want the implementation recipe, read sections 5, 6, 8, 11, 14, 22,
and 23. If you are designing prompts, start with sections 12 and 23.
**If you want the expert-level design reasoning, go directly to section 25.**

---

## 1. Why Context Compression Is a Core Agent Primitive

Long-running agents accumulate a different kind of context from normal chatbots.
They do not only remember prose. They build a working state:

| Context type | Why it matters |
|:--------------|:----------------|
| User intent | The current request, constraints, preferences, and corrections |
| Tool evidence | File reads, shell output, search results, browser observations |
| Execution state | Modified files, running servers, failed tests, branch/session IDs |
| Decisions | Why a path was chosen and what alternatives were rejected |
| Open loops | Work in progress, blockers, questions still waiting for the user |

The model context window is finite, but agent work is cumulative. When the
window fills, Hermes has only four choices:

1. Drop old messages and hope the tail is enough.
2. Start a fresh session and lose continuity.
3. Ask a model to summarize everything in place.
4. Rewrite the transcript into a smaller but valid working record.

`ContextCompressor` implements option 4. The important distinction is that it
does not treat compression as a model trick. It treats compression as a state
transition in the agent runtime.

The output of compression must still be a valid provider transcript, must still
preserve the active task, must still respect tool-call protocol rules, and must
still be understandable to a future model that never saw the original turns.

---

## 2. Where It Sits in Hermes

The compressor implements the `ContextEngine` interface in
`agent/context_engine.py`. `AIAgent` constructs it unless a plugin registers a
replacement context engine.

```
AIAgent.run_conversation()
    |
    |-- estimate / read prompt token usage
    |-- context_compressor.should_compress(...)
    |
    `-- _compress_context(...)
            |
            |-- memory_manager.on_pre_compress(messages)
            |-- context_compressor.compress(messages, ...)
            |-- session split / continuation bookkeeping
            |-- reset prompt caches and tool-result dedupe caches
```

Compression can happen in three broad situations:

| Trigger | Purpose |
|:---------|:---------|
| Preflight compression | Shrink history before a request that is already near the threshold |
| Error recovery | React to provider context-length or payload-too-large errors |
| Manual `/compress [focus]` | Let the user compact deliberately, optionally around a topic |

The default threshold is configured under `compression.threshold`, often around
50% of the model context. Hermes also validates the auxiliary compression model
because a summarizer with a smaller context than the main model's compression
threshold can fail exactly when it is needed.

---

## 3. Key Parameters and State

Before diving into the algorithm, here are the constructor parameters and
instance state that drive every decision:

```python
ContextCompressor(
    model: str,                        # main model name (for context-length lookup)
    threshold_percent: float = 0.50,   # compress when prompt_tokens > context * 50%
    protect_first_n: int = 3,          # extra messages to protect beyond system prompt
    protect_last_n: int = 20,          # fallback message count (token budget takes priority)
    summary_target_ratio: float = 0.20,# tail budget = threshold_tokens * 20%
    quiet_mode: bool = False,
    summary_model_override: str = None,# use a different (cheaper) model for summarization
    base_url, api_key, provider, api_mode,  # passed to auxiliary_client.call_llm()
    config_context_length: int = None, # override auto-detected context length
)
```

Derived values computed in `__init__`:

```python
self.context_length    = get_model_context_length(model, ...)
self.threshold_tokens  = max(context_length * threshold_percent, MINIMUM_CONTEXT_LENGTH)
self.tail_token_budget = threshold_tokens * summary_target_ratio
self.max_summary_tokens = min(context_length * 0.05, 12_000)
```

Per-session mutable state (reset by `on_session_reset()`):

| Field | Purpose |
|:-------|:---------|
| `_previous_summary` | Body of the last generated summary for iterative updates |
| `_ineffective_compression_count` | Anti-thrashing: incremented when savings < 10% |
| `_last_compression_savings_pct` | Savings from the most recent compression |
| `_summary_failure_cooldown_until` | Monotonic timestamp; skip summary attempts until then |
| `_last_summary_error` | Short error text for user-facing warnings |
| `_last_summary_dropped_count` | How many messages were dropped without a summary |
| `_last_summary_fallback_used` | True if a static fallback marker was inserted |
| `_last_aux_model_failure_*` | Records when the summary model fell back to the main model |

---

## 4. The Compression Contract

A production compressor has to satisfy more than "make messages shorter."
Hermes encodes several invariants:

| Invariant | Implementation |
|:-----------|:----------------|
| System prompt remains authoritative | `_protect_head_size()` always protects the system message |
| Latest user request remains active | `_ensure_last_user_message_in_tail()` anchors the last user turn |
| Tool protocol remains valid | `_sanitize_tool_pairs()` removes orphan results and stubs missing results |
| Summaries are not active instructions | `SUMMARY_PREFIX` frames summaries as reference-only handoffs |
| Existing memory remains authoritative | Summary prefix and system note explicitly preserve `MEMORY.md` / `USER.md` |
| Summary errors are visible | fallback summary marker plus `_last_summary_*` fields |
| Compression cannot thrash forever | `should_compress()` backs off after repeated low-savings passes |

The most important design lesson: compression rewrites the transcript, so it
must preserve both semantic continuity and wire-format validity.

---

## 5. The Main Algorithm

`compress()` is the single public entry point. Here is the actual method
signature and the step-by-step flow it executes:

```python
def compress(
    self,
    messages: List[Dict[str, Any]],
    current_tokens: int = None,
    focus_topic: str = None,
) -> List[Dict[str, Any]]:
```

Steps in order:

```text
1. Guard: return unchanged if too few messages to compress.
2. Phase 1 — Prune: _prune_old_tool_results()  (no LLM, deterministic)
3. Phase 2 — Boundaries:
     compress_start = _protect_head_size() + _align_boundary_forward()
     compress_end   = _find_tail_cut_by_tokens()
4. Rehydrate: _find_latest_context_summary() → restore _previous_summary
5. Phase 3 — Summarize: _generate_summary(turns_to_summarize, focus_topic)
6. Phase 4 — Assemble:
     a. Copy head messages; inject compression note into system prompt.
     b. Insert summary (or static fallback if summary failed).
     c. Decide summary role to avoid consecutive same-role messages.
     d. If both roles collide, merge summary into first tail message.
     e. Copy tail messages.
7. Sanitize: _sanitize_tool_pairs()
8. Measure savings; update anti-thrashing counters.
```

The message partition looks like this:

```
index:  0          compress_start        compress_end          n-1
        |               |                     |                 |
        +---------------+---------------------+-----------------+
        | protected head | summarized middle   | protected tail  |
        | system prompt  | old turns           | recent context  |
        | + first turns  | (replaced by LLM    | (verbatim)      |
        |                |  summary)           |                 |
        +---------------+---------------------+-----------------+
```

Why not summarize everything except the latest message? Because early context is
often structural: the system prompt, initial project constraints, selected
language, repository root, and first user goal. Hermes keeps a small head and a
token-budgeted tail, then compresses the middle.

---

## 6. Phase 1: Cheap Tool-Output Pruning

LLM summarization is expensive and lossy. Hermes first performs deterministic
compression in `_prune_old_tool_results()`. The method runs three passes over
the message list:

```
Pass 1 — Deduplicate: walk backward; for each tool result >200 chars,
         hash the content. If the same hash was seen in a more recent message,
         replace the older copy with "[Duplicate tool output — same content as
         a more recent call]".

Pass 2 — Summarize: for each tool result outside the protected tail,
         replace large content (>200 chars) with a one-line informative summary.

Pass 3 — Truncate args: for each assistant message outside the protected tail,
         shrink large tool_call arguments while preserving valid JSON.
```

The output of Pass 2 is more useful than a blind placeholder. Examples:

```text
[terminal] ran `npm test` -> exit 0, 47 lines output
[read_file] read config.py from line 1 (3,400 chars)
[search_files] content search for 'compress' in agent/ -> 12 matches
[web_search] query='context compression' (4,200 chars result)
[delegate_task] 'refactor auth module' (1,800 chars result)
```

This pass is powerful because tool outputs dominate agent transcripts. A single
`read_file`, `search_files`, browser snapshot, or test run can cost more tokens
than dozens of short chat messages. Pruning them before summarization reduces
both the main transcript and the summarizer input.

### JSON-Preserving Argument Shrinking (Pass 3)

Old assistant tool calls can contain enormous arguments, especially `write_file`
calls with full file contents. Earlier systems often slice these strings:

```text
{"path": "...", "content": "long content...
```

That creates invalid JSON, and strict providers reject the entire next request
(MiniMax returns `invalid function arguments json string` and the session gets
stuck in a loop — issue #11762). Hermes instead parses the JSON, shrinks long
string leaves, and serializes it again. Non-string values are preserved.
Non-JSON arguments pass through unchanged.

```python
def _truncate_tool_call_args_json(args: str, head_chars: int = 200) -> str:
    try:
        parsed = json.loads(args)
    except (ValueError, TypeError):
        return args   # non-JSON: pass through unchanged

    def _shrink(obj):
        if isinstance(obj, str) and len(obj) > head_chars:
            return obj[:head_chars] + "...[truncated]"
        if isinstance(obj, dict):
            return {k: _shrink(v) for k, v in obj.items()}
        if isinstance(obj, list):
            return [_shrink(v) for v in obj]
        return obj

    return json.dumps(_shrink(parsed), ensure_ascii=False)
```

This is a small but critical production detail: compression must never produce a
transcript that the provider cannot parse.

---

## 7. Phase 2: Head Protection

`_protect_head_size()` treats `protect_first_n` as additional messages beyond
the system prompt:

```python
def _protect_head_size(self, messages):
    head = 0
    if messages and messages[0].get("role") == "system":
        head = 1          # system prompt is always implicitly protected
    return head + self.protect_first_n
```

`protect_first_n` defaults to 3, so the head covers: system prompt + first 3
non-system messages (typically the opening user turn and first assistant reply).

This matters because different call paths include different message shapes.
Gateway manual compression can strip or reconstruct system context differently
from CLI runtime compression. The compressor keeps the semantics stable by
making the system prompt implicitly protected when present.

After compression, `compress()` also injects a note into the system message so
the continuation model knows the transcript was rewritten:

```python
_compression_note = (
    "[Note: Some earlier conversation turns have been compacted into a "
    "handoff summary to preserve context space. The current session state "
    "may still reflect earlier work, so build on that summary and state "
    "rather than re-doing work. Your persistent memory (MEMORY.md, USER.md) "
    "remains fully authoritative regardless of compaction.]"
)
```

This note is appended to the system message content (not prepended) so it does
not displace the identity and tool-use guidance at the top of the prompt.

---

## 8. Phase 3: Tail Protection by Token Budget

The original version of many compressors protects "last N messages." Hermes
moves beyond that. `_find_tail_cut_by_tokens()` walks backward from the end,
accumulating approximate token cost until it reaches a soft ceiling.

```python
def _find_tail_cut_by_tokens(self, messages, head_end, token_budget=None):
    if token_budget is None:
        token_budget = self.tail_token_budget   # threshold_tokens * summary_target_ratio
    n = len(messages)
    min_tail = min(3, n - head_end - 1)         # hard minimum: always keep 3 messages
    soft_ceiling = int(token_budget * 1.5)      # allow one oversized message to stay whole
    accumulated = 0
    cut_idx = n

    for i in range(n - 1, head_end - 1, -1):
        msg = messages[i]
        content_len = _content_length_for_budget(msg.get("content") or "")
        msg_tokens = content_len // _CHARS_PER_TOKEN + 10   # +10 for role/metadata overhead
        for tc in msg.get("tool_calls") or []:              # include tool_call argument length
            msg_tokens += len(tc.get("function", {}).get("arguments", "")) // _CHARS_PER_TOKEN
        if accumulated + msg_tokens > soft_ceiling and (n - i) >= min_tail:
            break
        accumulated += msg_tokens
        cut_idx = i

    # Enforce minimum tail, then align to avoid splitting tool groups
    cut_idx = min(cut_idx, n - min_tail)
    cut_idx = self._align_boundary_backward(messages, cut_idx)
    cut_idx = self._ensure_last_user_message_in_tail(messages, cut_idx, head_end)
    return max(cut_idx, head_end + 1)
```

Key parameters:

| Value | Meaning |
|:-------|:---------|
| `tail_token_budget` | `threshold_tokens * summary_target_ratio` — scales with model context |
| `min_tail` | Hard minimum of 3 protected messages |
| `soft_ceiling` | `token_budget * 1.5` — allows one oversized recent message to stay whole |
| `_CHARS_PER_TOKEN` | Rough 4 chars/token estimator |

Why token budget beats message count:

| Scenario | Message count behavior | Token-budget behavior |
|:----------|:------------------------|:-----------------------|
| 20 short chat turns | Over-compresses useful recent context | Keeps many recent turns |
| 3 huge tool outputs | Preserves too much and still overflows | Cuts aggressively |
| 5 image turns | Treats them as small text blocks | Charges image token cost |
| One oversized recent result | May split awkwardly | Allows 1.5x ceiling to keep it whole |

The tail finder also includes tool-call argument length, not just message
content. This prevents old large `tool_calls` metadata from hiding in messages
that otherwise look empty.

---

## 9. Multimodal Budgeting

Hermes handles image tokens at two layers:

| Layer | Function | Purpose |
|:-------|:----------|:---------|
| Whole-message estimate | `estimate_messages_tokens_rough()` in `agent/model_metadata.py` | Decide whether the session is near the context threshold |
| Compressor tail budget | `_content_length_for_budget()` in `agent/context_compressor.py` | Decide which recent messages remain verbatim |

Both layers follow the same principle: count each image as a fixed token cost
and do not count the raw base64 payload as text.

### Whole-Message Rough Estimation

Preflight token estimation uses this shape:

```python
def estimate_messages_tokens_rough(messages: List[Dict[str, Any]]) -> int:
    """Rough token estimate for a message list (pre-flight only).

    Image parts (base64 PNG/JPEG) are counted as a flat ~1500 tokens per
    image - the Anthropic pricing model - instead of counting raw base64
    character length. Without this, a single ~1MB screenshot would be
    estimated at ~250K tokens and trigger premature context compression.
    """
    _IMAGE_TOKEN_COST = 1500
    total_chars = 0
    image_tokens = 0
    for msg in messages:
        total_chars += _estimate_message_chars(msg)
        image_tokens += _count_image_tokens(msg, _IMAGE_TOKEN_COST)
    return ((total_chars + 3) // 4) + image_tokens
```

This is a good production approach because base64 length is the wrong signal.
A 1 MB screenshot may have roughly 1,000,000 transport characters, but the model
does not price it as 250,000 text tokens. Counting raw base64 would make Hermes
think the context is far larger than it is and would trigger premature
compression.

The helper pair behind the estimator does two things:

| Helper | Behavior |
|:--------|:----------|
| `_estimate_message_chars()` | Builds a shadow message with image payloads stripped before text-length estimation |
| `_count_image_tokens()` | Counts image-like parts and adds a flat token cost per image |

`_count_image_tokens()` covers normal multimodal content lists, stashed
Anthropic blocks in `_anthropic_content_blocks`, and multimodal tool-result
envelopes that have not yet been converted.

### Compressor-Local Tail Accounting

`_content_length_for_budget()` handles plain strings, text blocks, image blocks,
and mixed content lists while deciding where the protected tail starts.

The key constant is:

```python
_IMAGE_TOKEN_ESTIMATE = 1600
_IMAGE_CHAR_EQUIVALENT = _IMAGE_TOKEN_ESTIMATE * _CHARS_PER_TOKEN
```

Hermes charges each image-like part (`image_url`, `input_image`, Anthropic-style
`image`) a fixed estimate. The value is 1600 here rather than 1500 because this
path is used for compression-boundary decisions and intentionally leans slightly
conservative.

This solves a real failure mode for creative or browser workflows. Without
image accounting, five image-bearing turns might look like a handful of short
text messages. The compressor would protect them all, then the provider would
receive a request far larger than the estimator predicted.

The algorithm is deliberately rough but conservative:

| Provider shape | Counted as image |
|:----------------|:------------------|
| OpenAI chat style `{type: "image_url"}` | yes |
| Responses API `{type: "input_image"}` | yes |
| Anthropic native `{type: "image"}` | yes |
| Stashed Anthropic blocks | yes in rough message estimator |
| Multimodal tool-result envelopes | yes in rough message estimator; stripped/pruned in compressor |
| Text blocks with `text` | text length only |
| Raw base64 URL payload | not counted directly |

The design rule is simple: estimate images as images, not as serialized bytes.
That avoids both bad extremes:

| Mistake | Result |
|:---------|:--------|
| Count base64 chars as text | Premature compression from huge transport payloads |
| Count images as zero text | Late compression or overprotected image-heavy tails |

---

## 10. Boundary Alignment: Do Not Split Tool Groups

Tool-calling transcripts have a protocol shape:

```text
assistant  { tool_calls: [call_a, call_b] }
tool       { tool_call_id: call_a, content: "..." }
tool       { tool_call_id: call_b, content: "..." }
assistant  { content: "..." }
```

If compression cuts between the assistant tool call and the tool results, the
provider rejects the transcript. If it cuts between tool results, later cleanup
can silently drop evidence.

Hermes uses two boundary aligners:

```python
def _align_boundary_forward(self, messages, idx):
    """Push compress_start forward past any orphan tool results at the head boundary."""
    while idx < len(messages) and messages[idx].get("role") == "tool":
        idx += 1
    return idx

def _align_boundary_backward(self, messages, idx):
    """Pull compress_end backward to avoid splitting an assistant+tool_results group."""
    check = idx - 1
    while check >= 0 and messages[check].get("role") == "tool":
        check -= 1
    # If we landed on the parent assistant with tool_calls, pull the boundary
    # before it so the whole group gets summarized together.
    if check >= 0 and messages[check].get("role") == "assistant" and messages[check].get("tool_calls"):
        idx = check
    return idx
```

After assembly, `_sanitize_tool_pairs()` is the final safety net:

```python
# 1. Remove tool results whose call_id has no surviving assistant tool_call
orphaned_results = result_call_ids - surviving_call_ids
messages = [m for m in messages if not (m.get("role") == "tool"
            and m.get("tool_call_id") in orphaned_results)]

# 2. Insert stub results for surviving assistant tool_calls whose results were dropped
for tc in assistant_msg.get("tool_calls") or []:
    if tc_id in missing_results:
        patched.append({
            "role": "tool",
            "content": "[Result from earlier conversation — see context summary above]",
            "tool_call_id": tc_id,
        })
```

The stub content is explicit: it points the model back to the summary for
semantic context while keeping the transcript wire-valid.

---

## 11. The Active Task Problem

The most subtle bug in context compression is losing the current task.

Imagine the last user message gets summarized into:

```markdown
## Pending User Asks
User asked: "Fix the flaky gateway reconnect test."
```

But the summary prefix tells the model: this summary is reference only; respond
only to messages after the summary. Now the active request is trapped inside a
reference block. The next model may ignore it, repeat older work, or claim there
is no current task.

This is not hypothetical — it was a real bug (issue #10896). `_align_boundary_backward`
can pull `compress_end` past a user message when it tries to keep a
`tool_call`/`tool_result` group together. The user message ends up in the
compressed middle, gets written into `## Pending User Asks`, and disappears from
the live transcript.

Hermes prevents this with `_ensure_last_user_message_in_tail()`:

```python
def _ensure_last_user_message_in_tail(self, messages, cut_idx, head_end):
    last_user_idx = self._find_last_user_message_idx(messages, head_end)
    if last_user_idx < 0 or last_user_idx >= cut_idx:
        return cut_idx   # already in the tail, nothing to do

    # The last user message is in the compressed middle — pull cut_idx back.
    # A user message is already a clean boundary (no tool group splitting risk),
    # so _align_boundary_backward is NOT called here — doing so would
    # unnecessarily pull the cut further back into the preceding tool group.
    return max(last_user_idx, head_end + 1)
```

The call chain in `_find_tail_cut_by_tokens` is:

```
_align_boundary_backward(messages, cut_idx)   # avoid splitting tool groups
    → may accidentally pull past last user message
_ensure_last_user_message_in_tail(...)         # correct that if it happened
```

This design is stricter than "summarize pending asks well." The latest user
message must remain an actual user message in the transcript. The summary may
describe it, but it cannot be the only representation of it.

---

## 12. Summary Prompt Design

`_generate_summary()` uses a structured checkpoint template, not an open-ended
"summarize this conversation" prompt.

### The `_template_sections` Variable — Not a Bug

Before looking at the sections, a common source of confusion in the code:

```python
# Inside _generate_summary():

_template_sections = f"""## Active Task
...
Target ~{summary_budget} tokens. ..."""   # ← defined here, summary_budget interpolated now

if self._previous_summary:
    prompt = f"""...
{_template_sections}"""                   # ← _template_sections referenced here
else:
    prompt = f"""...
{_template_sections}"""                   # ← and here
```

**This is not a self-reference or a bug.** `_template_sections` is a plain local
variable. It is defined once as an f-string (which interpolates `summary_budget`
at that moment), then referenced by name inside two *separate* f-strings that
build `prompt`. Python evaluates `_template_sections` before the outer f-string
is constructed, so the final `prompt` contains the fully-rendered template text.
The variable is reused across both the first-compaction path and the
iterative-update path to avoid duplicating ~60 lines of template text.

The only thing that changes between the two paths is the surrounding context
(previous summary vs. turns to summarize). The template structure itself is
identical — which is intentional: the output format must be stable so the
iterative update prompt can reliably parse and update it.

---

### Template Sections: Use Cases Covered

Each section in `_template_sections` targets a specific failure mode in agent
continuity. The table below maps every section to the use case it covers, the
failure it prevents, and the design choice behind it.

| Section | What it captures | Failure it prevents | Key design choice |
|:---------|:-----------------|:---------------------|:-------------------|
| `## Active Task` | The user's most recent unfulfilled request, verbatim | Agent resumes from wrong task, or treats summarized ask as reference-only and ignores it | Marked "SINGLE MOST IMPORTANT FIELD"; must be verbatim, not paraphrased; "None." if nothing outstanding |
| `## Goal` | The broader objective behind the current task | Agent loses sight of why it is doing the current task; local optimizations contradict the overall goal | Separate from Active Task so short-term work stays anchored to long-term intent |
| `## Constraints & Preferences` | User preferences, coding style, tool choices, explicit constraints | Agent violates user preferences it was told earlier (e.g. "use tabs not spaces", "don't use async") | Preserves user-specific guidance that would otherwise be lost when early turns are compressed |
| `## Completed Actions` | Numbered list: action, target, outcome, tool name | Agent redoes work already done; agent claims success without evidence | Numbered format with `[tool: name]` forces evidence (exact file, line, command, exit code), not narrative claims |
| `## Active State` | Working directory, branch, modified files, test status, running processes | Agent operates on wrong directory or branch; agent doesn't know which files are dirty | Reconstructs the workspace snapshot so the agent can continue without re-probing the environment |
| `## In Progress` | Work underway when compaction fired | Agent abandons half-finished work; agent starts the same work from scratch | Captures the mid-turn state that would otherwise vanish — the work that was happening at the exact moment the context window filled |
| `## Blocked` | Unresolved errors, missing credentials, external dependencies, exact error messages | Agent ignores known blockers and hits the same wall again | Exact error messages preserved so the agent can diagnose rather than rediscover |
| `## Key Decisions` | Technical decisions and the reasoning behind them | Agent re-litigates settled choices; agent makes a decision that contradicts an earlier one | "WHY they were made" is explicit — rationale is as important as the decision itself |
| `## Resolved Questions` | Questions the user asked that were already answered, with the answer | Agent re-answers questions the user already got answers to; wastes a turn | Separates "answered" from "pending" so the agent knows what is settled |
| `## Pending User Asks` | Questions or requests not yet answered or fulfilled | Agent forgets an outstanding user request that was not the most recent one | "None." if empty — forces the summarizer to be explicit rather than leaving the section blank |
| `## Relevant Files` | Files read, modified, or created, with a brief note on each | Agent re-reads files it already processed; agent edits the wrong file | Navigation index — makes file lookup cheap after resume without re-scanning the workspace |
| `## Remaining Work` | What is left to do, framed as context | Agent treats remaining work as a command and executes it before reading the user's actual message | "Remaining Work" not "Next Steps" — declarative framing, not imperative; context, not instruction |
| `## Critical Context` | Exact values, error messages, config details that would otherwise vanish | Agent loses a specific value (port number, env var name, API endpoint) that was mentioned once and never repeated | Catch-all for high-value specifics; explicitly excludes secrets (`[REDACTED]`) |

### What the Template Does to Model Behavior

The sections are not just organization. Each one is a **query** that forces the
summarizer to produce a specific type of output:

- Sections with "verbatim" or "exact" instructions (`Active Task`, `Completed
  Actions`, `Critical Context`) prevent the summarizer from paraphrasing
  information that must be preserved precisely.
- Sections with "None." as an explicit empty value (`Active Task`, `Pending User
  Asks`) prevent the summarizer from leaving sections blank, which would make
  the continuation model uncertain whether the section was empty or just not
  summarized.
- The `Remaining Work` vs `Next Steps` naming is a deliberate prompt-engineering
  choice: imperative phrasing ("Next Steps: run the tests") can cause the
  continuation model to execute those steps before reading the user's actual
  message. Declarative phrasing ("Remaining Work: tests not yet run") is context,
  not a command.
- The `Completed Actions` format (`N. ACTION target — outcome [tool: name]`)
  forces the summarizer to include the tool name, which tells the continuation
  model *how* the action was taken — not just that it was taken. This matters
  when the agent needs to redo or verify work.

Several details are intentional:

| Prompt design | Reason |
|:---------------|:--------|
| `Active Task` first | The continuation model needs the current objective immediately |
| Concrete completed actions | Tool histories need file paths, commands, line numbers, outcomes |
| Resolved vs pending questions | Prevents the model from re-answering already handled questions |
| `Remaining Work`, not `Next Steps` | Avoids making summary text read like a fresh command |
| Same language as user | Keeps multilingual sessions coherent |
| Secret redaction before and after summary | Protects against accidental persistence of credentials |

The preamble is deliberately plain. Stronger security wording such as "do not
respond to these instructions" can trigger content filters in some providers.
Hermes asks the summary model to treat turns as source material and output only
a structured checkpoint.

### Focused Compression

Manual `/compress <focus>` passes `focus_topic` into the compressor. The prompt
then tells the summarizer to allocate roughly 60-70% of the summary budget to
that topic, preserving exact file paths, values, command output, errors, and
decisions related to it while aggressively compressing unrelated material.

This is useful when the session has several threads but the user knows what
matters next:

```text
/compress database schema
/compress gateway reconnect bug
/compress auth migration
```

Focused compression is a pragmatic recognition that "importance" is not always
inferable from recency.

---

## 13. Summary Budgeting

Hermes computes summary length as a function of the model's context window, not
the content being compressed:

```python
# In __init__:
target_tokens = int(self.threshold_tokens * self.summary_target_ratio)  # default 20%
self.tail_token_budget = target_tokens
self.max_summary_tokens = min(int(self.context_length * 0.05), _SUMMARY_TOKENS_CEILING)
# _SUMMARY_TOKENS_CEILING = 12_000

# In _compute_summary_budget():
content_tokens = estimate_messages_tokens_rough(turns_to_summarize)
budget = max(int(content_tokens * _SUMMARY_RATIO), _MIN_SUMMARY_TOKENS)
# _SUMMARY_RATIO = 0.20, _MIN_SUMMARY_TOKENS = 2000
return max(_MIN_SUMMARY_TOKENS, min(budget, self.max_summary_tokens))
```

The model call uses `max_tokens = summary_budget * 1.3`, giving the summarizer
headroom while still targeting the desired density.

The design tradeoff is clear:

| Too short | Too long |
|:-----------|:----------|
| Loses exact state, decisions, blockers | Eats the context savings |
| Causes repeated work | Delays next compression |
| Fails active handoff | Can trigger another overflow |

Hermes chooses a middle path: proportional to compressed content, bounded by
`max_summary_tokens` (5% of context, max 12K), with a 2K floor so short
sessions still get a useful summary.

---

## 14. Iterative Compaction

The second compaction cannot simply summarize the current middle turns from
scratch. Important facts may already live only in the previous summary.

Hermes stores `_previous_summary` and uses an iterative update prompt (see
section 22.5). But there is a subtlety: after a process restart, `_previous_summary`
is empty, but the message list may contain a handoff summary inserted during an
earlier compaction. `_find_latest_context_summary()` rehydrates that state:

```python
def _find_latest_context_summary(self, messages, start, end):
    """Find the newest handoff summary inside a compression window."""
    for idx in range(end - 1, start - 1, -1):
        content = messages[idx].get("content")
        if self._is_context_summary_content(content):
            return idx, self._strip_summary_prefix(content)
    return None, ""
```

In `compress()`, this runs before `_generate_summary()`:

```python
summary_idx, summary_body = self._find_latest_context_summary(
    messages, summary_search_start, compress_end
)
if summary_idx is not None:
    if summary_body and not self._previous_summary:
        self._previous_summary = summary_body   # rehydrate iterative state
    # Only summarize turns AFTER the existing handoff
    turns_to_summarize = messages[max(compress_start, summary_idx + 1):compress_end]
```

This prevents "summary recursion":

```text
[USER]: [CONTEXT COMPACTION - REFERENCE ONLY] ...
```

If that text were treated as ordinary user content, the next summary would
compound the framing and confuse the model. Hermes strips the prefix, treats it
as previous summary state, and only summarizes new turns after it.

---

## 15. Handoff Framing and Role Alternation

The inserted summary starts with `SUMMARY_PREFIX`, a long reference-only marker.
Its job is to tell the continuation model:

1. Earlier turns were compacted.
2. The summary is background reference, not active instructions.
3. Do not fulfill requests mentioned only inside the summary.
4. Resume from `## Active Task`.
5. Persistent memory remains authoritative.
6. Respond to the latest user message after the summary.

### Role Alternation Decision Tree

Providers reject consecutive same-role messages. `compress()` must pick a role
for the summary message that avoids collisions with both the last head message
and the first tail message:

```python
last_head_role  = messages[compress_start - 1].get("role")  # role just before summary
first_tail_role = messages[compress_end].get("role")         # role just after summary

# Priority: avoid colliding with head (already committed)
if last_head_role in {"assistant", "tool"}:
    summary_role = "user"
else:
    summary_role = "assistant"

# If chosen role also collides with tail, try flipping
if summary_role == first_tail_role:
    flipped = "assistant" if summary_role == "user" else "user"
    if flipped != last_head_role:
        summary_role = flipped
    else:
        # Both roles collide — merge summary into first tail message instead
        _merge_summary_into_tail = True
```

When `_merge_summary_into_tail` is True, the summary is prepended to the first
tail message's content with a hard end marker:

```python
merged_prefix = (
    summary
    + "\n\n--- END OF CONTEXT SUMMARY — "
    "respond to the message below, not the summary above ---\n\n"
)
msg["content"] = _append_text_to_content(msg.get("content"), merged_prefix, prepend=True)
```

The same end marker is also appended when the standalone summary uses
`role="user"`, because weaker models may otherwise treat quoted historical user
requests in `## Active Task` as fresh input.

---

## 16. Failure Modes and Recovery

Compression runs when the session is already under pressure, so failure handling
matters.

### No Provider

If there is no auxiliary LLM provider, `_generate_summary()` enters a long
cooldown and returns `None`. `compress()` then inserts a static fallback summary
that says how many messages were removed and that they could not be summarized.

This is not ideal, but it is honest. Silent deletion is worse.

### Broken Auxiliary Model

If `auxiliary.compression.model` fails and differs from the main model, Hermes
retries once on the main model. This covers:

| Error class | Recovery |
|:-------------|:----------|
| 404 / 503 / model not found | Retry on main |
| Timeout / rate limit / gateway failure | Retry on main |
| JSON decode from broken proxy | Retry on main |
| Streaming closed early | Retry on main |
| Unknown aux error | Best-effort retry on main |

The failure is still recorded in `_last_aux_model_failure_*` so CLI/gateway
surfaces can warn the user that their compression model configuration is broken.

### Cooldowns

If compression fails on the final attempted model, Hermes pauses summary attempts
for a short period. JSON decode and premature stream close get shorter cooldowns
because they are often transient. No-provider errors get the long cooldown
because they are configuration problems.

### Static Fallback Marker

When a summary cannot be generated, `compress()` inserts a marker rather than
returning the original overlarge transcript:

```text
Summary generation was unavailable. N message(s) were removed to free context
space but could not be summarized.
```

It also sets:

| Field | Meaning |
|:-------|:---------|
| `_last_summary_fallback_used` | A static marker was inserted |
| `_last_summary_dropped_count` | How many messages were removed |
| `_last_summary_error` | Short error text for user-facing warnings |

This lets gateway hygiene and manual compression report degraded compression
instead of presenting a false success.

---

## 17. Anti-Thrashing

Compression can become pathological. If each pass saves only 1-2%, the runtime
could enter a loop:

```text
request too large -> compress -> still too large -> compress -> still too large
```

Hermes estimates the new transcript size after compression. If savings are under
10%, it increments `_ineffective_compression_count`. After two ineffective
compressions, `should_compress()` refuses automatic compression and suggests a
fresh session or focused compression.

This is a practical guardrail: lossy summarization has diminishing returns.
Eventually the right answer is to reset, branch, or ask the user for a focused
compaction topic.

---

## 18. Security: Redaction Before Persistence

Compression summaries are durable. They can be persisted in session history,
sent to an auxiliary model, and reused across future compactions.

That makes compression a security boundary, not only a context-management
feature. The key helper is `redact_sensitive_text()` in `agent/redact.py`:

```python
def redact_sensitive_text(
    text: str,
    *,
    force: bool = False,
    code_file: bool = False,
) -> str:
    ...
```

Hermes calls it twice in the compression path:

1. `_serialize_for_summary()` redacts message content and tool arguments before
   sending them to the summary model.
2. `_generate_summary()` redacts the summary output in case the summarizer echoed
   a secret despite instructions.

The prompt also explicitly tells the summarizer to replace API keys, tokens,
passwords, credentials, and connection strings with `[REDACTED]`.

### 17.1 Why a Regex Redactor Belongs in the Compressor Path

Summarization is lossy, but it can still preserve exact strings. That is a
feature for file paths and error messages, and a liability for secrets. If a
tool output contains an API key, a database URL, a JWT, or an OAuth callback
URL, the summarizer may copy it into the handoff summary unless the runtime
removes it first.

The redactor is deliberately broad. It catches:

| Secret shape | Examples handled |
|:--------------|:------------------|
| Vendor-prefixed tokens | `sk-...`, `ghp_...`, `github_pat_...`, `xoxb-...`, `AIza...`, `hf_...`, `pypi-...` |
| Environment assignments | `OPENAI_API_KEY=value`, `SLACK_TOKEN=value` |
| JSON fields | `"apiKey": "..."`, `"access_token": "..."`, `"password": "..."` |
| Authorization headers | `Authorization: Bearer ...` |
| Telegram bot tokens | `bot<digits>:<token>` and `<digits>:<token>` |
| Private key blocks | PEM private key sections |
| Database URLs | `postgres://user:pass@host`, Redis, MongoDB, MySQL, AMQP |
| JWTs | `eyJ...` token shapes |
| URL userinfo | `https://user:password@host/...` |
| Query params | `?access_token=...`, `?code=...`, `?signature=...` |
| Form bodies | `client_secret=...&code=...` |
| Platform identifiers | Discord mentions and E.164 phone numbers |

Most long tokens are partially masked, preserving enough prefix/suffix for
debugging. Shorter tokens are fully masked. Private key blocks are replaced with
a fixed `[REDACTED PRIVATE KEY]` marker.

### 17.2 Secure Defaults and Forced Redaction

`redact_sensitive_text()` has two switches that matter for compression design:

| Option | Meaning |
|:--------|:---------|
| `force=True` | Redact even if global log redaction is disabled |
| `code_file=True` | Skip env-assignment and JSON-field passes to reduce false positives in source code |

Compression uses redaction as a safety boundary. In that kind of boundary,
`force=True` is the safer posture because a user may disable log redaction for
debugging, but compression summaries can be persisted and sent to auxiliary
models. A logging preference should not automatically become a data-sharing
preference.

The `code_file` option is a useful design detail. Source code often contains
fixtures like `"apiKey": "test"` or constants like `MAX_TOKENS=...`. Blindly
redacting every key-like assignment in code makes summaries less useful and can
destroy debugging signal. Hermes keeps high-confidence patterns active while
letting source-code-specific callers avoid the noisiest false-positive passes.

### 17.3 Redaction Pipeline Shape

The implementation is a pipeline, not a single regex:

```text
input text
  -> known vendor token prefixes
  -> env assignments, unless code_file=True
  -> JSON secret fields, unless code_file=True
  -> Authorization bearer headers
  -> Telegram bot tokens
  -> private key blocks
  -> database connection strings
  -> JWTs
  -> URL userinfo
  -> sensitive URL query params
  -> form-urlencoded bodies
  -> Discord mentions and phone numbers
  -> redacted text
```

The ordering matters. Prefix patterns catch many common secrets early. URL query
redaction catches opaque values that do not have recognizable token prefixes.
Private key and DB URL passes handle large structured secrets that would be
poorly served by generic token masking.

### 17.4 What This Teaches

For context compression, redaction should be:

- **Pre-model:** redact before sending content to any auxiliary summarizer.
- **Post-model:** redact the summary output too, because models can echo secrets.
- **Shape-aware:** handle URLs, JSON, env dumps, headers, and private keys
  differently.
- **Debuggable:** preserve tiny hints for long tokens when safe.
- **Config-aware but boundary-safe:** user logging preferences should not weaken
  persistence or cross-model safety boundaries.

The broader lesson: summarization is a data exfiltration boundary. Treat it like
one, and make redaction part of the compression algorithm rather than an
afterthought.

---

## 19. Manual Compression and UX

Manual compression is not only a debugging tool. It gives users control over
attention.

CLI and gateway commands can run:

```text
/compress
/compress <focus topic>
```

The focus topic flows into `compress(..., focus_topic=...)`. User-facing
feedback is generated by `agent/manual_compression_feedback.py`, which reports
message counts, approximate token savings, warning state, and whether the
compression was a no-op.

Gateway sessions also run hygiene compression for long-lived chats. That is
important because messaging platforms can keep a session open for days or weeks,
and the user may not know they are approaching the context cliff.

---

## 20. Design Patterns Worth Reusing

### Pattern 1: Deterministic Compression Before LLM Compression

Use exact, cheap transformations first:

```text
dedupe old tool output
summarize tool result metadata
strip image payloads
truncate JSON safely
then call the LLM
```

This reduces cost and makes the LLM's job easier.

### Pattern 2: Preserve the Active Turn Outside the Summary

Do not rely on a summary to carry the current user request. Keep the latest
user turn as a real user message.

### Pattern 3: Structured Handoff, Not Narrative Recap

Agents need operational continuity, not a nice story. Force sections for:

```text
active task
completed actions
active state
blocked items
relevant files
remaining work
critical context
```

### Pattern 4: Validate Protocol Shape After Rewriting

Any compressor for tool-calling models needs a sanitizer. Tool calls and tool
results are paired protocol messages, not ordinary prose.

### Pattern 5: Iteratively Update Summaries

Recursive summarization loses details quickly. Keep a previous summary as state
and ask the summarizer to update it with new turns.

### Pattern 6: Make Degradation Explicit

If summarization fails, insert a marker and expose telemetry. Do not pretend the
summary succeeded.

---

## 21. What the Tests Tell Us

The compressor has targeted regression tests because most bugs only appear in
long sessions:

| Test theme | Behavior protected |
|:------------|:--------------------|
| Summary continuity | Existing handoffs are not serialized as fresh user turns |
| Last user anchoring | Active task does not disappear into reference summary |
| Tool-call integrity | Assistant tool calls and tool results remain provider-valid |
| JSON argument shrinking | Truncated tool arguments remain parseable JSON |
| Multimodal budgeting | Images count toward tail budget |
| Redaction boundaries | Secrets are masked across prefixes, URLs, headers, env dumps, JSON fields, and private keys |
| Auxiliary fallback | Broken compression model retries on main model |
| Failure markers | Summary failure records dropped count and inserts fallback |
| Focus topic | `/compress <focus>` reaches the summary prompt |

This is the right testing shape for context compression: not golden summaries,
but invariants around boundaries, role protocol, active-task preservation, and
failure semantics.

---

## 22. A Reference Algorithm

For another agent system, the Hermes approach can be summarized as:

```python
def compress(messages, token_count, focus=None):
    if too_few_messages(messages):
        return messages

    # Phase 1: deterministic pruning (no LLM)
    messages = deterministic_prune(messages)

    # Phase 2: compute boundaries
    head_end = protect_system_and_first_turns(messages)
    head_end = move_forward_past_orphan_tool_results(messages, head_end)

    tail_start = find_tail_start_by_token_budget(messages, head_end)
    tail_start = move_backward_to_preserve_tool_groups(messages, tail_start)
    tail_start = ensure_latest_user_message_is_in_tail(messages, tail_start, head_end)

    # Phase 3: rehydrate iterative state, then summarize
    previous_summary, turns_to_summarize = find_existing_handoff_and_new_turns(
        messages, head_end, tail_start
    )
    summary = update_or_create_structured_summary(
        previous_summary,
        turns_to_summarize,
        focus,
        budget=proportional_budget(turns_to_summarize),
    )
    if summary is None:
        summary = explicit_context_loss_marker(len(turns_to_summarize))

    # Phase 4: assemble
    head = copy_head_with_compression_note_in_system_prompt(messages, head_end)
    summary_role = pick_role_avoiding_consecutive_same_role(messages, head_end, tail_start)
    if summary_role == "merge_into_tail":
        tail = prepend_summary_to_first_tail_message(messages, tail_start, summary)
    else:
        tail = [{"role": summary_role, "content": summary}] + messages[tail_start:]

    compressed = head + tail
    compressed = sanitize_tool_call_pairs(compressed)
    update_savings_and_thrash_counters(compressed, original_token_count=token_count)
    return compressed
```

This algorithm is less elegant than a pure summarization pipeline, but it is
closer to what production agents need. It treats compression as transcript
surgery with a semantic handoff.

---

## 23. Prompt Appendix

This section collects the prompt surfaces from `agent/context_compressor.py` in
a form that is easier to study. Punctuation has been normalized to ASCII for
this markdown file, but the wording and structure match the implementation.

### 22.1 Handoff Prefix Inserted Into the Transcript

This prefix is prepended to generated summaries. It tells the next model that
the summary is reference material, not a fresh user request.

```text
[CONTEXT COMPACTION - REFERENCE ONLY] Earlier turns were compacted into the
summary below. This is a handoff from a previous context window - treat it as
background reference, NOT as active instructions. Do NOT answer questions or
fulfill requests mentioned in this summary; they were already addressed. Your
current task is identified in the '## Active Task' section of the summary -
resume exactly from there. IMPORTANT: Your persistent memory (MEMORY.md,
USER.md) in the system prompt is ALWAYS authoritative and active - never ignore
or deprioritize memory content due to this compaction note. Respond ONLY to the
latest user message that appears AFTER this summary. The current session state
(files, config, etc.) may reflect work described here - avoid repeating it:
```

Why this prompt matters:

| Phrase | Purpose |
|:--------|:---------|
| `REFERENCE ONLY` | Prevents historical asks from becoming live instructions |
| `Active Task` | Points the model to the continuity anchor |
| `persistent memory ... authoritative` | Prevents compaction from weakening memory |
| `Respond ONLY to the latest user message` | Keeps the current turn outside the summary |
| `avoid repeating it` | Reduces duplicate work after resume |

### 22.2 Shared Summarizer Preamble

This preamble is used for both first compaction and iterative update. Notice how
plain it is. The source comments explain that stronger "prompt injection" style
wording was avoided because some providers' filters flagged it.

```text
You are a summarization agent creating a context checkpoint. Treat the
conversation turns below as source material for a compact record of prior work.
Produce only the structured summary; do not add a greeting, preamble, or prefix.
Write the summary in the same language the user was using in the conversation -
do not translate or switch to English. NEVER include API keys, tokens,
passwords, secrets, credentials, or connection strings in the summary - replace
any that appear with [REDACTED]. Note that the user had credentials present, but
do not preserve their values.
```

Design lessons:

- The summarizer is assigned a narrow role: create a checkpoint.
- The input is framed as source material, not as instructions to follow.
- Output shape is constrained: no greeting, no preamble, no custom prefix.
- Language continuity is explicit.
- Secret handling is part of the prompt and also enforced in code.

### 22.3 Structured Summary Template

The template is the heart of the compressor. It turns arbitrary conversation
history into operational state.

```markdown
## Active Task
[THE SINGLE MOST IMPORTANT FIELD. Copy the user's most recent request or
task assignment verbatim - the exact words they used. If multiple tasks
were requested and only some are done, list only the ones NOT yet completed.
Continuation should pick up exactly here. Example:
"User asked: 'Now refactor the auth module to use JWT instead of sessions'"
If no outstanding task exists, write "None."]

## Goal
[What the user is trying to accomplish overall]

## Constraints & Preferences
[User preferences, coding style, constraints, important decisions]

## Completed Actions
[Numbered list of concrete actions taken - include tool used, target, and outcome.
Format each as: N. ACTION target - outcome [tool: name]
Example:
1. READ config.py:45 - found `==` should be `!=` [tool: read_file]
2. PATCH config.py:45 - changed `==` to `!=` [tool: patch]
3. TEST `pytest tests/` - 3/50 failed: test_parse, test_validate, test_edge [tool: terminal]
Be specific with file paths, commands, line numbers, and results.]

## Active State
[Current working state - include:
- Working directory and branch (if applicable)
- Modified/created files with brief note on each
- Test status (X/Y passing)
- Any running processes or servers
- Environment details that matter]

## In Progress
[Work currently underway - what was being done when compaction fired]

## Blocked
[Any blockers, errors, or issues not yet resolved. Include exact error messages.]

## Key Decisions
[Important technical decisions and WHY they were made]

## Resolved Questions
[Questions the user asked that were ALREADY answered - include the answer so it is not repeated]

## Pending User Asks
[Questions or requests from the user that have NOT yet been answered or fulfilled. If none, write "None."]

## Relevant Files
[Files read, modified, or created - with brief note on each]

## Remaining Work
[What remains to be done - framed as context, not instructions]

## Critical Context
[Any specific values, error messages, configuration details, or data that would be lost without explicit preservation. NEVER include API keys, tokens, passwords, or credentials - write [REDACTED] instead.]

Target ~{summary_budget} tokens. Be CONCRETE - include file paths, command outputs,
error messages, line numbers, and specific values. Avoid vague descriptions like
"made some changes" - say exactly what changed.

Write only the summary body. Do not include any preamble or prefix.
```

The order is deliberate:

| Section | Why it exists |
|:---------|:---------------|
| `Active Task` | Continuation starts from the right request |
| `Goal` | Keeps local work attached to the broader objective |
| `Constraints & Preferences` | Preserves user-specific guidance |
| `Completed Actions` | Gives evidence, not vague progress |
| `Active State` | Reconstructs the workspace after compaction |
| `In Progress` | Captures work interrupted mid-turn |
| `Blocked` | Keeps unresolved errors visible |
| `Key Decisions` | Prevents re-litigating design choices |
| `Resolved Questions` | Prevents duplicate answers |
| `Pending User Asks` | Separates unresolved asks from history |
| `Relevant Files` | Makes navigation cheap after resume |
| `Remaining Work` | Contextual next work without imperative phrasing |
| `Critical Context` | Preserves exact values that would otherwise vanish |

### 22.4 First-Compaction Prompt

When there is no previous summary, Hermes creates a checkpoint from scratch:

```text
{summarizer_preamble}

Create a structured checkpoint summary for the conversation after earlier turns
are compacted. The summary should preserve enough detail for continuity without
re-reading the original turns.

TURNS TO SUMMARIZE:
{content_to_summarize}

Use this exact structure:

{template_sections}
```

The important phrase is "without re-reading the original turns." The summary is
not an abstract. It is a replacement working record.

### 22.5 Iterative-Update Prompt

When a previous compaction already exists, Hermes updates that summary instead
of starting over:

```text
{summarizer_preamble}

You are updating a context compaction summary. A previous compaction produced
the summary below. New conversation turns have occurred since then and need to
be incorporated.

PREVIOUS SUMMARY:
{previous_summary}

NEW TURNS TO INCORPORATE:
{content_to_summarize}

Update the summary using this exact structure. PRESERVE all existing information
that is still relevant. ADD new completed actions to the numbered list
(continue numbering). Move items from "In Progress" to "Completed Actions" when
done. Move answered questions to "Resolved Questions". Update "Active State" to
reflect current state. Remove information only if it is clearly obsolete.
CRITICAL: Update "## Active Task" to reflect the user's most recent unfulfilled
request - this is the most important field for task continuity.

{template_sections}
```

This prompt solves three compaction problems:

| Problem | Prompt mechanism |
|:---------|:------------------|
| Summary drift | Preserve relevant existing information |
| Duplicate progress | Continue the completed-actions numbering |
| Stale active task | Force `Active Task` to the newest unfulfilled request |

### 22.6 Focus-Topic Prompt

Manual `/compress <focus>` appends this guidance to the summarizer prompt:

```text
FOCUS TOPIC: "{focus_topic}"
The user has requested that this compaction PRIORITISE preserving all
information related to the focus topic above. For content related to
"{focus_topic}", include full detail - exact values, file paths, command
outputs, error messages, and decisions. For content NOT related to the focus
topic, summarise more aggressively (brief one-liners or omit if truly
irrelevant). The focus topic sections should receive roughly 60-70% of the
summary token budget. Even for the focus topic, NEVER preserve API keys, tokens,
passwords, or credentials - use [REDACTED].
```

This is a practical prompt feature. Recency is not always the same as
importance, and the user may know which thread matters next.

### 22.7 Fallback Summary Marker

If summary generation fails, Hermes still has to shrink the transcript. It uses
an explicit marker instead of silently dropping the middle:

```text
[CONTEXT COMPACTION - REFERENCE ONLY] Earlier turns were compacted ...

Summary generation was unavailable. {n_dropped} message(s) were removed to free
context space but could not be summarized. The removed messages contained
earlier work in this session. Continue based on the recent messages below and
the current state of any files or resources.
```

The marker is intentionally blunt. It tells the model and the user-facing
runtime that continuity is degraded.

### 22.8 End Marker for User-Role Summaries

When the summary must be inserted as a `user` role, Hermes appends a separator
so weaker models do not treat historical quoted requests as new requests:

```text
--- END OF CONTEXT SUMMARY - respond to the message below, not the summary above ---
```

This is a small prompt-engineering guardrail around a real role-protocol
constraint. The compressor cannot always choose the semantically ideal role
because it must avoid invalid consecutive-role patterns for provider messages.

---

## 24. Learning Checklist

Use this checklist when evaluating or building a context compressor:

| Question | Hermes answer |
|:----------|:---------------|
| What triggers compression? | Token thresholds, context errors, payload errors, manual command |
| What remains verbatim? | System prompt, configured head, latest token-budget tail |
| What gets summarized? | The middle region after boundary alignment |
| How are tool results handled? | Deterministic prune first, protocol sanitizer after |
| How is the active task preserved? | Latest user message is forced into the protected tail |
| How are images counted? | Fixed token estimate per image-like part |
| How are repeated compactions handled? | Previous summary is updated, not re-summarized as a turn |
| How are secrets handled? | `redact_sensitive_text()` runs before the summarizer call and after summary output |
| What secret shapes are covered? | Vendor prefixes, env/JSON keys, auth headers, private keys, DB URLs, JWTs, URL params, form bodies |
| What if summarization fails? | Explicit fallback marker plus warning metadata |
| What stops loops? | Savings tracking and ineffective-compression backoff |

---

## 25. Expert Insights: What This Design Actually Teaches

The sections above describe *what* Hermes does. This section is about *why* the
choices were made and what they reveal about the harder problems in agent design.

---

### 25.1 The Fundamental Tension: Compression Is Lossy, But Agents Need Exact State

Every compression system faces this: you are trading fidelity for space. For
chatbots, that tradeoff is acceptable — a paraphrased conversation history is
fine. For agents, it is dangerous. An agent that misremembers a file path, a
test failure count, or a decision rationale will redo work, make wrong
assumptions, or silently corrupt state.

Hermes resolves this tension not by making compression lossless (impossible) but
by **forcing the summarizer to produce evidence, not claims**. The structured
template demands:

```
## Completed Actions
1. PATCH config.py:45 — changed `==` to `!=` [tool: patch]
2. TEST `pytest tests/` — 3/50 failed: test_parse, test_validate [tool: terminal]
```

Not: "Fixed a bug in config.py and ran tests." The format forces the model to
preserve the exact artifact (file, line, command, outcome) rather than
paraphrasing it. This is the most important prompt-engineering decision in the
entire compressor. The template is not just organization — it is a forcing
function against lossy abstraction.

**Takeaway for your own agent:** When you design a compression prompt, ask
yourself: does this template force the summarizer to preserve exact values, or
does it allow paraphrase? Paraphrase is the enemy of agent continuity.

---

### 25.2 The 50% Threshold Is Not Arbitrary — And It Can Kill You If Set Wrong

The default `threshold_percent = 0.50` means compression fires when the session
reaches half the model's context window. This is carefully chosen:

**Too low (e.g. 20%):** Compression fires constantly. Every few turns, you pay
the summarizer cost, invalidate prompt caches, and introduce lossy state. The
agent spends more time managing its own memory than doing work.

**Too high (e.g. 90%):** By the time compression fires, you are already in
overflow territory. The summarizer itself needs context to read the middle
region. If your main model has a 200K context and your summarizer has a 32K
context, firing at 90% means the middle region alone is ~180K tokens — the
summarizer cannot read it in one call. You have built a compressor that fails
exactly when it is needed most.

**The hidden coupling:** `tail_token_budget = threshold_tokens * summary_target_ratio`.
The tail budget scales with the threshold. Raise the threshold and the tail
grows proportionally. Lower it and the tail shrinks. This means changing the
threshold changes the amount of recent context preserved verbatim — a
non-obvious side effect that can cause the agent to lose recent work if the
threshold is set too aggressively.

**The summarizer context trap:** Hermes validates that the auxiliary compression
model's context window is larger than the compression threshold. If it is not,
the summarizer will fail on exactly the sessions that need compression most. This
validation is easy to skip and catastrophic to miss. Always check it.

---

### 25.3 Why `_previous_summary` Lives In Memory, Not In the Transcript

This is a subtle but critical design choice. The previous summary is stored as
`self._previous_summary` — in-memory instance state — not as a message in the
conversation history.

If it were in the transcript, it would be inside the compression window on the
next pass. The iterative update prompt would then summarize a summary, which
would summarize a summary of a summary. Each pass would lose more detail. After
three or four compressions, the handoff would be a vague narrative with no
concrete state.

By keeping `_previous_summary` in memory and using `_find_latest_context_summary()`
to rehydrate it from the transcript on restart, Hermes breaks the recursion
cycle. The summarizer always sees: "here is the previous checkpoint, here are
the new turns since then — update the checkpoint." It never sees: "here is a
summary of a summary, summarize it again."

**The rehydration path matters too.** After a process restart, `_previous_summary`
is empty. `_find_latest_context_summary()` scans the transcript for the handoff
marker and restores the state. Without this, the first compression after a
restart would treat the existing handoff as an ordinary user message and
summarize it — exactly the recursion problem the in-memory design was meant to
prevent.

**Takeaway:** In any iterative summarization system, the previous summary must
be state, not history. The moment it enters the conversation as a message, it
becomes subject to the next compression pass.

---

### 25.4 The Role Alternation Problem Reveals a Deeper Issue

The role alternation decision tree (section 15) looks like a provider
compatibility hack. It is actually a symptom of a deeper problem: **the
compressor is inserting a synthetic message into a conversation it did not
author**.

Every provider's message format assumes a natural conversation: user asks,
assistant responds, user asks again. The compressor breaks this assumption by
injecting a summary message that has no natural conversational role. It is not
a user message (the user didn't write it) and not an assistant message (the
assistant didn't generate it in response to a user turn). It is a synthetic
artifact.

The role alternation logic is the compressor trying to fit a square peg into a
round hole. The merge-into-tail path (prepending the summary to the first tail
message) is the most honest solution: it acknowledges that the summary is not a
standalone conversational turn and attaches it to the nearest real message.

**The broader lesson:** Any time you inject synthetic messages into a
conversation — for compression, for tool results, for system notes — you are
fighting the model's expectation of natural conversation flow. The more
synthetic messages you inject, the more the model's behavior diverges from its
training distribution. Design synthetic messages to be as invisible as possible,
and always test what happens when they land in unexpected roles.

---

### 25.5 The Structured Template Is Attention Steering, Not Just Organization

The 13-section template is often read as "good organization." It is actually
**attention steering** — a way of directing the summarizer's attention to the
information that matters for agent continuity, not the information that is most
salient in the conversation.

Consider what a naive summarizer would produce for a long coding session: a
narrative of what was discussed, with the most recent and most dramatic events
weighted most heavily. That is useful for a human reader but terrible for an
agent. The agent needs:

- The exact current task (not a paraphrase of it)
- The exact files modified (not "some files were changed")
- The exact test failures (not "tests were run")
- The exact blockers (not "there were some issues")

The template forces the summarizer to produce this operational state by making
each section a specific query: "what is the current task?", "what files were
modified?", "what tests failed?". The model cannot answer these with vague
narrative — the section headers demand specifics.

**`Remaining Work` vs `Next Steps`** is the clearest example of this. "Next
Steps" reads as an instruction: "do these things." "Remaining Work" reads as
context: "this is what is left." The difference matters because the continuation
model reads the summary before responding to the user. If the summary says "Next
Steps: run the tests," the model may run the tests before reading the user's
actual message. "Remaining Work: tests not yet run" is context, not a command.

---

### 25.6 The `focus_topic` Feature Is Solving the Recency Bias Problem

The default compression algorithm is recency-biased: it protects the most recent
turns and summarizes the older ones. This is correct for most sessions but wrong
for sessions with multiple concurrent threads.

Imagine a session where the agent has been working on three things: a database
migration, a UI bug, and a deployment script. The user's last few messages were
about the UI bug. Default compression will protect the UI bug context and
aggressively compress the database migration work — even if the database
migration is the most important thread.

`/compress database migration` tells the summarizer to allocate 60-70% of its
budget to that topic. This is not just a convenience feature. It is an
acknowledgment that **recency ≠ importance in long agent sessions**, and that
the user is the only reliable signal for what matters next.

The implementation is simple (append a paragraph to the prompt), but the insight
is significant: importance is a user-defined property, not an algorithmic one.
Any compression system that ignores user intent will eventually compress away
the wrong thing.

---

### 25.7 Compression Invalidates Prompt Caches — This Is a Cost Event

When compression fires, it rewrites the message history. Every cache breakpoint
that Anthropic (or any caching provider) had established for the old history is
now invalid. The next API call re-processes the entire compressed history at
full input token cost.

For a 200K-token session on Claude, a single compression event can cost
$0.30–$0.60 in cache misses (at ~$3/MTok input, ~75% cache hit rate). This is
not a performance issue — it is a billing event. In a gateway serving hundreds
of concurrent sessions, compression events can cause sudden cost spikes that
look like billing anomalies.

**The implication for threshold tuning:** Setting the threshold lower (compress
earlier, more often) means more frequent cache invalidations. Setting it higher
(compress later, less often) means each compression event is more expensive
because the history is larger. The optimal threshold is not just about context
management — it is a cost optimization problem.

Hermes does not currently model this tradeoff explicitly. A production system
serving many users should track compression frequency and cache miss rates
together, not separately.

---

### 25.8 The Auxiliary Model Quality Tradeoff Is Underappreciated

Using a cheap/fast model for compression (e.g. `gemini-3-flash` instead of
`claude-opus`) saves money on the summarizer call. But the quality of the
summary directly determines whether the agent can continue working after
compression.

A poor summary that loses a critical file path, misremembers a test failure, or
drops a blocker will cause the agent to redo work, make wrong assumptions, or
get stuck. The cost of that lost work — in API calls, user time, and
frustration — almost always exceeds the savings from using a cheaper summarizer.

**The right mental model:** The summarizer is not a cost center. It is a
reliability investment. The question is not "what is the cheapest model that can
produce a summary?" but "what is the cheapest model that can produce a summary
good enough that the agent can continue working correctly?"

For most sessions, a capable mid-tier model (Gemini Flash, Claude Haiku) is
sufficient. For sessions with complex technical state — many files modified,
intricate test failures, multi-step decisions — the main model is the right
choice for compression, even at higher cost.

Hermes exposes `summary_model_override` and falls back to the main model when
the auxiliary model fails. The fallback is correct. The default should be
evaluated per use case.

---

### 25.9 The Anti-Thrashing Threshold of 2 Is Aggressive — Here Is Why

After 2 consecutive compressions each saving less than 10%, `should_compress()`
stops firing. This is correct behavior but the threshold is aggressive.

Consider a session that is genuinely near its irreducible minimum — the system
prompt is large, the active task is complex, and the tail is full of recent
tool outputs that cannot be compressed further. Two passes each saving 8% is
not thrashing — it is the compressor doing its job on a dense session. But
Hermes will stop after those two passes and leave the session at risk of
overflow.

The right behavior in this case is to surface the situation to the user: "This
session is near its compression limit. Consider starting a fresh session or
using `/compress <focus>` to prioritize what matters most." Hermes does log a
warning, but the warning is easy to miss in a long-running gateway session.

**The deeper issue:** Anti-thrashing is a heuristic for "compression is not
working." But "not working" has two causes: (1) the session is genuinely dense
and cannot be compressed further, and (2) the summarizer is producing poor
summaries that don't actually reduce token count. These require different
responses. Hermes treats them the same way.

---

### 25.10 Compression as a Training Data Boundary

If you save agent trajectories for fine-tuning, compressed sessions look
structurally different from uncompressed ones. The `SUMMARY_PREFIX` marker, the
synthetic summary message, and the stub tool results are all artifacts of
compression that do not appear in natural conversations.

If you train on compressed trajectories without filtering or marking them, you
teach the model that `[CONTEXT COMPACTION — REFERENCE ONLY]` is a normal
conversational pattern. The model may start generating this text in its
responses, or may treat it as a signal to behave differently.

Hermes uses `ephemeral_system_prompt` to exclude certain content from
trajectories. The same principle should apply to compressed sessions: either
exclude them from training data, or mark them explicitly so the training
pipeline can handle them differently.

**The broader principle:** Any synthetic artifact you inject into the
conversation — compression markers, tool stubs, system notes — is a potential
training signal. Design these artifacts to be either invisible to the training
pipeline or explicitly labeled.

---

### 25.11 The Real Lesson: Compression Is a Runtime State Machine

The most important insight in the entire compressor is not any individual
algorithm. It is the framing in section 1: compression is a **state transition
in the agent runtime**, not a model trick.

This framing has concrete consequences:

- The output must be a valid provider transcript (wire-format validity).
- The active task must survive (semantic continuity).
- Tool call/result pairs must be intact (protocol validity).
- The model must know the transcript was rewritten (epistemic honesty).
- Failures must be visible (operational transparency).

A system that treats compression as "call an LLM to summarize the chat" will
fail on all five of these. It will produce invalid transcripts, lose active
tasks, break tool protocols, confuse the model about its own history, and hide
failures.

The Hermes compressor is complex because the problem is complex. Every piece of
complexity — the boundary aligners, the tool pair sanitizer, the active task
anchor, the role alternation logic, the iterative summary state — exists because
someone hit a real failure and had to fix it.

When you build your own compressor, start with the simplest possible version and
add complexity only when you hit the specific failure it prevents. The Hermes
implementation is a map of the failures you will encounter.

---

## 26. Key Takeaways

The compressor is one of the clearest examples of Hermes' agent design
philosophy: model calls are only one piece of the system. The durable behavior
comes from the surrounding control plane.

The good ideas are:

- Summarize the middle, not the live task.
- Protect by token budget, not message count.
- Account for image tokens explicitly.
- Shrink tool data deterministically before asking an LLM to summarize.
- Use a structured handoff with active state, completed actions, blockers, and
  critical context.
- Treat previous summaries as state, not as new conversation turns.
- Preserve provider message validity after every rewrite.
- Surface degraded compression instead of hiding it.

The result is a compressor that does more than make the prompt shorter. It lets
Hermes keep working across long sessions without asking the model to remember
what no longer fits.