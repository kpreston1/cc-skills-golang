---
name: golang-observability
description: "Golang everyday observability — the always-on signals in production using Datadog. Covers structured logging with slog + Datadog bridge, distributed tracing with dd-trace-go v2, continuous profiling with Datadog profiler, and APM setup. Apply when instrumenting Go services with Datadog, setting up dd-trace-go, adding spans, correlating logs with traces, wiring traced HTTP/DB/Redis/AWS clients, or propagating traces across service boundaries."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.2.1"
  openclaw:
    emoji: "📡"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent WebFetch WebSearch
---

**Persona:** You are a Go observability engineer. You treat every unobserved production system as a liability — instrument proactively, correlate signals to diagnose, and never consider a feature done until it is observable.

**Modes:**

- **Coding / instrumentation** (default): Add observability to new or existing code — add spans, wire traced clients, set up structured logging. Follow the sequential instrumentation guide.
- **Review mode** — reviewing a PR's instrumentation changes. Check that new code creates spans, uses traced constructors, and logs with context variants. Sequential.
- **Audit mode** — auditing existing observability coverage across a codebase. Launch up to 3 parallel sub-agents — one for spans/tracing, one for logging, one for traced client wiring.

> **Community default.** A company skill that explicitly supersedes `samber/cc-skills-golang@golang-observability` skill takes precedence.

# Go Observability with Datadog

This project uses **Datadog APM** via `dd-trace-go/v2` for distributed tracing, structured logging via `log/slog` with the Datadog slog bridge, and Datadog Continuous Profiling for always-on profiling. All observability utilities live in the shared `utils` package.

Install:

```bash
go get github.com/DataDog/dd-trace-go/v2
```

## Best Practices Summary

1. **Always use `utils.StartObservability()`** at application startup — never call `tracer.Start()` directly
2. **Always call `utils.ShutdownObservability()`** on graceful shutdown — it flushes buffered spans before exit
3. **Use `utils.StartSpan(ctx)`** for service/method spans — it auto-names the span from the caller function name
4. **Always `defer span.Finish()`** immediately after creating a span — leaked spans never reach Datadog
5. **Use `utils.RecordSpanError(span, err)`** to mark spans as failed — do not swallow errors silently
6. **Use `utils.NewTracedSlogHandler()`** for the slog handler — it injects `dd.trace_id` and `dd.span_id` into every log line automatically
7. **Use `slog.*Context(ctx, ...)`** variants, not bare `slog.*()` — the context carries the active span for trace correlation
8. **Use traced constructors** (`NewTracedHTTPClient`, `NewTracedDBConnection`, `NewTracedCacheConnection`, `NewTracedAWS`) — never use unwrapped clients, they produce gaps in traces
9. **Propagate context everywhere** — pass `ctx` through every function call; dropping context breaks the trace chain
10. **Use `utils.MarshalSpan` / `utils.ResumeSpanFromTrace`** for distributed tracing across async boundaries (message queues, async jobs)
11. **A feature is not done until it has spans** — every service method, DB query, and external API call must be traceable

## Cross-References

See `samber/cc-skills-golang@golang-error-handling` skill for the single error handling rule. See `samber/cc-skills-golang@golang-troubleshooting` skill for using Datadog APM to diagnose production issues. See `samber/cc-skills-golang@golang-context` skill for context propagation across service boundaries.

## APM Lifecycle

```go
func main() {
    utils.StartObservability()
    defer utils.ShutdownObservability()

    // ... application setup ...
}
```

`ShutdownObservability` is a no-op when `DD_TRACE_ENABLED=false`, making it safe to call unconditionally.

**Environment detection:** Use `utils.IsDevEnv(cfg.Environment)` (free function in the shared `utils` package) to branch on environment — do not add `IsDevEnv()` / `IsProdEnv()` methods to config structs. The free function is the established Eden-wide pattern and keeps config structs as plain data holders.

## Creating Spans

Use `utils.StartSpan(ctx)` for all service and method spans. It automatically names the span after the calling function:

```go
func (s *OrderService) Create(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    span := utils.StartSpan(ctx)
    defer span.Finish()

    order, err := s.repo.Insert(ctx, req.ToOrder())
    if err != nil {
        utils.RecordSpanError(span, err)
        return nil, errors.Wrap(err, "inserting order")
    }

    return order, nil
}
```

For outbound HTTP client spans, use `utils.StartReqClientSpan(ctx)`:

```go
func (c *Client) FetchUser(ctx context.Context, id string) (*User, error) {
    span := utils.StartReqClientSpan(ctx)
    defer span.Finish()

    // ... make HTTP request ...
}
```

