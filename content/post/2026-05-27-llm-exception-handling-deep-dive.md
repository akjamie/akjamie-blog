---
layout: post
title: "Exception Handling for LLM Calls in Production Agentic Systems"
date: "2026-05-27"
author: "Jamie Zhang"
tags: ["Hermes", "Exception Handling", "LLM APIs", "Resilience", "Agentic AI"]
categories: ["AI"]
description: "A deep dive into agent/auxiliary_client.py from Hermes Agent. Written for engineers building production agentic applications who need to go beyond 'catch Exception and retry.'"
image: "/img/ai-01.jpg"
keywords: ["exception handling", "billing error", "rate limit error", "connection timeout", "provider fallback", "unsupported parameter"]
---

# Exception Handling for LLM Calls in Production Agentic Systems

> A deep dive into `agent/auxiliary_client.py` from Hermes Agent.
> Written for engineers building production agentic applications who need
> to go beyond "catch Exception and retry."


---

## Why LLM Exception Handling Is Different

Most backend services talk to one upstream. When it fails, you retry or return an error.

LLM-backed agents are different in three ways that make naive exception handling dangerous:

1. **The call is expensive.** A failed LLM call that retried five times against a dead endpoint burned five API round-trips, five timeouts, and potentially five billing events — before the user saw anything.

2. **The failure taxonomy is wide and overlapping.** HTTP 429 can mean "slow down for 30 seconds" or "your account is out of money." HTTP 402 can mean "billing exhaustion" or "transient quota reset." HTTP 400 can mean "bad request" or "context too long" or "this model doesn't support `temperature`." Treating them all as `Exception` and retrying is wrong for most of them.

3. **The agent has side tasks running in parallel.** Context compression, session search, title generation, and vision analysis all call LLMs. If any of them fail and block, the main agent loop stalls. If they fail silently, the agent loses state. The exception handling must be both resilient and transparent.

Hermes solves this with a layered exception handling architecture in `call_llm()` and `async_call_llm()`. This blog dissects every layer.

---

## The Architecture: Seven Layers of Recovery

```
call_llm(task, messages, ...)
    │
    ├─ 1. Provider resolution (fail fast if no credentials)
    │
    ├─ 2. First attempt: client.chat.completions.create(**kwargs)
    │       │
    │       ├─ 3. Temperature rejection → strip & retry once
    │       │
    │       ├─ 4. max_tokens rejection → strip & retry once
    │       │
    │       ├─ 5. Auth failure (401) → refresh credentials → retry same provider
    │       │       ├─ Nous: mint fresh agent key via Portal API
    │       │       └─ Others: rotate credential pool entry
    │       │
    │       ├─ 6. Pool recovery (auth/payment/rate-limit) → rotate pool → retry same provider
    │       │
    │       └─ 7. Cross-provider fallback (payment/connection/rate-limit on auto)
    │               → _try_payment_fallback() → next available provider
    │
    └─ raise (if all layers exhausted)
```

Each layer handles a specific failure class. The order matters — a wrong ordering causes either silent data loss or unnecessary fallbacks.

---

## Layer 1: Fail Fast on Missing Credentials

```python
client, final_model = _get_cached_client(resolved_provider, ...)
if client is None:
    _explicit = (resolved_provider or "").strip().lower()
    if _explicit and _explicit not in {"auto", "openrouter", "custom"}:
        raise RuntimeError(
            f"Provider '{_explicit}' is set in config.yaml but no API key "
            f"was found. Set the {_explicit.upper()}_API_KEY environment "
            f"variable, or switch to a different provider with `hermes model`."
        )
    # For auto/custom: try the full auto chain
    client, final_model = _get_cached_client("auto", main_runtime=main_runtime)
if client is None:
    raise RuntimeError(
        f"No LLM provider configured for task={task}. Run: hermes setup")
```

**The insight:** Distinguish between "user explicitly configured a provider that has no credentials" (hard fail with a clear message) and "auto-detection found nothing" (try the full chain before giving up). Conflating these produces confusing errors — "OpenRouter not configured" when the user set `provider: deepseek` and forgot to set `DEEPSEEK_API_KEY`.

