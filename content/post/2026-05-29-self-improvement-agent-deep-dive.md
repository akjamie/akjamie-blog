---
layout: post
title: "The Self-Improvement Agent in Hermes: A Deep Dive"
date: "2026-05-29"
author: "Jamie Zhang"
tags: ["Hermes", "Self-Improvement", "Durable Skills", "Procedural Memory", "Agentic AI"]
categories: ["AI"]
description: "Staff-engineer-level notes on Hermes Agent's procedural self-improvement loop: how the runtime turns solved problems, user corrections, and hard-won debugging paths into durable skills without interrupting the foreground task."
image: "/img/ai-01.jpg"
keywords: ["self-improvement agent", "procedural memory", "skills creation", "curator", "background review", "provenance"]
---

# The Self-Improvement Agent in Hermes: A Deep Dive

> Staff-engineer-level notes on Hermes Agent's procedural self-improvement
> loop: how the runtime turns solved problems, user corrections, and hard-won
> debugging paths into durable skills without interrupting the foreground task.


---

> [!NOTE]
> ### Executive TL;DR
> Hermes self-improvement is not a magic `auto_create_skill()` function. It is a
> runtime pattern composed from five pieces:
>
> - **Foreground guidance:** the main agent is told to save or patch skills when
>   a complex workflow succeeds.
> - **Iteration counters:** the runtime tracks how much tool-heavy work has
>   happened since the last skill review.
> - **Background review fork:** after the user-visible answer is delivered,
>   Hermes starts a quiet review agent with the conversation transcript.
> - **Narrow tool whitelist:** the review fork may call only memory and skill
>   tools, not arbitrary shell, web, or browser tools.
> - **Procedural memory tool:** `skill_manage` writes, patches, deletes, and
>   annotates skills under Hermes' skill library.
>
> The architecture matters more than the prompt. The learning loop is isolated
> from the foreground answer, bounded by tool permissions, marked with
> provenance, and implemented through the same skill API the main agent can use.

---

## How to Read This Article

This blog is written for engineers designing production agents, not for users
looking for the `/skills` command.

Read it in four passes:

| Pass | Sections | What to learn |
|:------|:----------|:---------------|
| Conceptual | 1-3 | What Hermes means by self-improvement |
| Runtime | 4-8 | How the loop fires and how the review fork runs |
| Persistence | 9-12 | How skills are written, protected, and curated |
| Design | 13-18 | The architectural lessons and tradeoffs |

If you only want the implementation map, read sections 4, 5, 6, 9, and 16.
If you are designing your own learning agent, read sections 2, 13, 15, and 18.

---

## 1. What "Self-Improvement" Means in Hermes

The phrase "self-improvement" is easy to overstate. Hermes does not retrain its
model. It does not edit its own Python source after every session. It does not
blindly archive every conversation as a permanent instruction.

Hermes improves by maintaining **procedural memory**.

Procedural memory is different from user memory:

| Store | Captures | Example |
|:-------|:----------|:---------|
| User/profile memory | Who the user is and durable preferences | "The user prefers concise final answers." |
| Session history | What happened in previous conversations | "In session X, pytest failed due to missing env var." |
| Skills | How to do a recurring class of task | "When debugging gateway slash commands, inspect command registry first." |

The self-improvement loop is the mechanism that keeps the third store alive.
When a task exposes a useful workflow, a pitfall, a correction, or a reusable
debugging path, Hermes can update a skill so future sessions start with that
knowledge already in the prompt.

That is the key design move: Hermes improves **the operating procedure around
the model**, not the model weights.

---

## 2. Why Skills Are the Right Unit of Learning

An agent can learn at several granularities:

1. Save raw transcripts.
2. Save facts into memory.
3. Save task summaries.
4. Save executable procedures.
5. Rewrite its source code.

Hermes chooses option 4 for most self-improvement.

Raw transcripts are too noisy. Facts are too weak for procedures. Summaries are
useful for recall but not action. Source rewrites are high risk and require
code review. Skills sit in the middle: structured enough to guide future work,
plain-text enough to inspect, and narrow enough to patch.

A good skill answers:

- When should this skill trigger?
- What sequence should the agent follow?
- What tools should it prefer?
- What pitfalls has experience revealed?
- How should the agent verify success?
- What supporting references, templates, or scripts exist?

