---
layout: post
title: "What Claude Code's Source Code Taught Me About Building Production AI Agents"
date: "2026-05-04"
author: "Jamie Zhang"
tags: ["Agentic AI", "LLM", "AI Agents", "Tool Use"]
categories: ["AI"]
description: "A Staff Engineer's deep-dive into Claude Code's source code — the architectural patterns, trust boundaries, and self-healing mechanisms that make a production CLI agent work."
image: "/img/ai-01.jpg"
keywords: ["Claude Code", "AI agent architecture", "LangGraph", "tool orchestration", "agentic AI", "Zod schema", "MCP"]
---

## The Problem with "AI Agent" Blog Posts

Most content about AI agents describes patterns in the abstract. Tools and loops explained with boxes and arrows. Mermaid diagrams that look clean but tell you nothing about how it actually works under the hood.

This post is different. I read the source code.

Specifically, I spent time with the `src/` directory of Anthropic's Claude Code — a production-grade CLI agent — and worked through a deep course that traces every concept to its actual file, function, and line number. I didn't run it (missed too many libs), but the learning notes told the full story.

What I found was a remarkably well-engineered system. And if you're building agentic software at Staff Engineer level, some of its patterns are worth stealing.

---

## The Architectural Mental Model: Inversion of Control

The first thing to unlearn is imperative control flow.

In a conventional application, you write the `if/else`. You call `fs.readFile()`, parse the result, conditionally branch. You own the sequence.

In an agentic application, the **LLM is the control flow**. Your job is to build a safe, robust **harness** for it to operate within.

```
Conventional:  You → Write Code → Define Steps → Execute
Agentic:       You → Define Goal + Tools → LLM decides steps → Execute
```

The harness does four things:
1. Serializes available tools into JSON schemas
2. Provides a sandbox and security boundary
3. Feeds LLM requests to the OS, OS results back to the LLM
4. Loops until the LLM says "I'm done"

The architectural actors:

| Actor | Role | Location |
|---|---|---|
| **The Brain** | External LLM (Claude) — reasoning | `api.anthropic.com` |
| **The Engine** | Run loop, session state as a state machine | `src/QueryEngine.ts` |
| **The Harness** | Tool registry + execution + security boundary | `src/tools/`, `src/Tool.ts` |
| **The Trust Boundary** | Action verification before execution | `bashPermissions.ts`, `bashSecurity.ts` |

The key insight: **the harness is more than a list of tools**. It encompasses tool registration, Zod-based runtime validation, permission verification, environment injection (`ToolUseContext`), and concurrency control. Everything between the LLM's intent and your machine's reality.

---

## The Tool Contract: Zod as Documentation and Validation

`src/Tool.ts` is the architectural keystone. Every built-in tool, every MCP-bridged tool, and every sub-agent tool conforms to the same `Tool<Input, Output, P>` interface.

```typescript
export type Tool<Input, Output, P> = {
  readonly name: string
  readonly inputSchema: Input  // Zod schema — validated at runtime, sent as JSON Schema to API
  call(args: z.infer<Input>, context: ToolUseContext, canUseTool: CanUseToolFn, parentMessage: AssistantMessage): Promise<ToolResult<Output>>
  checkPermissions(input: z.infer<Input>, context: ToolUseContext): Promise<PermissionResult>
  validateInput?(input: z.infer<Input>, context: ToolUseContext): Promise<ValidationResult>
  isReadOnly(input: z.infer<Input>): boolean
  isConcurrencySafe(input: z.infer<Input>): boolean
}
```

**Zod does double duty.** In most systems, you validate incoming data and document your API separately. Here, the `inputSchema` is a runtime Zod schema that:

1. **Validates** the LLM's JSON at runtime (catches hallucinated fields, wrong types)
2. **Documents** the tool for the LLM via JSON Schema serialization

One schema, two purposes. The `.describe()` strings on each field become the **LLM's instruction manual** — prompt engineering embedded in the schema definition.

```typescript
command: z.string().describe('The command to execute'),
timeout: semanticNumber(z.number().optional()).describe(`Optional timeout in ms (max ${getMaxTimeoutMs()})`),
```

`z.strictObject()` rejects any extra fields the LLM hallucinates. If the LLM sends `{ command: "ls", extraField: true }`, validation fails immediately — before any permission check or execution.

### Dependency Injection Without a Framework

No DI container. Instead, a single `ToolUseContext` object is threaded through every `call()`:

```typescript
export type ToolUseContext = {
  options: { tools: Tools; mcpClients: MCPServerConnection[]; mainLoopModel: string; thinkingConfig: ThinkingConfig }
  abortController: AbortController
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  readFileState: FileStateCache
  messages: Message[]
  agentId: AgentId | null
}
```

When `runAgent.ts` spawns a child agent, it creates a **new context** with an isolated `abortController`, a cloned `readFileState`, and a no-op `setAppState`. This is how you get isolation without process forking.