**The pattern:** Fail fast with actionable messages. The error message names the exact env var to set and the exact command to run. This is not a nice-to-have — it is the difference between a 30-second fix and a 30-minute debugging session.

---

## Layer 2: The Error Classifiers

Before any retry logic, Hermes classifies every exception into one of four categories. These classifiers are the foundation of the entire recovery system.

```python
def _is_payment_error(exc: Exception) -> bool:
    status = getattr(exc, "status_code", None)
    if status == 402:
        return True
    err_lower = str(exc).lower()
    if status in {402, 429, None}:
        if any(kw in err_lower for kw in (
            "credits", "insufficient funds", "can only afford",
            "billing", "payment required"
        )):
            return True
    return False

def _is_rate_limit_error(exc: Exception) -> bool:
    if type(exc).__name__ == "RateLimitError":
        return True   # OpenAI SDK sometimes omits .status_code
    if status == 429:
        if any(kw in err_lower for kw in (
            "rate limit", "too many requests", "try again", "retry after"
        )):
            return True
        # Generic 429 without billing keywords = rate limit
        if not any(kw in err_lower for kw in (
            "credits", "insufficient funds", "billing", "payment required"
        )):
            return True
    return False

def _is_connection_error(exc: Exception) -> bool:
    from openai import APIConnectionError, APITimeoutError
    if isinstance(exc, (APIConnectionError, APITimeoutError)):
        return True
    err_type = type(exc).__name__
    if any(kw in err_type for kw in ("Connection", "Timeout", "DNS", "SSL")):
        return True
    err_lower = str(exc).lower()
    if any(kw in err_lower for kw in (
        "connection refused", "timed out", "connection reset",
        "incomplete chunked read", "peer closed connection",
        "response ended prematurely", "unexpected eof",
    )):
        return True
    return False

def _is_auth_error(exc: Exception) -> bool:
    status = getattr(exc, "status_code", None)
    if status == 401:
        return True
    err_lower = str(exc).lower()
    return "error code: 401" in err_lower or "authenticationerror" in type(exc).__name__.lower()
```

**The critical insight: HTTP 429 is ambiguous.** It can mean rate limiting (transient, retry in 30s) or billing exhaustion (permanent until you top up). Hermes disambiguates by checking the error message body. If it contains billing keywords (`credits`, `insufficient funds`, `can only afford`), it's a payment error. If it contains rate-limit keywords (`rate limit`, `try again`, `retry after`), it's a rate limit. A generic 429 with no billing keywords defaults to rate limit.

Getting this wrong is expensive: treating a billing exhaustion as a rate limit causes the agent to retry against a dead provider for minutes. Treating a rate limit as billing causes the agent to fall back to a different (potentially more expensive) provider unnecessarily.

**The `RateLimitError` class name check** is a real-world fix. The OpenAI SDK's `RateLimitError` sometimes omits `.status_code` on the exception object. Without the class name check, these errors fall through all classifiers and get re-raised as unhandled exceptions.

---

## Layer 3: Temperature Rejection — Strip and Retry

```python
try:
    return _validate_llm_response(
        client.chat.completions.create(**kwargs), task)
except Exception as first_err:
    if "temperature" in kwargs and _is_unsupported_temperature_error(first_err):
        retry_kwargs = dict(kwargs)
        retry_kwargs.pop("temperature", None)
        logger.info("Auxiliary %s: provider rejected temperature; retrying once without it", task)
        try:
            return _validate_llm_response(
                client.chat.completions.create(**retry_kwargs), task)
        except Exception as retry_err:
            # If retry still fails with payment/connection/auth, fall through
            if not (_is_payment_error(retry_err) or _is_connection_error(retry_err)
                    or _is_auth_error(retry_err) or "max_tokens" in str(retry_err)):
                raise
            first_err = retry_err
            kwargs = retry_kwargs  # carry stripped kwargs forward
```

**Why this exists:** Different providers have different opinions about `temperature`. Some models (Kimi/Moonshot) manage temperature server-side and reject any client-provided value. Some reasoning models (o1, DeepSeek-R1) reject temperature entirely. Some providers accept it but return a 400 with "unsupported parameter: temperature."