This makes skill creation a form of operational knowledge capture. Hermes is
not trying to "remember everything"; it is trying to preserve the parts of a
session that change future behavior.

---

## 3. The High-Level Architecture

At a high level, self-improvement looks like this:

```text
User turn
  |
  v
Foreground AIAgent handles the task
  |
  |-- uses tools, reads files, debugs, edits, tests
  |
  v
Final response is delivered to the user
  |
  v
Runtime checks memory/skill review triggers
  |
  v
Background AIAgent fork receives the transcript
  |
  |-- tool whitelist: memory + skills only
  |-- prompt: review conversation and update memory/skills
  |
  v
Review fork calls skill_manage(...)
  |
  v
Skill library is patched or extended
  |
  v
Future sessions load improved skill guidance
```

There are two important separations:

1. **Learning happens after the answer.** The user does not wait for the review
   fork before getting the response.
2. **Learning writes through a tool.** The review fork does not mutate files by
   inventing ad hoc shell commands. It calls `skill_manage`, the same procedural
   memory API the rest of Hermes uses.

This is what makes the design production-shaped rather than demo-shaped.

---

## 4. Where the Logic Lives

If you search for `auto_create_skill` in `run_agent.py`, you will not find it.
The capability is distributed across a few explicit runtime surfaces:

| File | Responsibility |
|:------|:----------------|
| `agent/prompt_builder.py` | Foreground guidance that tells the main agent when to save or patch skills |
| `run_agent.py` | Counters, review prompts, background review fork, trigger points |
| `toolsets.py` | Exposes `skills_list`, `skill_view`, and `skill_manage` in the core toolset |
| `tools/skill_manager_tool.py` | Implements skill creation, patching, supporting files, deletion, and telemetry |
| `tools/skill_provenance.py` | Marks whether a write came from foreground work or background review |
| `tools/skill_usage.py` | Tracks agent-created skills and patch activity for curator workflows |
| `agent/curator.py` | Periodic consolidation and lifecycle management for accumulated skills |

The absence of one central function is intentional. Self-improvement crosses
the agent loop, tool authorization, prompt design, persistence, and curation.
Packing that into one "create skill" routine would hide the real system
boundaries.

---

## 5. The Foreground Nudge

The first layer is not background automation. It is the main system prompt.

`agent/prompt_builder.py` contains `SKILLS_GUIDANCE`, which tells the agent:

```text
After completing a complex task (5+ tool calls), fixing a tricky error,
or discovering a non-trivial workflow, save the approach as a skill with
skill_manage so you can reuse it next time.

When using a skill and finding it outdated, incomplete, or wrong, patch it
immediately with skill_manage(action='patch')...
```

This matters because the foreground agent has the richest local context. It
knows what it just discovered, what it tried, what failed, and what finally
worked. If the task naturally produces a reusable method, the foreground agent
can save it immediately.

But foreground learning has a weakness: it competes with the user's task. If
the agent is trying to finish a bug fix, forcing it to stop and author a skill
can distract from delivery. Hermes therefore also has the background pass.

The two modes are complementary:

| Mode | Best for |
|:------|:----------|
| Foreground `skill_manage` | Obvious, user-requested, or immediately relevant skill updates |
| Background review | Retrospective cleanup after the user-visible work is complete |

---

## 6. The Skill Nudge Counter

Hermes does not run a skill review after every tiny turn. It uses a cadence.

In `AIAgent.__init__`, `run_agent.py` initializes:

```python
self._skill_nudge_interval = 10
skills_config = _agent_cfg.get("skills", {})
self._skill_nudge_interval = int(
    skills_config.get("creation_nudge_interval", 10)
)
```

The default means: after roughly ten tool-calling iterations since the last
skill review, consider running a background skill review.

This is a pragmatic signal. Tool-heavy work is where procedural knowledge tends
to emerge:

- multiple file reads and patches
- failed test loops
- provider quirks
- CLI invocations
- recovery from bad assumptions
- user corrections during implementation

Hermes increments `_iters_since_skill` during the agent loop. When the counter
reaches the configured threshold and `skill_manage` is available, it flips a
review flag and resets the counter.

That last condition is important:

```text
only review skills if "skill_manage" is in valid_tool_names
```

If the agent does not have the skill management tool, it does not pretend it
can improve the skill library.

