# Coding Guide — Permanent Principles

Principles established through real code review and production experience. These apply to all projects, not just Meridian. Add new principles as they are discovered; cite the source (PR, review session, bug postmortem) so future readers understand the context.

---

## API Contracts & Return Types

### The return type is the contract

> **A function that returns `T` is promising "I always produce a valid T." When that's not true, the type must say so.**

The most common API contract violation in Rust (and any language) is a function that claims to always return a value but sometimes returns a degraded, empty, or meaningless one. The caller has no way to tell the difference.

**The rule:** If a function can fail or produce no meaningful result, its return type must encode that possibility:

| Situation | Return type |
|-----------|-------------|
| Always succeeds | `T` |
| May produce no result | `Option<T>` |
| May fail with an error the caller should handle | `Result<T, E>` |
| May fail, caller only cares if it worked | `Result<(), E>` |

**Never use a sentinel value to signal failure** — `""`, `0`, `"{}"`, `Vec::new()` returned from a function that "failed" silently. The caller cannot distinguish "empty but correct" from "empty because broken."

```rust
// Wrong — returns T but lies about it:
pub fn to_db_row(&self) -> (&'static str, String) {
    serde_json::to_string(self)
        .unwrap_or_else(|_| "{}".to_string())  // "{}" is a lie
    // caller has no idea serialisation failed
}

// Correct — return type matches the contract:
pub fn to_db_row(&self) -> Option<(&'static str, String)> {
    match serde_json::to_string(self) {
        Ok(payload) => Some((self.event_type(), payload)),
        Err(e) => {
            tracing::warn!(error = %e, "serialisation failed — event dropped");
            None
        }
    }
}
```

**Ask:** "Is a degraded result better than no result?" If writing a garbage value to a DB, emitting corrupt data, or returning an empty response is worse than skipping the operation entirely — the return type should be `Option<T>`. Let the caller decide whether to skip, retry, or alert.

*Source: `/reviewer --ride-along` Session 03, analytics.rs `to_db_row` fix*

---

## Observability

### Silent fallbacks are invisible failures

> **Every `unwrap_or_else`, `unwrap_or`, `unwrap_or_default`, and `ok()` that discards an error MUST emit a log before returning the fallback.**

A fallback that fires without logging is indistinguishable from correct operation. You will not know it happened until you're debugging a production incident by which time the evidence is gone.

```rust
// Wrong — silent, invisible degradation:
some_operation().unwrap_or_else(|_| default_value)

// Correct — visible degradation, operator can detect and alert:
some_operation().unwrap_or_else(|e| {
    tracing::warn!(error = %e, context = "relevant context", "operation failed — using default");
    default_value
})
```

*Source: `/reviewer --ride-along` Session 03, analytics.rs*

### Use the right log level

| Level | Macro | When |
|-------|-------|------|
| ERROR | `tracing::error!()` | Unexpected failure requiring investigation — broken invariant, data loss risk |
| WARN  | `tracing::warn!()`  | Unexpected but recoverable — event dropped, fallback triggered, degraded mode |
| INFO  | `tracing::info!()`  | Normal significant events — server started, migration ran, scheduler tick |
| DEBUG | `tracing::debug!()` | Useful during development, too noisy for production |
| TRACE | `tracing::trace!()` | Extremely fine-grained — loop iterations, per-byte ops |

**Choosing between WARN and ERROR:** If a on-call engineer receiving an alert at 2am would consider it urgent and actionable, use `error!`. If it's worth knowing about but doesn't require immediate action, use `warn!`.

### Log fields are typed, not interpolated

```rust
// Wrong — unstructured, unsearchable, unparseable:
tracing::error!("fetch failed for url: {url} with status {status}");

// Correct — structured, queryable, filterable:
tracing::error!(url = %url, status = status.as_u16(), "fetch failed");
```

`%value` uses the `Display` trait. `?value` uses the `Debug` trait. Both produce key=value pairs that log aggregation systems (Loki, Datadog, CloudWatch) can index and query. A format string produces an opaque string blob.

---

## Error Handling

### Library crates vs application code

| Context | Use | Why |
|---------|-----|-----|
| Library crate | `thiserror` | Callers need to `match` on error variants for control flow |
| Application / binary | `anyhow` | Errors propagate to a handler; context via `.context()` is enough |
| Never | `anyhow` in a library public API | Removes caller's ability to match on error type |

### Log errors once, at the handling site