**The pattern:** Detect parameter rejection errors, strip the offending parameter, retry once. If the retry also fails with a different error class (payment, connection, auth), carry the stripped kwargs forward into the next recovery layer rather than re-raising. This prevents a double failure: "temperature rejected, then payment error" should fall through to the payment fallback, not surface as a temperature error.

**The `_is_unsupported_temperature_error` function** checks for provider-specific 400 patterns: `"unsupported parameter"`, `"temperature is not supported"`, `"unknown parameter: temperature"`, and the ZAI-specific error code `1210`. Each of these is a real provider quirk discovered in production.

---

## Layer 4: `max_tokens` Rejection — Strip and Retry

```python
_is_zai_param_error = (
    "1210" in err_str
    and "bigmodel" in str(getattr(client, "base_url", ""))
)
if max_tokens is not None and (
    "max_tokens" in err_str
    or "unsupported_parameter" in err_str
    or _is_unsupported_parameter_error(first_err, "max_tokens")
    or _is_zai_param_error
):
    kwargs.pop("max_tokens", None)
    kwargs.pop("max_completion_tokens", None)
    try:
        return _validate_llm_response(
            client.chat.completions.create(**kwargs), task)
    except Exception as retry_err:
        if not (_is_payment_error(retry_err) or _is_connection_error(retry_err)
                or _is_rate_limit_error(retry_err)):
            raise
        first_err = retry_err
```

**The ZAI-specific case** is instructive. ZAI's vision models (glm-4v-flash) return error code `1210` ("API 调用参数有误") when `max_tokens` is passed on multimodal calls. The error message does NOT contain "max_tokens" — it's a Chinese-language error about invalid parameters. Without the explicit `"1210" in err_str and "bigmodel" in base_url` check, this error would fall through all classifiers and surface as an unhandled exception.

**The pattern:** Provider-specific error codes require provider-specific detection. A generic "bad request" classifier is not enough. When you add a new provider, audit its error codes and add them to the appropriate classifier.

**Both `max_tokens` and `max_completion_tokens` are stripped** because different providers use different parameter names for the same concept. OpenAI uses `max_completion_tokens` for newer models; most others use `max_tokens`. Stripping both ensures the retry doesn't fail on the wrong parameter name.

---

## Layer 5: Auth Failure — Refresh and Retry Same Provider

```python
# Nous-specific: mint a fresh agent key via the Portal API
client_is_nous = (
    resolved_provider == "nous"
    or base_url_host_matches(_base_info, "inference-api.nousresearch.com")
)
if _is_auth_error(first_err) and client_is_nous:
    refreshed_client, refreshed_model = _refresh_nous_auxiliary_client(...)
    if refreshed_client is not None:
        logger.info("Auxiliary %s: refreshed Nous runtime credentials after 401, retrying", task)
        return _validate_llm_response(
            refreshed_client.chat.completions.create(**kwargs), task)

# Generic: refresh credentials from auth store / pool
if (_is_auth_error(first_err)
        and resolved_provider not in {"auto", "", None}
        and not client_is_nous):
    if _refresh_provider_credentials(resolved_provider):
        logger.info("Auxiliary %s: refreshed %s credentials after auth error, retrying",
                    task, resolved_provider)
        return _retry_same_provider_sync(...)
```

**The insight:** A 401 does not always mean "wrong credentials." It often means "credentials expired." OAuth tokens have TTLs. Agent keys have TTLs. The correct response to a 401 is to refresh the credentials and retry — not to fall back to a different provider.

**Nous gets special treatment** because its auth model is more complex: it uses short-lived agent keys minted from a longer-lived OAuth token. When the agent key expires, `_refresh_nous_auxiliary_client` calls the Portal API to mint a fresh one. This is the same path the main agent loop uses for 401 recovery, ensuring auxiliary tasks and the main loop stay in sync.

**The `resolved_provider not in {"auto", "", None}` guard** prevents auth refresh from running when the provider was auto-detected. If auto-detection picked OpenRouter and OpenRouter returns 401, refreshing OpenRouter credentials is the right move. But if auto-detection picked a provider the user didn't explicitly configure, refreshing its credentials is wrong — the user may not even know that provider was selected.

---