---

## 7. Triggering After the Final Response

The trigger happens after the foreground turn is complete.

In the normal conversation path, `run_agent.py` checks:

```python
if (
    self._skill_nudge_interval > 0
    and self._iters_since_skill >= self._skill_nudge_interval
    and "skill_manage" in self.valid_tool_names
):
    _should_review_skills = True
    self._iters_since_skill = 0
```

Then, only after final response handling:

```python
if final_response and not interrupted and (
    _should_review_memory or _should_review_skills
):
    self._spawn_background_review(
        messages_snapshot=list(messages),
        review_memory=_should_review_memory,
        review_skills=_should_review_skills,
    )
```

There is a similar block for the newer turn abstraction path, where
`turn.tool_iterations` is added to `_iters_since_skill` and the same background
review is spawned if the threshold fires.

This is a subtle but important design:

- no final response, no learning pass
- interrupted turn, no learning pass
- no skill tool, no skill pass
- review runs after delivery, not before

That avoids recording half-finished work as durable procedure.

---

## 8. The Background Review Fork

The heart of the system is `_spawn_background_review(...)` in `run_agent.py`.

It creates a second `AIAgent` in a background thread. This review agent is not a
subroutine inside the main model call. It is a separate model invocation with
the completed transcript as `conversation_history` and a review prompt appended
as the next user message.

Conceptually:

```python
review_agent = AIAgent(
    model=self.model,
    max_iterations=16,
    quiet_mode=True,
    platform=self.platform,
    provider=self.provider,
    api_mode=parent_api_mode,
    base_url=parent_base_url,
    api_key=parent_api_key,
    credential_pool=self._credential_pool,
    parent_session_id=self.session_id,
)

review_agent.run_conversation(
    user_message=review_prompt,
    conversation_history=messages_snapshot,
)
```

Several implementation details are doing real work here:

| Detail | Why it matters |
|:--------|:----------------|
| `quiet_mode=True` | The review fork should not look like a second conversation |
| `max_iterations=16` | Learning is bounded and cannot run forever |
| inherited runtime credentials | The fork uses the same provider/model context as the parent |
| copied session id/start | Cache keys and session attribution stay aligned |
| status output suppression | Review warnings do not leak into the user's UI |
| background thread approval callback | Dangerous command prompts auto-deny instead of blocking stdin |

The fork is deliberately boring. It is just another Hermes agent, but with a
different goal and a tighter toolbox.

---

## 9. The Review Prompt Is Policy, Not Just Prompting

The `_SKILL_REVIEW_PROMPT` in `run_agent.py` is long because it encodes product
policy, not just model instruction.

It tells the review fork to look for:

- user corrections to style, tone, format, verbosity, or workflow
- non-trivial techniques, fixes, workarounds, or debugging paths
- loaded skills that proved wrong, stale, or incomplete
- reusable support files such as references, templates, and scripts

It also tells the fork where to put the learning:

1. Patch a currently loaded skill.
2. Patch an existing umbrella skill.
3. Add a support file under an existing umbrella.
4. Create a new class-level umbrella skill only when nothing fits.

The "umbrella" language is doing architectural work. Without it, agents tend to
create one skill per incident:

```text
bad:
  fix-pytest-error-may-29
  debug-run-agent-skill-nudge
  patch-discord-command-alias-today

better:
  python-test-debugging
  hermes-agent-runtime
  gateway-command-routing
```

The review prompt also contains negative guidance. Do not save:

- missing binaries
- unconfigured credentials
- fresh install errors
- transient failures that resolved
- one-off task narratives
- broad negative claims like "browser tools do not work"

That last category is especially important. A self-improving agent can poison
itself with stale prohibitions. If a tool failed because of a temporary setup
issue, recording "never use this tool" is worse than not learning at all.

My view: this is one of the most mature parts of the design. Good learning
systems need anti-learning rules. The hard problem is not only deciding what to
remember; it is preventing accidental permanent constraints.

---

## 10. Tool Whitelisting: The Safety Boundary

The review fork inherits the transcript, but it does not inherit unlimited
power.

Inside `_spawn_background_review`, Hermes computes a whitelist from:

```python
get_tool_definitions(
    enabled_toolsets=["memory", "skills"],
    quiet_mode=True,
)
```