### Fail-Closed Defaults via `buildTool()`

`buildTool()` is a factory that enforces safe defaults:

```typescript
const TOOL_DEFAULTS = {
  isConcurrencySafe: () => false,  // Assume NOT safe — conservative
  isReadOnly: () => false,          // Assume it writes data — stricter perms
  isDestructive: () => false,
  checkPermissions: (input) => Promise.resolve({ behavior: 'allow', updatedInput: input }),
}
```

Notice `isReadOnly` defaults to `false`. If a developer writes a benign tool but forgets to set `isReadOnly`, the system treats it as a destructive write and may prompt the user. **The failure mode is friction, not a security hole.**

---

## The Run Loop: Self-Healing via Error Strings

`src/query.ts` (~1,700 lines) contains the actual `while(true)` loop that calls the Anthropic API. Each iteration:

1. Pre-process messages (context compaction, truncate oversized tool results)
2. Call API — `deps.callModel()` abstracts the HTTP call for testability
3. Stream response — process `tool_use` blocks as they arrive
4. Execute tools — partition by concurrency safety, run in parallel or serial
5. Decide: continue re-entering the loop, or return `end_turn`

**The self-healing pattern is critical:**

> [!WARNING]
> **Don't throw exceptions to stop the flow.** In a typical app, if a file isn't found, you throw an error and a global handler returns HTTP 500. In an agent loop, if `FileReadTool` fails, it **catches the error and returns it as a string** in the `tool_result`. The LLM reads the error and decides to fix the path or try a different approach. The loop *continues*.

The error recovery patterns:
- **Prompt too long** → Reactive emergency compaction of message history
- **Max output tokens** → Retry up to 3 times with extended budget
- **Streaming failure** → Tombstone orphaned partial responses, reset state, retry with fallback model

`turnCount` and `maxTurns` act as circuit breakers to prevent infinite loops.

---

## Tool Orchestration: Concurrency Is Input-Dependent

When the LLM returns multiple `tool_use` blocks in one response, `partitionToolCalls()` splits them into batches. **Not all tools are safe to run in parallel.** Reading two files simultaneously is fine. Running `mkdir` and then `cd` into it is not.

The key insight: **concurrency safety is input-dependent**. `BashTool.isConcurrencySafe(input)` delegates to `checkReadOnlyConstraints(input)` — analyzing the actual shell command. `ls` is concurrent-safe. `rm` is not. The same tool type can go into different batches depending on its arguments.

```typescript
// Example partitioning
LLM requests: [FileRead, FileRead, Bash(mkdir), FileRead]

Batch 1: { isConcurrencySafe: true,  blocks: [FileRead, FileRead] }  → Parallel
Batch 2: { isConcurrencySafe: false, blocks: [Bash(mkdir)] }          → Serial
Batch 3: { isConcurrencySafe: true,  blocks: [FileRead] }            → Parallel
```

### StreamingToolExecutor: Latency Optimization

A more sophisticated executor starts tool execution **while the LLM is still streaming**. If the LLM emits a FileRead block early in its response, the file is already read by the time the stream ends.

When a Bash tool errors during parallel execution, it **cancels all sibling tools** — because Bash commands often have implicit dependencies (`mkdir dir && cd dir`). If `mkdir` fails, `cd` is pointless. Other tool failures are isolated.

---

## BashTool: Shell Execution with Defensive Engineering

`BashTool` (~1,100 lines) is the most complex built-in tool. Several patterns stand out:

### AST Parsing via Tree-sitter

Regex-based command validation fails against injection. `r\m`, `"r"m`, `$(echo rm)`, `eval $(base64 -d)` all bypass simple pattern matching.

Claude Code uses **Tree-sitter** (`tree-sitter-bash`) to parse commands into a full AST. The security engine walks the AST to detect dynamic execution (`CommandSubstitution`, `ProcessSubstitution` nodes), strip wrapper commands (`timeout`, `nice`, `nohup`), and decompose compound commands into individually-validated leaf nodes.

### Output Protection

A single Bash command outputting 500KB would blow the LLM's context window. `BashTool` enforces `maxResultSizeChars` (30,000 chars). When truncated, the full output is persisted to `/tool-results/{uuid}.txt` and the LLM receives a preview + path. This forces the LLM to paginate or `grep` rather than crashing the system.

### Auto-Backgrounding

If a synchronous command exceeds 15 seconds, the engine automatically detaches it to prevent UI freeze. The LLM receives: *"Command exceeded the assistant-mode blocking budget (15s) and was moved to the background with ID: X."*

---

## The Permission System: Trust Boundaries

Every tool invocation must pass a dual-phase check:

1. `validateInput()` — Logical validation (Is the path formatted correctly?)
2. `checkPermissions()` — Trust validation (Does the user consent?)

