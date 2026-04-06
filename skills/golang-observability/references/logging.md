# Structured Logging with `slog`

→ See `samber/cc-skills-golang@golang-error-handling` skill for the single handling rule.

## Why Structured Logging

Structured logs emit key-value pairs instead of freeform strings. Datadog Log Management can index, filter, and aggregate structured fields — something impossible with `log.Printf` output.

```go
// ✗ Bad — freeform string, impossible to filter by user_id
log.Printf("ERROR: failed to create user %s: %v", userID, err)

// ✓ Good — structured key-value pairs, machine-parseable
slog.Error("user creation failed",
    "user_id", userID,
    "error", err,
)
// JSON output: {"time":"2025-01-15T10:30:00Z","level":"ERROR","msg":"user creation failed","user_id":"u-123","error":"connection refused","dd.trace_id":"...","dd.span_id":"..."}
```

## Handler Setup

Use `NewTracedSlogHandler` from the shared `utils` package. It bridges `slog` to Datadog and automatically injects `dd.trace_id` and `dd.span_id` into every log record, correlating logs with APM traces.

```go
import "your-org/utils"

// Production — JSON to stdout with Datadog trace correlation
logger := slog.New(utils.NewTracedSlogHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))
slog.SetDefault(logger)

// Development — human-readable text (plain slog.TextHandler is fine without correlation)
logger := slog.New(slog.NewTextHandler(os.Stderr, &slog.HandlerOptions{
    Level: slog.LevelDebug,
}))
slog.SetDefault(logger)
```

`NewTracedSlogHandler` wraps the standard JSON handler with the `slogtrace` bridge from `gopkg.in/DataDog/dd-trace-go.v2`. When a span is active on the context, the bridge extracts `dd.trace_id`, `dd.span_id`, and `dd.service` and appends them to the log record. This links log lines to traces in the Datadog UI automatically.

## Log Levels

```go
slog.Debug("cache lookup", "key", cacheKey, "hit", false)
slog.Info("order created", "order_id", orderID, "total", amount)
slog.Warn("rate limit approaching", "current_usage", 0.92, "limit", 1000)
slog.Error("payment failed", "order_id", orderID, "error", err)
```

**Rule of thumb**: if you're unsure between Warn and Error, ask "did the operation succeed?" If yes (even with degradation), use Warn. If no, use Error.

## Cost of Logging

Logging is not free. Each log line costs CPU (serialization), I/O (disk/network), and money (Datadog log ingestion and indexing). The cost scales with volume, controlled directly by log level.

- **Debug level in production** can generate millions of log lines per minute in a busy service, inflating Datadog costs by 10-100x
- **Info level** is the typical production default — it provides enough visibility without excessive volume
- Debug level MUST be disabled in production — use `slog.LevelInfo` in production and `slog.LevelDebug` only in development or when actively debugging a specific issue

## Logging with Context

MUST use the `*Context` variants. When `NewTracedSlogHandler` is configured, trace ID and span ID are automatically injected into every log record that passes through a context with an active span.

```go
// ✗ Bad — no trace correlation, log line cannot be linked to a Datadog trace
slog.Error("query failed", "error", err)

// ✓ Good — dd.trace_id/dd.span_id attached automatically
slog.ErrorContext(ctx, "query failed", "error", err)
```

## Adding Request-Scoped Attributes

Use `slog.With()` to create a child logger that includes attributes on every line. Middleware can inject request-scoped fields so all downstream logs carry the same context.

```go
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        logger := slog.With(
            "request_id", r.Header.Get("X-Request-ID"),
            "method", r.Method,
            "path", r.URL.Path,
        )
        ctx := WithLogger(r.Context(), logger)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

## Common Logging Mistakes

```go
// ✗ Bad — errors MUST be either logged OR returned, NEVER both (single handling rule violation)
if err != nil {
    slog.Error("query failed", "error", err)
    return fmt.Errorf("query: %w", err) // error gets logged twice up the chain
}

// ✓ Good — return with context, log at the top level
if err != nil {
    return fmt.Errorf("querying users: %w", err)
}

// ✗ Bad — NEVER log PII (emails, SSNs, passwords, tokens)
slog.Info("user logged in", "email", user.Email, "ssn", user.SSN)

// ✓ Good — log identifiers, not sensitive data
slog.Info("user logged in", "user_id", user.ID)
```
