---
layout: post
title: "Observability & Trace Collection: Spring Boot & OpenTelemetry Engineering Reference"
date: "2026-06-25"
author: "Jamie Zhang"
tags: ["Spring Boot", "OpenTelemetry", "Micrometer", "Tracing", "Langfuse", "DeepSeek", "LLM Observability"]
categories: ["AI"]
description: "A staff-engineer-level reference on trace collection, data masking, and fallback handling using Spring Boot 4.1, Micrometer, and OpenTelemetry to Langfuse."
image: "/img/ai-01.jpg"
---

# Observability & Trace Collection — Engineering Reference

> **Audience:** Staff engineers and senior contributors who need to understand the exact
> implementation mechanics, call chains, and design rationale — not the "what" but the
> "how" and "why".

---

> [!NOTE]
> ### Executive TL;DR
> This system implements LLM observability and trace collection using a pure **OpenTelemetry (OTel) bridge** integrated with Spring Boot 4.1 and the Micrometer Observation API, bypassing the Langfuse SDK entirely.
> Key architecture highlights:
> - **Zero Langfuse SDK overhead**: Pure standard OTLP/HTTP protobuf export to Langfuse.
> - **Selective tracing**: [TracingObservationPredicate](https://github.com/akjamie/code-review-orchestrator/blob/main/src/main/java/org/akj/reviewer/config/TracingObservationPredicate.java) gates and drops standard Spring Boot system/web spans, focusing exclusively on AI and pipeline operations.
> - **Secure data handling**: [SensitiveDataMaskingFilter](https://github.com/akjamie/code-review-orchestrator/blob/main/src/main/java/org/akj/reviewer/config/SensitiveDataMaskingFilter.java) intercepts telemetry on observation stop, stripping credentials and PII via concurrent-safe regex mapping.
> - **Robust streaming fallbacks**: [LangfuseObservationHandler](https://github.com/akjamie/code-review-orchestrator/blob/main/src/main/java/org/akj/reviewer/config/LangfuseObservationHandler.java) acts as a last-resort fallback to capture token outputs and tool call parameters for streaming or tool-call-only models.

---

## Terminology

Three terms appear throughout this doc and are often conflated. Precise definitions in the
context of this stack:

| Term | Layer | Definition |
|:------|:-------|:------------|
| **Trace** | Langfuse / OTel | A single end-to-end operation — in this system, one PR review. Contains a tree of spans sharing the same `traceId`. Langfuse shows a trace as a top-level row in the UI with its own `input` and `output`. |
| **Span** | OTel | A named, timed unit of work within a trace — one LLM call, one tool invocation, or the root `review-pipeline` itself. Spans nest via `parentSpanId`. Each span carries key-value attributes (e.g. `gen_ai.request.model`, `langfuse.observation.input`). |
| **Observation** | Micrometer | The Micrometer-layer abstraction that maps 1:1 to an OTel `Span` after the bridge converts it. In code we work with `Observation` and `Observation.Context`; by the time data reaches Langfuse it is a span. Think of `Observation` as the mutable staging object before it is sealed and exported as an immutable `Span`. |

The relationship: `Observation.stop()` → Micrometer bridge → `OTel Span` sealed →
`BatchSpanProcessor` queues it → `OtlpHttpSpanExporter` sends it → Langfuse stores it and
groups spans by `traceId` to reconstruct the trace tree.

---

## Technology Stack & Binding Points

```
Spring Boot 4.1 / Micrometer Observation API
  └─ spring-boot-starter-opentelemetry  ← bridges Micrometer → OTel SDK
       └─ OTel SDK (BatchSpanProcessor)
            └─ OtlpHttpSpanExporter     ← OTLP/HTTP protobuf to Langfuse
                 └─ Langfuse            ← http://localhost:3000/api/public/otel
```

**No Langfuse SDK is used.** The entire integration is pure OpenTelemetry. Spring Boot's
auto-configuration wires the `MicrometerObservationAutoConfiguration` bridge that converts
Micrometer `Observation` → OTel `Span` before export.

Spring AI automatically creates `Observation` instances for every `ChatClient` call using
`ChatModelObservationContext` — the same context our custom components read and enrich.

### DeepSeek Auto-Configuration
The DeepSeek model integration is auto-configured via `spring-ai-autoconfigure-model-deepseek.jar` through `DeepSeekChatAutoConfiguration`:
- **Properties**: Binds configurations under the `spring.ai.deepseek.chat.*` namespace (e.g. `temperature`, `model`).
- **Instantiation**: Creates the `DeepSeekChatModel` bean and injects the `ObservationRegistry` directly into its builder.
- **Tracing Mechanism**: Observability is baked natively inside the model class (`DeepSeekChatModel`). When `call()` or `stream()` is invoked, it creates observations dynamically using `ChatModelObservationDocumentation.CHAT_MODEL_OPERATION` without requiring external decorators or proxies.

---

## Full Call Chain (per LLM invocation)

```
HTTP Request ──► ReviewPipeline.execute()
                  │
                  ├─① ReviewContextHolder.set(userId, sessionId)   ← ThreadLocal
                  │
                  ├─② Observation.start("review-pipeline", registry) ← root span
                  │     └─ Context.root().makeCurrent()              ← isolate from HTTP span
                  │
                  └─③ orchestrator.runAgents(...)  ── Virtual Threads ──►
                          │
                          └─ BaseReviewAgent.review()
                                └─ ChatClient.call()
                                     │
                                     └─[Spring AI internally]
                                        ChatModelObservationContext.start()
                                             │
                                             ▼
                                    ┌──────────────────────────────────────────┐
                                    │      Micrometer Observation Pipeline      │
                                    │                                           │
                                    │  GATE    TracingObservationPredicate      │
                                    │          .test(name, ctx) → true/false    │
                                    │                                           │
                                    │  FILTER  SensitiveDataMaskingFilter       │
                                    │          .map(ctx) → enriched ctx         │
                                    │          (runs before handlers, on stop)  │
                                    │                                           │
                                    │  HANDLER LangfuseObservationHandler       │
                                    │          .onStop(ctx)                     │
                                    │          (runs after all handlers)        │
                                    │                                           │
                                    │  HANDLER OTel bridge                      │
                                    │          Observation → OTel Span          │
                                    │          → BatchSpanProcessor             │
                                    │          → OtlpHttpSpanExporter           │
                                    └──────────────────────────────────────────┘
```

---

## Class-by-Class Implementation Breakdown

### `TracingObservationPredicate` — Gate
- **Source Link:** [TracingObservationPredicate.java](https://github.com/akjamie/code-review-orchestrator/blob/main/src/main/java/org/akj/reviewer/config/TracingObservationPredicate.java)
- **Interface:** `io.micrometer.observation.ObservationPredicate`
- **Role:** Filters incoming observations to eliminate tracing noise.

```java
@Override
public boolean test(String name, Observation.Context context) {
    return name != null && (
        name.startsWith("spring.ai.") ||   // ChatClient, ChatModel, tool spans
        name.startsWith("gen_ai.")     ||   // OTel GenAI semantic conventions
        name.equals("review-pipeline")      // Custom root trace
    );
}
```

**Effect:** Prevents standard web server request or database query spans from polluting the AI-focused telemetry backend. Only the core pipeline operations and LLM actions proceed.

---

### Trace/Span Decision Logic Hierarchy
Every core component in Spring AI (models, advisors, tool callbacks) initializes observations unconditionally.
- **ChatModel Spans**: Emitted with name `gen_ai.client.operation` (or `spring.ai.chat`) inside the model call path.
- **Advisor Spans**: Emitted with name `spring.ai.advisor` during chat client advisor executions.
- **Tool Spans**: Emitted with name `gen_ai.tool_call` when model functions are invoked.

`TracingObservationPredicate` acts as a selective filter/gate. Spans that do not match the predicate pattern are demoted to no-ops at creation time, bypassing downstream filters, handlers, and exporter queues.

---

### `SensitiveDataMaskingFilter` — Enrichment & Masking
- **Source Link:** [SensitiveDataMaskingFilter.java](https://github.com/akjamie/code-review-orchestrator/blob/main/src/main/java/org/akj/reviewer/config/SensitiveDataMaskingFilter.java)
- **Interface:** `io.micrometer.observation.ObservationFilter`
- **Role:** Performs context enrichment (injecting user metadata and capturing inputs/outputs) and redacting PII or secrets.

#### Enrichment Flow
1. **Input / Output Mapping**: Reads `ChatModelObservationContext` or tool argument metadata and populates standard span attributes:
   - Sets `langfuse.observation.input` to the prompt text or tool call JSON arguments.
   - Sets `langfuse.observation.output` to the completion text or tool call results.
2. **Metadata Injection**: Resolves user and session identifiers from the current context and maps them to `langfuse.user.id` and `langfuse.session.id`.

#### Masking Logic & Concurrency Guard
To avoid exposing sensitive information, the filter iterates through sensitive keys (such as prompts, completions, and arguments) and applies a series of masking rules for PEM private keys, GitHub PATs, AI keys (`sk-***`), Bearer tokens, basic auth credentials, and email addresses.

Because `context.getHighCardinalityKeyValues()` is a live mutable collection in some Micrometer versions, mutating it directly throws a `ConcurrentModificationException`. To safely update attributes, a snapshot-then-update approach is required:

```java
// Safe update strategy inside map(...)
var keysToRemove = context.getHighCardinalityKeyValues().stream()
        .map(KeyValue::getKey)
        .toList(); // snapshot
keysToRemove.forEach(context::removeHighCardinalityKeyValues);
updated.forEach(context::addHighCardinalityKeyValue);
```

---

### `LangfuseObservationHandler` — Last-Resort Output Fallback
- **Source Link:** [LangfuseObservationHandler.java](https://github.com/akjamie/code-review-orchestrator/blob/main/src/main/java/org/akj/reviewer/config/LangfuseObservationHandler.java)
- **Interface:** `io.micrometer.observation.ObservationHandler<Observation.Context>`
- **Role:** Ensures completions and tool payload data are consistently captured, even during non-standard lifecycles.

#### Fallback Strategy
During streaming responses, `observation.stop()` frequently fires before the response aggregation is completed. To ensure no data loss, this handler acts as a last-resort callback:

1. **Tier 1 (Filter Win)**: If the filter successfully populated `langfuse.observation.output`, the handler skips processing.
2. **Tier 2 (Completion Copy)**: Copies `gen_ai.completion` to `langfuse.observation.output` if present.
3. **Tier 3 (Direct Response Reading)**: If both are missing, it reads the response stream directly from `ChatModelObservationContext`, parsing both text and tool-call payload JSON structures.

#### Ordering Guarantee
By declaring `LOWEST_PRECEDENCE` and wrapping execution in the handler phase, this handler is guaranteed to execute *after* all of Spring AI's built-ins have populated the observation context.

---

### `ReviewContextHolder` — Cross-Thread Context Propagation
- **Source Link:** [ReviewContextHolder.java](https://github.com/akjamie/code-review-orchestrator/blob/main/src/main/java/org/akj/reviewer/config/ReviewContextHolder.java)
- **Role:** Manages the thread-bound review context.

```java
public class ReviewContextHolder {
    private static final InheritableThreadLocal<ReviewContext> CONTEXT = new InheritableThreadLocal<>();
    // ... set, get, and clear wrappers
}
```

**Virtual Threads Compatibility:**
Because agent orchestration spawns tasks via virtual thread pools (`newVirtualThreadPerTaskExecutor()`), utilizing `InheritableThreadLocal` guarantees that child virtual threads automatically inherit context credentials (user and session data) from the parent request thread at creation time without explicit plumbing.

---

### `ReviewPipeline` — Root Trace Construction
- **Source Link:** [ReviewPipeline.java](https://github.com/akjamie/code-review-orchestrator/blob/main/src/main/java/org/akj/reviewer/orchestrator/ReviewPipeline.java)
- **Role:** Orchestrates the multi-agent execution pipeline.

#### Root Span Isolation Pattern
If an agent initiates an observation, it inherits the active OpenTelemetry context (e.g. the inbound HTTP endpoint span), nesting the entire review pipeline inside it. To display reviews as clean, top-level traces in Langfuse, the pipeline temporarily decouples the active context:

```java
// Temporarily decouple Micrometer's current scope to isolate it from the HTTP span context.
Observation.Scope originalScope = observationRegistry.getCurrentObservationScope();
observationRegistry.setCurrentObservationScope(null);
Observation observation;
try (io.opentelemetry.context.Scope otelScope = io.opentelemetry.context.Context.root().makeCurrent()) {
    observation = Observation.start("review-pipeline", observationRegistry);
} finally {
    observationRegistry.setCurrentObservationScope(originalScope); // Restore HTTP context
}
```

This isolates the `"review-pipeline"` span as the root of its own trace, ensuring logical boundaries are clear in the Langfuse dashboard.

---

## Observation Lifecycle — Exact Invocation Sequence

For a single `ChatClient.call()` within an agent:

```
t=0  ChatModelObservationContext.start()
       ├─ TracingObservationPredicate.test("spring.ai.chat", ctx)
       │    → true  (name starts with "spring.ai.")
       └─ Observation is live; HTTP call to DeepSeek proceeds

t=1  DeepSeek API returns response (or streaming aggregation completes)
       └─ ChatModelObservationContext.setResponse(response)

t=2  observation.stop() called by Spring AI (in doFinally / afterReceive)
       │
       ├─ SensitiveDataMaskingFilter.map(ctx)      ← FILTER PHASE
       │    ├─ extract prompt → langfuse.observation.input
       │    ├─ extract completion (if response != null) → langfuse.observation.output
       │    ├─ inject user/session from ReviewContextHolder
       │    └─ apply masking regexes to all sensitive keys
       │
       ├─ [Spring AI built-in handlers run]        ← HANDLER PHASE (in order)
       │    └─ ChatModelCompletionObservationHandler.onStop() → logs to SLF4J only
       │
       └─ LangfuseObservationHandler.onStop(ctx)   ← LAST HANDLER
            ├─ input fallback: fill langfuse.observation.input if blank
            └─ output 3-tier fallback (Tier 1 usually wins on non-streaming path)

t=3  OTel bridge converts Observation → OTel Span
       └─ All highCardinalityKeyValues become span attributes

t=4  BatchSpanProcessor buffers the span

t=5  (async) OtlpHttpSpanExporter POSTs to
       http://localhost:3000/api/public/otel
       Headers: Authorization=Basic <pk:sk base64>, x-langfuse-ingestion-version=4
```

---

## Span Attribute Mapping to Langfuse Fields

Langfuse reads specific OTel span attributes and maps them to its UI fields:

| OTel Span Attribute | Langfuse Field | Set by |
|:---------------------|:----------------|:--------|
| `langfuse.observation.input` | Span **Input** | `SensitiveDataMaskingFilter` / `LangfuseObservationHandler` |
| `langfuse.observation.output` | Span **Output** | `SensitiveDataMaskingFilter` / `LangfuseObservationHandler` |
| `langfuse.trace.input` | Trace **Input** | `ReviewPipeline` |
| `langfuse.trace.output` | Trace **Output** | `ReviewPipeline` |
| `langfuse.trace.name` | Trace **Name** | `SensitiveDataMaskingFilter` (copies `"review-pipeline"`) |
| `langfuse.user.id` | Trace **User** | `SensitiveDataMaskingFilter` |
| `langfuse.session.id` | Trace **Session** | `SensitiveDataMaskingFilter` |
| `gen_ai.request.model` | Span model label | Spring AI (auto) |
| `gen_ai.usage.input_tokens` | Token counts | Spring AI (auto) |
| `gen_ai.usage.output_tokens` | Token counts | Spring AI (auto) |

---

## Sensitive Key Set (Masking Scope)

The filter only masks attributes whose key is in this exact set:

```java
private static final Set<String> SENSITIVE_KEYS = Set.of(
    "gen_ai.prompt",
    "gen_ai.completion",
    "spring.ai.tool.call.arguments",
    "spring.ai.tool.call.result",
    "langfuse.observation.input",
    "langfuse.observation.output",
    "langfuse.trace.input",
    "langfuse.trace.output"
);
// + any key starting with "gen_ai.prompt." or "gen_ai.completion." (index variants)
```

Low-cardinality attributes (model name, HTTP status, etc.) are **never** masked.

---

## Infrastructure & OTLP Transport

```yaml
# application.yml — no explicit OTLP config needed here.
# Spring Boot reads standard OTel environment variables from the process environment.

management:
  tracing:
    sampling:
      probability: 1.0   # sample every span
```

```bash
# .env — loaded by run.ps1 before starting the JVM
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:3000/api/public/otel
OTEL_EXPORTER_OTLP_HEADERS=Authorization=Basic <base64(pk:sk)>,x-langfuse-ingestion-version=4
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf   # Langfuse does NOT support gRPC OTLP
```

`spring-boot-starter-opentelemetry` reads these via `OtlpAutoConfiguration` at startup —
no code-level configuration required.

The Langfuse self-hosted backend (`docker-compose.langfuse.yml`):

| Service | Port | Role |
|:---------|:------|:------|
| `langfuse-server` | `3000` | OTLP ingest + UI |
| `langfuse-worker` | — | Async event processing (ClickHouse writes) |
| `postgres` | `5432` | Project/session metadata |
| `clickhouse` | `8123` | Observation time-series storage |
| `minio` | `9091` | Large payload object store (S3-compatible) |
| `redis` | — | Worker job queue |

---

## Key Design Decisions — Rationale

| Decision | Alternative considered | Why this choice |
|:----------|:----------------------|:-----------------|
| `ObservationFilter` for prompt extraction | `ObservationHandler` | Filters run before handlers and have write access to the context; handlers see the final state |
| `InheritableThreadLocal` for context propagation | Passing `ReviewContext` as method arg | Virtual threads inherit it automatically; no plumbing changes needed across agent call chain |
| Root trace isolation via `Context.root().makeCurrent()` | Letting the HTTP span be the parent | Makes each PR review a top-level trace in Langfuse, not buried inside an HTTP span tree |
| 3-tier fallback in handler | Single write point in filter | Filter misses streaming completions; handler's `onStop` fires after stream aggregation is complete |
| `LOWEST_PRECEDENCE` on both filter and handler | Custom numeric order | Filter and handler pipelines are independent; `LOWEST_PRECEDENCE` in each ensures Spring AI built-ins run first in both pipelines |
| OTLP/HTTP protobuf, no Langfuse SDK | Langfuse Python/JS SDK, or OpenLLMetry | Zero SDK dependency; standard OTel is stable, protocol-agnostic, and portable |

---

## Engineering Insights, Gotchas & Forward-Looking Notes

### The Streaming Output Gap — What Actually Happens

The doc says the 3-tier handler fallback covers streaming. It is worth being precise about **why** the race condition exists and **what the fallback actually observes**.

Spring AI's streaming path (e.g. `DeepSeekChatModel.internalStream`) is implemented as a reactive `Flux`. The observation lifecycle is:

```
Flux.create(...)
  .doOnNext(chunk → aggregate into MessageAggregator)
  .doFinally(_ → observation.stop())     ← fires on last item / error / cancel
```

`MessageAggregator.setResponse()` is called **inside** the subscriber's `onNext` chain **before** `doFinally`. So by the time `observation.stop()` fires:

1. `setResponse()` has already been called on the context ✓
2. `SensitiveDataMaskingFilter.map()` runs — `ctx.getResponse()` **is not null** ✓

This means the streaming gap that existed in earlier Spring AI versions **may be fully closed** in the current version. The 3-tier handler is a safety net, not an active fix for a current failure mode. If you observe empty outputs in Langfuse for streaming calls, **verify `setResponse()` ordering** before blaming the handler — it may indicate a Spring AI version regression, not an application bug.

> **Diagnostic:** Enable `logging.level.io.micrometer.observation=TRACE` and look for `"langfuse.observation.output already set by filter"` vs `"Setting langfuse.observation.output from ChatModelObservationContext"` to determine which tier is firing.

---

### OTLP Payload Size & Diff-Length Risk

This application truncates diffs to 80,000 characters before sending them to agents. Those diffs flow through as prompt text into `langfuse.observation.input`. The full formatted prompt (system prompt + diff + prior messages) can approach **100 KB per span attribute**.

The OTel SDK's `BatchSpanProcessor` defaults:
- Max queue size: **2,048 spans**
- Max export batch size: **512 spans**
- Export interval: **5 seconds**
- Export timeout: **30 seconds**

A single review triggers ~5–6 spans (4 agents + synthesizer + root). At 100 KB per span, one review generates ~500 KB of OTLP payload — well within limits for development.

**Where this breaks at scale:**
- Langfuse's OTLP ingestion has a **4 MB hard limit per HTTP request**. A batch of 512 spans at 100 KB each = 51 MB — the exporter will silently drop oversized batches.
- The `BatchSpanProcessor` queue is in-memory. Under bursty load (many PRs in parallel), the queue fills and new spans are dropped with a `"DROPPED"` log at WARN level.

**Mitigation if this becomes a problem:**
```java
// Truncate langfuse.observation.input/output at the attribute level
// before it reaches the OTel bridge, e.g. in LangfuseObservationHandler:
private static final int MAX_ATTR_LENGTH = 32_768; // 32 KB

private String truncate(String value) {
    if (value == null || value.length() <= MAX_ATTR_LENGTH) return value;
    return value.substring(0, MAX_ATTR_LENGTH) + "\n...[TRUNCATED]";
}
```

No such truncation exists today — worth adding before production deployment.

---

### `InheritableThreadLocal` on Java 25 — Technical Debt

`ReviewContextHolder` uses `InheritableThreadLocal`, which works for the current `Executors.newVirtualThreadPerTaskExecutor()` pattern. However:

- **Java 21+ ScopedValue Migration path:** `ThreadLocal` (and by extension `InheritableThreadLocal`) is being phased out for virtual thread workloads. JEP 481 (`ScopedValue`) is standard/finalized in Java 25 and is the JDK's recommended replacement.
- **Inheritance is copy-on-create:** The value is copied when the child virtual thread is created. If the parent thread modifies the holder after thread creation (unlikely here, but possible if the design evolves), the child sees a stale value.
- **`clear()` is non-trivial in nested scenarios:** If a virtual thread creates further child threads (e.g. async tool calls spawning more Virtual Threads), those grandchildren inherit the value correctly, but `clear()` on the parent does not propagate — the grandchildren keep the stale reference until their own GC cycle.

**Recommended migration to `ScopedValue` (standardized in Java 25):**

```java
// Replace in ReviewContextHolder:
public static final ScopedValue<ReviewContext> CONTEXT = ScopedValue.newInstance();

// Usage in ReviewPipeline.execute():
ScopedValue.where(ReviewContextHolder.CONTEXT, new ReviewContext(userId, sessionId))
           .run(() -> {
               // all code here, including virtual thread spawning, sees the value
           });
// No explicit clear() needed — ScopedValue is automatically unbound at the end of run()
```

`ScopedValue` provides structured, bounded scope — the value is never visible outside the `run()` block, eliminating the leak risk entirely.

---

### OTLP Export Failure — Silent Data Loss

If Langfuse is unreachable, the `BatchSpanProcessor` logs errors and drops spans after the export timeout expires. **The application itself is unaffected** — observability failure does not propagate back to the request path.

What you will see in logs:
```
WARN  io.opentelemetry.sdk.trace.export.BatchSpanProcessor - Exporter failed, Spans that were previously queued are discarded.
```

What you will **not** see: any retry, any dead-letter queue, any alerting.

**Debugging OTLP connectivity:**
```bash
# Verify the endpoint is reachable and auth header is correct
curl -v -X POST http://localhost:3000/api/public/otel/v1/traces \
  -H "Authorization: Basic $(echo -n 'pk-lf-xxx:sk-lf-xxx' | base64)" \
  -H "x-langfuse-ingestion-version: 4" \
  -H "Content-Type: application/x-protobuf" \
  --data-binary @/dev/null
# Expect HTTP 200; 401 = wrong key; 404 = wrong URL path; connection refused = Langfuse down
```

**The `x-langfuse-ingestion-version: 4` header is mandatory for Langfuse 3.x.** Omitting it causes Langfuse to silently accept the request with HTTP 200 but discard the payload. This is the most common "traces not appearing" root cause.

---

### `sampling.probability: 1.0` — Production Consideration

Currently every span is sampled (`probability: 1.0`). For development this is correct. For production with high PR volume:

- Each PR review generates ~6 spans
- At 100 PRs/day = 600 spans/day — entirely manageable for ClickHouse
- At 10,000 PRs/day = 60,000 spans/day — still fine; ClickHouse is designed for this

Sampling reduction is unlikely to be needed unless the LLM call latency causes span attribute accumulation to outpace ClickHouse write throughput. Leave at `1.0` unless you observe ClickHouse CPU/memory pressure at scale.

---

### Attribute Write Ordering — Why Tier 1 Is the Common Path

The filter runs during `observation.stop()`. At that moment, for a **non-streaming** `ChatClient.call()`, the response is synchronously available. The filter writes `langfuse.observation.output` directly from the response object.

When the OTel bridge runs (after all handlers), it reads the final state of the context's `highCardinalityKeyValues` map and converts each entry to an OTel span attribute. The bridge does not differentiate between attributes written by the filter vs. the handler — it sees only the final map state. This means:

- If the filter writes `langfuse.observation.output` (Tier 1 path), the handler's Tier 1 check short-circuits and does nothing — **zero duplication risk**.
- If the handler writes it (Tier 2 or 3), it calls `removeHighCardinalityKeyValues(key)` before `addHighCardinalityKeyValue(key, value)`, ensuring no duplicate keys exist.

The `setHighCardinalityKeyValue` helper in both components enforces this:

```java
private void setHighCardinalityKeyValue(Observation.Context context, String key, String value) {
    if (value != null) {
        context.removeHighCardinalityKeyValues(key);   // idempotent remove
        context.addHighCardinalityKeyValue(KeyValue.of(key, value));
    }
}
```

This is the correct pattern when writing to Micrometer's observation context outside the initial observation construction — always remove before add.

---

### Differentiating Spring Boot Web/System Traces vs. Spring AI Traces

When running a standard Spring Boot web application, typical system spans (HTTP requests, database queries) are emitted alongside Spring AI spans. They can be distinguished and isolated using the following techniques:

1. **Naming Conventions**:
   - HTTP Server requests: `GET /api/review/{pr}` (from Spring Boot webmvc observations)
   - Spring AI LLM operations: `gen_ai.client.operation`
   - Spring AI Advisors: `spring.ai.advisor`
   - Spring AI Tool Calls: `gen_ai.tool_call`
2. **Filtering at the Micrometer Level**:
   Use `TracingObservationPredicate` to drop HTTP server and system spans before they reach the exporter queues:
   ```java
   public boolean test(String name, Observation.Context context) {
       // Filter out standard Boot web/database traces to avoid polluting the AI tracing backend
       return name != null && (name.startsWith("gen_ai") || name.startsWith("spring.ai") || name.equals("review-pipeline"));
   }
   ```
3. **Filtering in Analytics Backends**:
   - Spring AI spans contain domain-specific attributes such as `gen_ai.system` (`deepseek`), `gen_ai.usage.input_tokens`, and `langfuse.observation.input`.
   - Spring Boot system spans carry HTTP and process-related attributes (`http.request.method`, `url.path`, `db.system`).

   These distinct namespaces allow backends (like Langfuse or OpenTelemetry collectors) to cleanly route or query traces based on attribute existence.