These are **separate phases with different failure modes**. Both return strings to the LLM (not exceptions). The LLM can self-correct after a validation error; it adapts strategy after a permission denial.

### The Hybrid Model

For commands that don't match explicit rules, a deterministic + heuristic hybrid is used:
- **Deterministic rules** (fast, exact): always-allow/deny lists, read-only auto-allow
- **Heuristic classifier** (for unknowns): normalizes the AST, checks known patterns

Without the heuristic, every unfamiliar command would trigger a UI prompt — permission fatigue destroys user trust.

### Environment Variable Safety

`PATH` and `LD_PRELOAD` are deliberately excluded from the safe environment variable allowlist. If `PATH` were allowed, the LLM could run `PATH=/attacker/bin npm install` and match an `npm:*` allow-rule while executing malicious code.

---

## MCP: Dynamic Tool Registration at Runtime

The Model Context Protocol answers: **How do you add tools to an agent without recompiling?**

MCP servers are separate processes. The agent connects at runtime, asks "What tools do you have?", and registers them dynamically into the tool registry.

```typescript
// MCP tools are wrapped to conform to the standard Tool interface
const wrappedTool: Tool = {
  name: `mcp__${serverName}__${tool.name}`,  // Namespace collision prevention
  inputSchema: convertJsonSchemaToZod(tool.inputSchema),  // JSON Schema → Zod
  call: async (args, context) => {
    const result = await mcpClient.callTool(tool.name, args)
    return { data: result.content }
  },
  isReadOnly: () => false,
  isConcurrencySafe: () => true,
}
```

Configuration lives in `.mcp.json` with environment variable expansion for secrets. The agent watches this file and hot-reloads connections on save — no restart needed.

### MCP vs Built-in Tools

| Aspect | Built-in Tool | MCP Tool |
|---|---|---|
| Registration | Compiled into agent | Discovered at runtime |
| Schema | Zod (TypeScript-native) | JSON Schema (language-agnostic) |
| Execution | In-process function call | Cross-process JSON-RPC |
| Language | Must be TypeScript | Any (Python, Go, Rust...) |

---

## Multi-Agent: Sub-Agents as Tools

A sub-agent is just another tool. From the parent's perspective, calling `AgentTool` is no different from calling `BashTool`.

```typescript
// When LLM calls AgentTool, runAgent.ts:
// 1. Creates a child AbortController (aborting child doesn't kill parent)
// 2. Filters tool set (sub-agents can't spawn more sub-agents by default)
// 3. Creates isolated ToolUseContext (no-op setAppState, cloned file cache)
// 4. Calls QueryEngine.ask() — creates a new engine instance with fresh history
// 5. Returns final answer string as tool_result
```

Depth limits and budget splitting prevent runaway recursion:

```
Parent budget: $1.00 remaining
  → Child A gets: $0.30 max
  → Child B gets: $0.30 max
  → Parent retains: $0.40 for recovery
```

The parent never sees the sub-agent's internal message history — only the final answer string. Full transcripts are persisted to a sidechain log for debugging, keeping the parent's context window clean.

---

## What This Means for Staff Engineers

If you're evaluating agent frameworks or building agentic systems, these patterns are worth understanding at the implementation level — not just the concept level:

**1. Zod as the tool contract is a best practice.** The dual-use of schema-as-validation and schema-as-documentation removes a whole category of drift between what the tool does and what the LLM thinks it does.

**2. Self-healing loops require explicit design.** You can't just wrap everything in try-catch. Errors must be returned as strings the LLM can reason about, and the loop must have circuit breakers (maxTurns, budget caps).

**3. Concurrency safety is per-invocation, not per-tool-type.** The `isConcurrencySafe()` method receives the actual input, not just the tool definition. A read-only `grep` is parallelizable; a write-capable `tee` is not.

**4. Security must be at the AST level.** Regex patterns for command validation are bypassable. Tree-sitter AST analysis is the correct model.

**5. Permission systems need a heuristic layer.** Pure allow/deny lists cause prompt fatigue. The hybrid model balances strict security with usable UX.

**6. Multi-agent architectures need isolation at every layer.** Abort controllers, state mutation, file caches, message history — all must be isolated per agent instance, not just at the process boundary.

---

## Further Reading

- [LangGraph 2026 Architecture Patterns](/) — LangGraph's approach to stateful agent orchestration
- [Model Context Protocol Documentation](https://modelcontextprotocol.io) — MCP specification and server implementations
- Source code: [Anthropic's Claude Code](https://github.com/anthropics/claude-code) — the reference implementation this post is based on

---

*This post is based on study notes from the Claude Agent Course (`claude-agent-course`), a practitioner's deep-dive into Claude Code's source code. The course covers Module 0–8 + deep-dives across Tool.ts, QueryEngine, tool orchestration, permissions, MCP, and multi-agent patterns.*