For detailed span API, custom tags, and distributed tracing patterns see **[Tracing](./references/tracing.md)**.

## Structured Logging with Datadog Correlation

Use `utils.NewTracedSlogHandler` to create the slog handler. It produces JSON logs and automatically injects Datadog trace and span IDs so Datadog can correlate logs with traces:

```go
logger := slog.New(utils.NewTracedSlogHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))
slog.SetDefault(logger)
```

Always use context variants so trace IDs are injected:

```go
// ✗ Bad — no trace correlation
slog.Error("query failed", "error", err)

// ✓ Good — dd.trace_id and dd.span_id injected automatically
slog.ErrorContext(ctx, "query failed", "error", err)
```

For log levels, structured fields, and common mistakes see **[Logging](./references/logging.md)**.

## Traced Client Constructors

Never use unwrapped clients — they produce silent gaps in your Datadog traces. Always use the `utils` constructors:

| Client | Constructor |
| --- | --- |
| Echo HTTP router | `utils.NewTracedEchoRouter(e, cfg)` |
| Outbound HTTP client | `utils.NewTracedHTTPClient(c)` |
| PostgreSQL (`pgx`) | `utils.NewTracedDBConnection(dsn)` |
| Redis / Valkey | `utils.NewTracedCacheConnection(url, caCertPath)` |
| AWS SDK v2 | `utils.NewTracedAWS(ctx, optFns...)` |

```go
// Echo router
e := echo.New()
e = utils.NewTracedEchoRouter(e, utils.DefaultTracedEchoRouterConfig())

// HTTP client
httpClient := utils.NewTracedHTTPClient(&http.Client{Timeout: 10 * time.Second})

// DB
db, err := utils.NewTracedDBConnection(os.Getenv("DATABASE_URL"))

// Redis
cache, err := utils.NewTracedCacheConnection(os.Getenv("REDIS_URL"), "")

// AWS
awsCfg, err := utils.NewTracedAWS(ctx)
```

## Distributed Tracing Across Async Boundaries

When passing work across a message queue, async job, or any boundary that breaks the Go context chain, serialize the current span before enqueuing and resume it on the other side:

```go
// Producer — serialize the current span into a string
func (s *OrderService) Enqueue(ctx context.Context, order Order) error {
    ddTrace := utils.MarshalSpan(ctx)

    msg := Message{
        Payload: order,
        DDTrace: ddTrace, // carry the trace context with the message
    }
    return s.queue.Publish(msg)
}

// Consumer — resume the trace from the serialized string
func (w *Worker) Process(msg Message) error {
    span, ctx := utils.ResumeSpanFromTrace(context.Background(), msg.DDTrace)
    defer span.Finish()

    // all spans created from ctx are children of the original trace
    return w.orderService.Fulfill(ctx, msg.Payload)
}
```

## Definition of Done for Observability

A feature is not production-ready until it is observable. Before marking a feature done, verify:

- [ ] **Spans created** — every service method, DB query, and external API call creates a span with `utils.StartSpan(ctx)`
- [ ] **Errors recorded** — all error paths call `utils.RecordSpanError(span, err)`
- [ ] **Traced clients used** — no raw `sql.Open`, `redis.NewClient`, `http.Client`, or `aws.Config` without wrapping
- [ ] **Logging uses context** — all `slog.*` calls use the `*Context(ctx, ...)` variant
- [ ] **No PII in logs or span tags** — never log emails, passwords, SSNs, or tokens
- [ ] **Errors either logged or returned, never both** — see `samber/cc-skills-golang@golang-error-handling`

## Common Mistakes

```go
// ✗ Bad — span never finishes, never reaches Datadog
span := utils.StartSpan(ctx)
// ... forgot defer span.Finish()

// ✓ Good
span := utils.StartSpan(ctx)
defer span.Finish()
```

```go
// ✗ Bad — raw DB connection, no tracing
db, err := sql.Open("pgx", dsn)

// ✓ Good — traced connection
db, err := utils.NewTracedDBConnection(dsn)
```

```go
// ✗ Bad — log AND return (error gets logged twice up the chain)
if err != nil {
    slog.ErrorContext(ctx, "query failed", "error", err)
    return errors.Wrap(err, "query")
}

// ✓ Good — return with context, log once at the top level
if err != nil {
    return errors.Wrap(err, "querying users")
}
```

```go
// ✗ Bad — context dropped, trace chain broken
go func() {
    result, err := s.db.QueryContext(context.Background(), "SELECT ...")
}()

// ✓ Good — propagate ctx
go func() {
    result, err := s.db.QueryContext(ctx, "SELECT ...")
}()
```