## Layer 6: Credential Pool Recovery — Rotate and Retry

```python
pool_provider = _recoverable_pool_provider(resolved_provider, client)
if pool_provider and (_is_auth_error(first_err) or _is_payment_error(first_err)
                      or _is_rate_limit_error(first_err)):
    recovery_err = first_err
    if _is_rate_limit_error(first_err):
        # Rate limits are often transient — try once more before rotating
        try:
            return _validate_llm_response(
                client.chat.completions.create(**kwargs), task)
        except Exception as retry_err:
            if not (_is_auth_error(retry_err) or _is_payment_error(retry_err)
                    or _is_rate_limit_error(retry_err)):
                raise
            recovery_err = retry_err
    if _recover_provider_pool(pool_provider, recovery_err):
        logger.info("Auxiliary %s: recovered %s via credential-pool rotation after %s",
                    task, pool_provider, type(recovery_err).__name__)
        return _retry_same_provider_sync(...)
```

**The credential pool** is a multi-key store for the same provider. When one key is exhausted (rate-limited, billing depleted, or auth-expired), the pool rotates to the next available key. This is the right recovery for multi-key setups before falling back to a different provider entirely.

**Rate limits get one extra retry** before pool rotation. This is because rate limits are often transient — a 429 that fires at the exact moment a rate window resets may succeed on the very next attempt. Rotating the pool on every 429 would exhaust pool entries unnecessarily.

**The `_recoverable_pool_provider` function** checks whether the current client is backed by a pool that has more than one entry. A single-key pool cannot recover by rotation — it would just retry the same exhausted key. This check prevents the pool recovery path from running when it has nowhere to rotate to.

---

## Layer 7: Cross-Provider Fallback — The Last Resort

```python
should_fallback = (
    _is_payment_error(first_err)
    or _is_connection_error(first_err)
    or _is_rate_limit_error(first_err)
)
is_auto = resolved_provider in {"auto", "", None}
if should_fallback and is_auto:
    if _is_payment_error(first_err):
        reason = "payment error"
        _mark_provider_unhealthy(
            _recoverable_pool_provider(resolved_provider, client) or resolved_provider
        )
    elif _is_rate_limit_error(first_err):
        reason = "rate limit"
    else:
        reason = "connection error"
    logger.info("Auxiliary %s: %s on %s (%s), trying fallback",
                task, reason, resolved_provider, first_err)
    fb_client, fb_model, fb_label = _try_payment_fallback(
        resolved_provider, task, reason=reason)
    if fb_client is not None:
        fb_kwargs = _build_call_kwargs(fb_label, fb_model, messages, ...)
        return _validate_llm_response(
            fb_client.chat.completions.create(**fb_kwargs), task)

# Connection errors poison the cached client — evict it
if _is_connection_error(first_err):
    try:
        _evict_cached_client_instance(client)
    except Exception:
        logger.debug("Auxiliary: cache eviction after connection error failed", exc_info=True)
raise
```

**The `is_auto` guard is critical.** Cross-provider fallback only runs when the provider was auto-detected. If the user explicitly configured `provider: openrouter`, falling back to Nous Portal when OpenRouter is rate-limited would be surprising and potentially expensive. Explicit provider = hard constraint. Auto = best-effort.

**Payment errors mark the provider unhealthy** for 10 minutes. This prevents the next auxiliary call (which may fire 5 seconds later for a different task) from hitting the same depleted provider again. Without this cache, a long session with frequent auxiliary calls would burn dozens of doomed 402 round-trips against an empty OpenRouter balance.

**Connection errors evict the cached client.** A connection error leaves the cached httpx transport in a closed or half-read state. Reusing it on the next call would fail immediately with a "connection error" that looks like a new failure but is actually the same dead transport. Evicting the cache forces the next call to build a fresh client.

---

## The Unhealthy Provider Cache