Then it installs a thread-local whitelist:

```python
set_thread_tool_whitelist(
    review_whitelist,
    deny_msg_fmt=(
        "Background review denied non-whitelisted tool: {tool_name}. "
        "Only memory/skill tools are allowed."
    ),
)
```

This is a major architectural choice. The review agent is allowed to write
memory and skills. It is not allowed to run shell commands, browse the web,
open browsers, modify project files, send messages, or execute arbitrary
plugins.

That makes the review fork a **knowledge maintenance agent**, not a second
worker agent.

This is the boundary I would preserve in any production implementation. If a
background learning pass can run arbitrary tools, it becomes another agentic
task runner, and now the user has two agents competing for side effects.

---

## 11. `skill_manage`: Procedural Memory as a Tool

The persistence API is `tools/skill_manager_tool.py`.

Its module docstring says the key idea plainly:

```text
Skills are the agent's procedural memory: they capture how to do a specific
type of task based on proven experience.
```

The tool supports:

| Action | Purpose |
|:--------|:---------|
| `create` | Create a new skill directory and `SKILL.md` |
| `edit` | Rewrite an existing `SKILL.md` |
| `patch` | Targeted find-and-replace in `SKILL.md` or support file |
| `delete` | Remove/archive a skill |
| `write_file` | Add reference, template, script, or asset files |
| `remove_file` | Remove a support file |

The `create` path validates the name, category, frontmatter, content size, and
name collision. It writes atomically, optionally scans the generated skill, and
returns a structured JSON result.

That matters because skills are not just prompt snippets. They are file-system
artifacts with layout, metadata, support files, and lifecycle management.

The default layout is:

```text
~/.hermes/skills/
  my-skill/
    SKILL.md
    references/
    templates/
    scripts/
    assets/
```

This layout lets the agent keep the `SKILL.md` concise while moving bulky
session details, fixtures, and reusable scripts into support directories.

---

## 12. Provenance: Background-Created vs User-Directed

Hermes tags write origin at the start of each `run_conversation()` call:

```python
set_current_write_origin(
    getattr(self, "_memory_write_origin", "assistant_tool")
)
```

The background review fork sets:

```python
review_agent._memory_write_origin = "background_review"
review_agent._memory_write_context = "background_review"
```

Then `skill_manage` checks that provenance:

```python
if action == "create":
    if is_background_review():
        mark_agent_created(name)
```

This distinction is easy to miss but very important.

A foreground skill creation may be user-directed: "Create a skill for this."
That is the user's artifact. A background-created skill is agent-generated
procedural memory. The curator can treat those differently.

This is the kind of metadata that looks optional until you operate the system
for months. Without provenance, you cannot safely answer questions like:

- Which skills did the agent invent by itself?
- Which skills are user-authored?
- Which skills are safe candidates for auto-consolidation?
- Which skills should be protected from cleanup?

Self-improvement without provenance becomes unmaintainable accumulation.

---

## 13. Why the Review Fork Runs After the User Response

The timing is not incidental. Learning after response has three benefits:

1. **Latency isolation.** The user gets the answer when the work is done, not
   when the learning pass is done.
2. **Attention isolation.** The foreground model focuses on solving the task,
   not simultaneously producing a polished reusable procedure.
3. **State quality.** The review sees the complete arc: initial request,
   failed attempts, final solution, and user-visible outcome.

There is a cost: the background pass may miss nuance that the foreground model
had in its working context. Hermes compensates by passing the full message
snapshot and allowing foreground skill saves when obvious.

This is a common production pattern: put the optional quality-improvement work
on the side path, but keep a manual/foreground escape hatch for high-signal
events.

---

## 14. Why the Review Fork Is a Full `AIAgent`

Hermes could have implemented self-improvement as a plain LLM call:

```text
summarize this transcript into a skill update
```

Instead, it creates another `AIAgent`.

That gives the review pass:

- normal tool calling
- the same memory/skill dispatch system
- the same provider runtime
- the same session-aware logging
- the same error handling
- the same structured tool result messages

The downside is complexity. A forked agent has to suppress status output,
inherit credentials, avoid deadlocking on approvals, and avoid recursively
triggering more review passes.

Hermes handles the recursion issue explicitly:

```python
review_agent._memory_nudge_interval = 0
review_agent._skill_nudge_interval = 0
```