Every time the same error is logged as it propagates with `?`, you get duplicate log entries. Log at the place where you handle it (catch it, convert it to a response, or surface it to the user). Everywhere else, just propagate with `?`.

```rust
// Wrong — logged twice for one error:
fn fetch() -> Result<(), FetchError> {
    let body = http_get(url).map_err(|e| {
        tracing::error!("http error: {e}");  // logged here
        e
    })?;
    Ok(())
}

fn run() {
    if let Err(e) = fetch() {
        tracing::error!("fetch failed: {e}");  // and here
    }
}

// Correct — propagate cleanly, log once at the handling site:
fn fetch() -> Result<(), FetchError> {
    let body = http_get(url)?;  // propagates, no log
    Ok(())
}

fn run() {
    if let Err(e) = fetch() {
        tracing::error!(error = %e, "feed fetch failed");  // one log, at the right level
    }
}
```

---

## Rust Safety

### Never use `unwrap()` or `expect()` in library crates

`unwrap()` panics on `None`/`Err`. Panics in library code are unrecoverable from the caller's perspective. Use `?` to propagate errors, or restructure so the function's types prove it cannot fail.

`expect("message")` is acceptable only when the message explains the *invariant* that guarantees it cannot fail — not just what you hoped would be true:

```rust
// Acceptable — invariant is stated:
let pool = open_pool(url).await.expect("pool open called once at startup; URL pre-validated");

// Not acceptable — just wishful thinking:
let value = map.get("key").expect("key should be there");
```

### Integer arithmetic on untrusted input

In debug mode, overflow panics. In release mode, it wraps silently (two's complement). Code that seems correct in tests can silently produce wrong values in production.

```rust
// Wrong — silent wrap in release:
let total = a + b;

// Correct for any input you don't fully control:
let total = a.checked_add(b).ok_or(MyError::Overflow)?;
// or, where saturation is the right semantic:
let total = a.saturating_add(b);
```

Never use `as` for narrowing casts — it silently truncates. Use `TryFrom`/`TryInto`:

```rust
let n: i64 = 1_000_000_000_000;
let m = n as i32;  // silently truncates — wrong
let m = i32::try_from(n)?;  // returns Err on overflow — correct
```

---

## Privacy

### Privacy-as-types

The most robust privacy enforcement is structural — make it *impossible* to emit PII by encoding the constraint in the type system.

```rust
// Wrong — raw string can be anything, including a domain name:
struct ArticleEvent { feed_info: String }

// Correct — the type guarantees it's a hash, not a domain:
struct ArticleEvent { feed_domain_hash: DomainHash }

struct DomainHash(String);  // only constructable via hash_id()
```

When PII leakage is a compile error rather than a runtime bug, you don't need code review to catch it — the compiler does.

### `#[derive(Debug)]` on sensitive types leaks to logs

Any struct with `#[derive(Debug)]` that contains passwords, API keys, tokens, or PII will print those values in log output whenever the struct is debug-formatted. Use `secrecy::Secret<T>` for sensitive fields — its `Debug` impl always prints `[REDACTED]`.

```rust
// Wrong — API key in every debug log:
#[derive(Debug)]
struct ProviderConfig { api_key: String }

// Correct:
use secrecy::Secret;
#[derive(Debug)]
struct ProviderConfig { api_key: Secret<String> }
```

---

## Async

### No blocking calls inside `async fn`

Any call that takes more than ~100µs without an `.await` blocks the entire tokio runtime thread, preventing other tasks from progressing.

| Blocking operation | Correct async equivalent |
|-------------------|--------------------------|
| `std::thread::sleep(d)` | `tokio::time::sleep(d).await` |
| `std::fs::read_to_string(path)` | `tokio::fs::read_to_string(path).await` |
| CPU-intensive work (ML inference, compression, hashing) | `tokio::task::spawn_blocking(|| ...).await` |

### spawn_blocking vs rayon

- `spawn_blocking`: for blocking I/O that must happen on a thread (legacy sync libraries, blocking DB calls)
- `rayon`: for CPU-bound parallel computation (data processing, batch operations)

`spawn_blocking` uses a pool capped at ~500 threads — wrong for CPU-bound work where you want threads = CPU cores.

---

*This guide is maintained in `saldistefano/claude-skills/references/coding-guide.md`.*
*Add new principles with: source context, the wrong pattern, the correct pattern, and the rule stated plainly.*