```python
_AUX_UNHEALTHY_TTL_SECONDS = 600  # 10 minutes
_aux_unhealthy_until: Dict[str, float] = {}

def _mark_provider_unhealthy(provider: str, ttl: Optional[float] = None) -> None:
    label = _normalize_chain_label(provider)
    expires_at = time.time() + (ttl if ttl is not None else _AUX_UNHEALTHY_TTL_SECONDS)
    _aux_unhealthy_until[label] = expires_at
    logger.warning(
        "Auxiliary: marking %s unhealthy for %ds (payment / credit error). "
        "Subsequent auxiliary calls will skip it until %s.",
        label, int(ttl or _AUX_UNHEALTHY_TTL_SECONDS),
        time.strftime("%H:%M:%S", time.localtime(expires_at)),
    )

def _is_provider_unhealthy(label: str) -> bool:
    expires_at = _aux_unhealthy_until.get(label)
    if expires_at is None:
        return False
    if time.time() >= expires_at:
        _aux_unhealthy_until.pop(label, None)  # lazy eviction
        return False
    return True
```

**The problem this solves:** In a long-running gateway session, auxiliary tasks fire frequently — every compression, every title generation, every session search. If OpenRouter is depleted, each of these tasks would hit OpenRouter first (it's first in the auto-detection chain), get a 402, then fall back. That's one wasted RTT per auxiliary call, potentially dozens per hour.

**The solution:** When any caller observes a payment error, mark that provider unhealthy for 10 minutes. All subsequent auxiliary calls skip it entirely. The TTL auto-expires so a topped-up account recovers without manual intervention.

**The log message includes the exact expiry time** (`time.strftime`). This is a small but important detail: when debugging "why is my agent using Nous instead of OpenRouter?", seeing "OpenRouter marked unhealthy until 14:32:15" is immediately actionable. "OpenRouter marked unhealthy for 600 seconds" requires mental arithmetic.

---

## The `_is_unsupported_parameter_error` Pattern

```python
def _is_unsupported_parameter_error(exc: Exception, param: str) -> bool:
    """Detect provider 400s for an unsupported request parameter."""
    status = getattr(exc, "status_code", None)
    if status != 400:
        return False
    err_lower = str(exc).lower()
    param_lower = param.lower()
    # Match: "unsupported parameter: temperature", "temperature is not supported",
    # "unknown parameter: temperature", "unrecognized request argument: temperature"
    if param_lower in err_lower:
        if any(kw in err_lower for kw in (
            "unsupported", "not supported", "unknown parameter",
            "unrecognized", "invalid parameter",
        )):
            return True
    # OpenAI SDK structured error: {"error": {"param": "temperature", "code": "unsupported_parameter"}}
    body = getattr(exc, "body", None) or {}
    if isinstance(body, dict):
        error_obj = body.get("error", {})
        if isinstance(error_obj, dict):
            if error_obj.get("param") == param or error_obj.get("code") == "unsupported_parameter":
                return True
    return False
```

**The insight:** Provider 400 errors for unsupported parameters come in many forms. Some providers use structured error bodies (`{"error": {"param": "temperature", "code": "unsupported_parameter"}}`). Others use free-text messages. The function checks both, and it's parameterized so the same logic covers `temperature`, `max_tokens`, `seed`, `top_p`, and any future parameter.

**This is a generalization of a specific production bug.** The original code had a temperature-specific detector. When `max_tokens` started causing the same class of errors on different providers, the pattern was generalized rather than duplicated. This is the right engineering instinct: when you write the same exception detection logic twice, extract it.

---

## The Timeout Architecture for Streaming Calls

```python
total_timeout = timeout if isinstance(timeout, (int, float)) and timeout > 0 else None
deadline = time.monotonic() + float(total_timeout) if total_timeout else None
timed_out = threading.Event()
timeout_timer: Optional[threading.Timer] = None

def _close_client_on_timeout() -> None:
    timed_out.set()
    close = getattr(self._client, "close", None)
    if callable(close):
        try:
            close()
        except Exception:
            pass
    try:
        _evict_cached_client_instance(self._client)
    except Exception:
        pass

try:
    if total_timeout:
        timeout_timer = threading.Timer(float(total_timeout), _close_client_on_timeout)
        timeout_timer.daemon = True
        timeout_timer.start()
    with self._client.responses.stream(**resp_kwargs) as stream:
        for _event in stream:
            _check_cancelled()
            ...
except Exception as exc:
    if timed_out.is_set():
        raise TimeoutError(_timeout_message()) from exc
    raise
finally:
    if timeout_timer is not None:
        timeout_timer.cancel()
```

**The problem:** Streaming LLM calls can stall indefinitely. The provider may accept the connection, start streaming, then stop mid-response without closing the connection. The httpx client's per-request timeout only covers the initial connection and the time between chunks — it does not enforce a total response time.

**The solution:** A `threading.Timer` fires after the total timeout, closes the httpx transport, and evicts the cached client. The `timed_out` event flag lets the exception handler distinguish "timed out" from "other error" and raise a clean `TimeoutError` instead of a confusing `ConnectionError`.

**The cache eviction on timeout** is critical. After the transport is closed, the cached client is poisoned — any subsequent call using it will fail immediately. Without eviction, the next auxiliary call reuses the dead client and fails with a connection error that looks like a new failure.

**The `_check_cancelled()` function** checks both the deadline and the agent's interrupt flag. This means a user pressing Ctrl+C during a streaming auxiliary call will cleanly interrupt it rather than waiting for the timeout to fire.

---

## Expert Insights

### 1. The Exception Handling IS the Product

For auxiliary LLM calls, the happy path is trivial: call the API, return the response. The exception handling is where the real engineering lives. A production agentic system will spend 5-10% of its time in exception recovery paths. Those paths determine whether the agent degrades gracefully or fails catastrophically.

### 2. Classify Before You Recover

Every recovery action in `call_llm` is gated on a classifier (`_is_payment_error`, `_is_rate_limit_error`, `_is_connection_error`, `_is_auth_error`). The classifiers are the most important code in the file. If a classifier is wrong, the recovery action is wrong. Invest in making classifiers precise.

The most common mistake is treating all 429s as rate limits. A 429 with "insufficient credits" in the body is a billing exhaustion — retrying it will fail immediately, and retrying it 10 times will burn 10 API round-trips. Check the body, not just the status code.

### 3. The `is_auto` Guard Protects User Intent

Cross-provider fallback only runs when `resolved_provider in {"auto", "", None}`. This is not a performance optimization — it is a correctness constraint. When a user explicitly configures `provider: openrouter`, they are making a deliberate choice. Silently routing to Nous Portal when OpenRouter is rate-limited violates that choice and can cause unexpected billing on a provider the user didn't intend to use.

Explicit provider = hard constraint. Auto = best-effort. Never conflate them.

### 4. Connection Errors Poison Cached Clients

When a connection error occurs, the cached httpx transport is in an undefined state. It may be closed, half-read, or have a dead async event loop. Reusing it will fail immediately. Always evict the cached client after a connection error, even if you found a fallback provider. The next call to the original provider should build a fresh client.

**"Why not just create a new client on every call and skip the cache entirely?"**

This is a reasonable question. Client construction itself is cheap — allocating a connection pool object, copying config, creating a few Python objects. No I/O, no DNS, no TCP. Typically ~0.1–0.5ms. The expensive part is the TCP+TLS handshake, which happens on the first request, not on client construction.

| Operation | Typical cost |
|:---|:---|
| Client construction | ~0.2ms (pure in-memory) |
| TCP handshake | ~1–5ms LAN, ~50–200ms WAN |
| TLS handshake | ~10–100ms |

If you create a new client on every call, httpx opens a fresh TCP+TLS connection every time. For LLM APIs over HTTPS, that's 100–300ms of overhead per call — on top of the actual model latency. For a 2-second LLM call, that's a 5–15% overhead tax on every single request.

What you're really caching is not the client object — it's the connection pool inside it. A cached client reuses established TCP connections and TLS sessions across calls. A new client pays the handshake cost every time.

**The right pattern: cache the client, evict on error.**

```python
_clients: dict[str, httpx.AsyncClient] = {}

def get_client(provider: str) -> httpx.AsyncClient:
    if provider not in _clients:
        _clients[provider] = httpx.AsyncClient(...)  # cheap: ~0.2ms
    return _clients[provider]  # returns pooled connections

def evict_client(provider: str):
    client = _clients.pop(provider, None)
    if client:
        asyncio.create_task(client.aclose())  # drain gracefully

async def call_llm(provider: str, payload):
    client = get_client(provider)
    try:
        return await client.post(...)
    except (httpx.ConnectError, httpx.RemoteProtocolError):
        evict_client(provider)   # poisoned — throw it away
        raise                    # next call rebuilds fresh
```

Happy path: reuses TCP/TLS connections, fast. Error path: evicts the poisoned client, next call pays the reconnect cost once and re-warms the pool. The reconnect penalty on error is acceptable because you already have a failure to handle — a clean retry with a fresh client is strictly better than retrying with a broken one.

### 5. The Unhealthy Provider Cache Is a Cost Control Mechanism

The 10-minute unhealthy cache is not primarily about performance — it is about waste elimination. A common misconception: "a 402 has no token cost, so why does it matter?"

A 402 has no token cost, but it has three real costs:

**1. Latency cost (the most visible)**

Without the unhealthy cache, every session pays the round-trip to a provider you already know is depleted:

```
Session A: OpenRouter → 402 (50–200ms) → fallback → response ✓
Session B: OpenRouter → 402 (50–200ms) → fallback → response ✓
Session C: OpenRouter → 402 (50–200ms) → fallback → response ✓
... × hundreds of concurrent sessions
```

With the unhealthy cache:

```
Session A: OpenRouter → 402 (50–200ms) → fallback → response ✓  [marks unhealthy]
Session B: skip OpenRouter → fallback → response ✓  (0ms wasted)
Session C: skip OpenRouter → fallback → response ✓  (0ms wasted)
```

Every user is paying 50–200ms of pure waste waiting for a response you already know will fail.

**2. Infrastructure cost**

Each HTTP round-trip consumes CPU, a socket, a connection from your pool, and thread/coroutine scheduling time. At hundreds of concurrent sessions, this is measurable load on your gateway process — for zero benefit.

**3. Fallback provider cost (the actual billing impact)**

You probably pay the fallback provider per call or per token. If OpenRouter is your cheap primary and Anthropic-direct is your fallback:

```
Without cache: 500 sessions × (1 wasted OpenRouter call + 1 Anthropic call)
With cache:    500 sessions × (1 Anthropic call) + 1 wasted OpenRouter call
```

The fallback call happens either way — but without the cache, you're also burning 500 unnecessary outbound HTTPS requests that show up in your infrastructure billing, rate limit counters, and connection pool exhaustion.

Some providers also count failed requests against your rate limit quota. A 402 from OpenRouter may still consume a request slot depending on their implementation.

The unhealthy cache converts "hundreds of wasted round-trips per hour" into "one 402, then skip for 10 minutes." The 10-minute TTL auto-expires so a topped-up account recovers without manual intervention.

### 6. Carry Stripped kwargs Forward

When a parameter-stripping retry (temperature, max_tokens) fails with a different error class, carry the stripped kwargs forward into the next recovery layer. This prevents double failures: "temperature rejected, then payment error" should fall through to the payment fallback using the temperature-stripped kwargs, not re-raise the temperature error.

**The concrete failure scenario without carrying kwargs forward:**

```python
original_kwargs = {"model": "gpt-4", "temperature": 2.5, "max_tokens": 999999}

# Layer 1: temperature=2.5 is invalid → strip it, retry
stripped_kwargs = {"model": "gpt-4", "max_tokens": 999999}
→ retry with stripped_kwargs
→ NOW hits 402 (balance depleted)

# Layer 2 catches the 402 → but retries with... original_kwargs  ← ❌
→ hits 422 again (temperature still invalid)
→ crashes with temperature error, not payment error
```

The user sees a parameter error even though the real problem is payment. The temperature fix was already solved — you threw it away.

**With carrying kwargs forward:**

```python
original_kwargs = {"model": "gpt-4", "temperature": 2.5, "max_tokens": 999999}

# Layer 1: strips temperature, retries
stripped_kwargs = {"model": "gpt-4", "max_tokens": 999999}
→ hits 402

# Layer 2 catches the 402 → uses stripped_kwargs as the base  ← ✓
→ switches provider, retries with {"model": "gpt-4", "max_tokens": 999999}
→ succeeds (or fails cleanly with a payment error, not a param error)
```

**The mental model:** think of `kwargs` as a document being progressively corrected. Each recovery layer should hand off its corrected version, not the original. If Layer 2 grabs the original instead of Layer 1's output, it silently undoes Layer 1's work.

```
original_kwargs
    → [Layer 1 fix] → stripped_kwargs   # temperature problem solved
        → [Layer 2 fix] → fallback_kwargs  # payment problem solved
            → [Layer 3 fix] → ...
```

The key is a single `current_kwargs` variable that accumulates all fixes and gets passed to every subsequent attempt, regardless of which layer is handling the error:

```python
async def call_with_recovery(provider, **kwargs):
    current_kwargs = kwargs  # ← this travels forward and mutates

    try:
        return await call(provider, **current_kwargs)

    except InvalidParameterError:
        current_kwargs = strip_invalid_params(current_kwargs)  # fix it
        try:
            return await call(provider, **current_kwargs)
        except PaymentError:
            # carry current_kwargs (already stripped) into next layer
            return await fallback_provider_call(**current_kwargs)  # ✓

    except PaymentError:
        return await fallback_provider_call(**current_kwargs)  # ✓
```

In Hermes, this is the `first_err = retry_err; kwargs = retry_kwargs` pattern at the end of each stripping block:

```python
except Exception as retry_err:
    if not (_is_payment_error(retry_err) or _is_connection_error(retry_err)):
        raise
    first_err = retry_err
    kwargs = retry_kwargs  # ← carry stripped kwargs forward
```

`first_err` carries the new error class into the next classifier check. `kwargs` carries the cleaned request into the next retry attempt. Both must be updated together.

### 7. Log at the Right Level

- `logger.debug`: Internal state, credential resolution, cache hits/misses. Verbose, for debugging.
- `logger.info`: Recovery actions taken (retried, rotated, fell back). Visible in normal operation.
- `logger.warning`: Degraded state (provider marked unhealthy, fallback activated, credentials missing). Requires attention.
- `logger.error`: Unrecoverable failures. Requires immediate action.

The unhealthy provider log uses `logger.warning` and includes the exact expiry time. This is the right level — it is not an error (the system recovered), but it is not normal operation either.

### 8. Test the Exception Paths, Not Just the Happy Path

The test suite for `call_llm` should cover:
- 402 on primary provider → fallback to secondary
- 429 (rate limit) on primary → fallback to secondary
- 429 (billing) on primary → fallback to secondary (different from rate limit)
- 401 on Nous → mint fresh agent key → retry
- 401 on other provider → rotate pool → retry
- Temperature rejection → strip → retry
- max_tokens rejection → strip → retry
- Connection error → evict cache → raise
- Timeout → evict cache → raise TimeoutError

Most of these are not tested in typical LLM application test suites. They are the failures that happen in production at 2am.

---

## Summary: The Exception Handling Taxonomy

| Error class | Detection | Recovery | Fallback? |
|:---|:---|:---|:---|
| No credentials | `client is None` | Fail fast with actionable message | No |
| Temperature rejected | `_is_unsupported_temperature_error` | Strip `temperature`, retry once | Only if retry also fails |
| `max_tokens` rejected | `"max_tokens" in err_str` or ZAI code 1210 | Strip `max_tokens`/`max_completion_tokens`, retry once | Only if retry also fails |
| Auth expired (Nous) | `_is_auth_error` + Nous base URL | Mint fresh agent key, retry same provider | No |
| Auth expired (other) | `_is_auth_error` + explicit provider | Refresh credential store, retry same provider | No |
| Pool exhausted | `_is_auth_error` / `_is_payment_error` / `_is_rate_limit_error` + pool exists | Rotate pool entry, retry same provider | No |
| Payment / billing | `_is_payment_error` + auto provider | Mark unhealthy 10min, try next provider | Yes (auto only) |
| Rate limit | `_is_rate_limit_error` + auto provider | One immediate retry, then try next provider | Yes (auto only) |
| Connection error | `_is_connection_error` + auto provider | Evict cached client, try next provider | Yes (auto only) |
| Connection error | `_is_connection_error` + explicit provider | Evict cached client | No |
| Unclassified | Everything else | Re-raise | No |