That prevents "review of the review" cascades.

My architectural read: using a full agent is the right call because the thing
being built is not a summary string. It is a controlled mutation to a durable
knowledge store. Tool semantics matter.

---

## 15. The Security Model

Self-improvement is a write path. Write paths need boundaries.

Hermes uses several:

| Boundary | Implementation |
|:----------|:----------------|
| Tool availability | Review fires only if `skill_manage` is valid |
| Tool whitelist | Background review can only use memory/skills |
| Action schema | `skill_manage` requires structured action arguments |
| Path validation | Support files must stay under allowed subdirectories |
| Atomic writes | Skill files are written via temp file + replace |
| Optional scanner | `skills.guard_agent_created` can scan agent-written skills |
| Provenance | Background-created skills are marked separately |
| Curator lifecycle | Agent-created skills can later be consolidated or pruned |

There is an honest comment in `tools/skill_manager_tool.py`: the agent-created
skill scanner is off by default because the same agent can already execute code
through other tools. Turning on a scanner can add friction and false positives
without changing the fundamental trust boundary.

That is a pragmatic security stance. The strongest boundary is not the scanner;
it is the background review whitelist. The review fork cannot run terminal
commands in the first place.

---

## 16. How the Skill Becomes Active Later

Creating a skill is only half the loop. The next session has to see it.

`toolsets.py` includes the skills tools in `_HERMES_CORE_TOOLS`:

```python
"skills_list", "skill_view", "skill_manage"
```

When the system prompt is built, `run_agent.py` checks whether skills tools are
available:

```python
has_skills_tools = any(
    name in self.valid_tool_names
    for name in ["skills_list", "skill_view", "skill_manage"]
)
```

If skills are available, it calls `build_skills_system_prompt(...)`. That
prompt includes the skill ecosystem and gives the model access to skill lookup
and loading behavior.

So the loop is:

```text
session N discovers technique
  -> background review creates/patches skill
  -> skill prompt cache is cleared
  -> future session sees the updated skill library
  -> agent can load/apply the improved procedure
```

`skill_manage` clears the skills system prompt cache after successful mutation.
That avoids the classic bug where the agent writes a skill but future prompt
construction still sees a stale skill snapshot.

---

## 17. Curation: Self-Improvement Needs Maintenance

If every successful session creates a skill, the library eventually rots:

- duplicates appear
- narrow one-off skills pile up
- old guidance becomes stale
- overlapping umbrellas compete
- prompt listings grow noisy

Hermes has a separate curator layer in `agent/curator.py` and related CLI
commands. The background review prompt even mentions this:

```text
If you notice two existing skills that overlap, note it in your reply --
the background curator handles consolidation at scale.
```

This is another mature design point. Self-improvement is not only write-time
capture. It is lifecycle management:

| Phase | Question |
|:-------|:----------|
| Creation | Is this learning worth saving? |
| Patch | Did a new pitfall refine an existing procedure? |
| Support files | Should bulky detail move out of `SKILL.md`? |
| Consolidation | Do two skills cover the same class? |
| Archival | Is an agent-created skill stale or unused? |
| Pinning | Did the user mark a skill as protected? |

Learning systems need garbage collection. Otherwise, the "improved" prompt
becomes a landfill of obsolete advice.

---

## 18. Design Insights

### Insight 1: Self-improvement should be procedural, not autobiographical

The worst version of an agent memory system saves every story about what
happened. That feels like learning, but it forces future agents to re-derive
the lesson from narrative.

Hermes pushes toward class-level skills. The saved artifact should say:

```text
When debugging this kind of problem, do these steps.
```

not:

```text
On Friday, the user asked X, then I tried Y, then Z happened.
```

Autobiography is useful as reference material. Procedure belongs in `SKILL.md`.

### Insight 2: Anti-learning is as important as learning

The review prompt's "Do NOT capture" section is not defensive fluff. It prevents
permanent damage.

Agents are prone to overgeneralizing failures:

- "npm failed once" becomes "do not use npm"
- "browser setup was broken" becomes "browser is unavailable"
- "credentials were missing" becomes "this integration does not work"

Good self-improvement systems need rules for what not to encode. Otherwise,
they slowly accumulate fear.

### Insight 3: Background learning must be side-effect constrained

The review fork can mutate memory and skills, but not arbitrary files. That is
the right boundary.

A learning pass should not be able to continue the user's engineering task. It
should only update the knowledge layer that helps future tasks.

This is the difference between:

```text
background reviewer
```

and:

```text
unsupervised second agent
```

The first is operationally safe. The second creates race conditions and trust
problems.

### Insight 4: Prompt design carries product policy

The self-improvement prompt encodes taste:

- prefer patching existing skills
- avoid session-specific names
- use support files for detail
- treat user frustration as high-signal
- avoid environment-specific prohibitions

These are product decisions. They shape the knowledge base over time. In an
agent system, prompts like this are not decoration; they are policy modules.

### Insight 5: Provenance is not optional

Once agents can create knowledge artifacts, you need to know where each artifact
came from.

User-authored, bundled, hub-installed, foreground-created, and
background-created skills should not all be treated identically. Hermes marks
background-created skills so curator workflows can manage them without
confusing them with user-owned content.

### Insight 6: Cache invalidation is part of learning

The learning loop would be broken if the agent wrote a skill and then continued
serving stale skill prompts. `skill_manage` clears the skill prompt cache after
successful mutations.

This is a small implementation detail with big consequences. Durable learning
has to reach the next prompt construction path.

---

## 19. Failure Modes to Watch

If you build a similar system, these are the failure modes I would test first:

| Failure | Symptom | Mitigation |
|:---------|:---------|:------------|
| Over-capture | Too many tiny skills | Prefer umbrella patching before creation |
| Under-capture | Hard lessons never saved | Foreground guidance plus background cadence |
| Stale prohibitions | Agent refuses tools based on old failures | Explicit anti-learning rules |
| Review latency | User waits for skill writing | Spawn review after final response |
| Review side effects | Background agent modifies task files | Tool whitelist |
| Recursive learning | Review triggers another review | Disable nudge intervals in fork |
| Skill cache staleness | New skills not visible later | Clear prompt cache on mutation |
| Provenance loss | Curator cannot tell ownership | Mark background-created skills |
| Prompt bloat | Skill list becomes noisy | Curator consolidation and archival |

The subtle one is over-capture. A self-improving agent that writes too much can
get worse. The knowledge base becomes noisy, contradictory, and expensive to
load. The best learning loop is selective.

---

## 20. A Minimal Implementation Recipe

If you wanted to implement the Hermes pattern in another agent runtime, the
minimal version would look like this:

1. Define a procedural knowledge artifact with metadata and support files.
2. Expose a structured `knowledge_manage` tool for create, patch, and support
   file writes.
3. Add foreground guidance telling the agent when to save or patch procedures.
4. Track a signal correlated with hard work, such as tool iterations.
5. After final response, spawn a quiet review agent with the transcript.
6. Whitelist the review agent to memory/procedure tools only.
7. Use a review prompt that strongly prefers patching existing procedures.
8. Mark provenance for background-created artifacts.
9. Clear any prompt/index caches after mutation.
10. Run periodic curation to merge, archive, and repair accumulated knowledge.

The key is to treat self-improvement as a **runtime subsystem**, not as a
single prompt.

---

## 21. Why This Design Feels Different from "Memory"

Many agent frameworks have memory. Fewer have procedural maintenance.

Memory usually answers:

```text
What should I know about this user or past session?
```

Skills answer:

```text
How should I operate when this class of work appears again?
```

That difference changes the shape of the artifact. A memory can be a sentence.
A skill needs trigger conditions, steps, pitfalls, tools, and verification.

Hermes' self-improvement loop recognizes that solved work is often more
valuable as a reusable method than as a remembered fact.

---

## 22. Final Takeaway

Hermes self-improvement is not the agent deciding to rewrite itself. It is a
disciplined loop for converting experience into inspected, structured,
procedural knowledge.

The architecture has a clear philosophy:

- keep user delivery fast
- learn after the task is complete
- write through explicit tools
- constrain background side effects
- prefer improving existing knowledge over creating new fragments
- track provenance
- curate the library over time

That is the practical version of self-improvement for production agents. Not
"the model gets smarter" in the abstract, but "the system gets better at doing
recurring work because yesterday's hard-earned procedure is available tomorrow."

---

*Hermes Agent - May 2026*
