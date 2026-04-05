# Distributed Tracing with dd-trace-go v2

→ See `samber/cc-skills-golang@golang-context` skill for propagating context across service boundaries.

When using `dd-trace-go`, refer to the [official documentation](https://docs.datadoghq.com/tracing/trace_collection/automatic_instrumentation/dd_libraries/go/) for up-to-date API signatures.

## Why Tracing

When a request crosses multiple services, logs from each service are isolated. Tracing connects them: a single trace shows the full request path with timing for every operation. Datadog APM lets you answer "why was this request slow?" across any number of services.

## APM Setup

Call `utils.StartObservability()` at application startup. It reads configuration from environment variables — no code-level configuration needed:

| Env var | Purpose | Example |
| --- | --- | --- |
| `DD_SERVICE` | Service name shown in Datadog APM | `order-service` |
| `DD_ENV` | Deployment environment | `production`, `staging` |
| `DD_VERSION` | Service version for deployment tracking | `1.4.2` |
| `DD_AGENT_HOST` | Datadog Agent host | `localhost` (default) |
| `DD_TRACE_ENABLED` | Disable tracing without code changes | `false` |

```go
func main() {
    utils.StartObservability()
    defer utils.ShutdownObservability()
}
```

## Creating Spans

### Service / method spans

`utils.StartSpan(ctx)` creates a span named after the calling function automatically. Use it for all service methods, repository methods, and any meaningful operation:

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

func (r *OrderRepo) Insert(ctx context.Context, order Order) (*Order, error) {
    span := utils.StartSpan(ctx)
    defer span.Finish()

    _, err := r.db.ExecContext(ctx, "INSERT INTO orders ...", order.ID)
    if err != nil {
        utils.RecordSpanError(span, err)
        return nil, errors.Wrap(err, "exec insert")
    }
    return &order, nil
}
```

### Adding custom tags to spans

When you need to attach request-scoped metadata (user ID, order ID, tenant) to a span for filtering in Datadog:

```go
import "github.com/DataDog/dd-trace-go/v2/ddtrace/tracer"

func (s *OrderService) Create(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    span := utils.StartSpan(ctx)
    defer span.Finish()

    // Add searchable tags visible in Datadog APM
    span.SetTag("order.payment_method", req.PaymentMethod)
    span.SetTag("order.amount", req.Amount)
    span.SetTag("user.id", req.UserID)

    // ...
}
```

### Where to add spans

Spans MUST be created for:

- Every service method (business logic layer)
- Every repository/database method
- Every external API call
- Every message queue publish/consume operation
- Any operation that is slow or could fail

## HTTP Tracing

### Echo router

Use `utils.NewTracedEchoRouter` — it wraps Echo with `echotrace.Wrap`, which:
- Creates a span for every incoming request
- Registers routes in the Datadog API Catalog before traffic arrives
- Excludes OPTIONS, HEAD, and `/healthz` by default

```go
e := echo.New()
e = utils.NewTracedEchoRouter(e, utils.DefaultTracedEchoRouterConfig())

// Custom ignore rules:
e = utils.NewTracedEchoRouter(e, utils.TracedEchoRouterConfig{
    IgnoredMethods: utils.MethodSet(http.MethodOptions, http.MethodHead),
})
```

### Outbound HTTP client

Use `utils.NewTracedHTTPClient` — it wraps the client with `httptrace.WrapClient`:

```go
httpClient := utils.NewTracedHTTPClient(&http.Client{
    Timeout: 10 * time.Second,
})

// The client automatically creates child spans for every request,
// named by method + host (e.g. "GET api.stripe.com")
resp, err := httpClient.Get("https://api.stripe.com/v1/charges")
```

## Database Tracing

Use `utils.NewTracedDBConnection` — it registers the `pgx` driver with `sqltrace` and opens a traced connection. Every `QueryContext`, `ExecContext`, and `QueryRowContext` call automatically creates a child span:

```go
db, err := utils.NewTracedDBConnection(os.Getenv("DATABASE_URL"))
if err != nil {
    return errors.Wrap(err, "opening db")
}

// This query automatically appears as a child span in Datadog
rows, err := db.QueryContext(ctx, "SELECT * FROM orders WHERE user_id = $1", userID)
```

**Important:** Always use `*Context` variants (`QueryContext`, `ExecContext`, `QueryRowContext`). Non-context methods do not carry the span and produce gaps in traces.

## Redis / Valkey Tracing

Use `utils.NewTracedCacheConnection` — it wraps the Redis client with `redistrace.WrapClient`:

```go
// Without TLS
cache, err := utils.NewTracedCacheConnection(os.Getenv("REDIS_URL"), "")

// With custom CA cert (e.g. self-signed for Valkey)
cache, err := utils.NewTracedCacheConnection(os.Getenv("REDIS_URL"), "/etc/ssl/valkey-ca.pem")

// Redis commands appear as child spans automatically
val, err := cache.Get(ctx, "user:123").Result()
```

## AWS SDK Tracing

Use `utils.NewTracedAWS` — it appends `awstrace` middleware to the AWS config. All AWS API calls (S3, SQS, DynamoDB, etc.) appear as child spans:

```go
awsCfg, err := utils.NewTracedAWS(ctx)
if err != nil {
    return errors.Wrap(err, "loading aws config")
}

s3Client := s3.NewFromConfig(awsCfg)
// S3 calls automatically appear as child spans
```

## Distributed Tracing Across Async Boundaries

When traces cross an async boundary (message queue, scheduled job, webhook), the Go context chain is broken. Use `utils.MarshalSpan` to serialize the trace context into a string, carry it with the message, and `utils.ResumeSpanFromTrace` to reconnect on the other side:

```go
// Producer — serialize current span into the message
func (s *OrderService) PublishEvent(ctx context.Context, order Order) error {
    span := utils.StartSpan(ctx)
    defer span.Finish()

    event := OrderEvent{
        Order:   order,
        DDTrace: utils.MarshalSpan(ctx), // carries trace_id + span_id
    }
    return s.publisher.Publish(event)
}

// Consumer — resume trace from message
func (w *Worker) HandleEvent(event OrderEvent) error {
    span, ctx := utils.ResumeSpanFromTrace(context.Background(), event.DDTrace)
    defer span.Finish()

    // child spans will appear under the original producer trace in Datadog
    return w.fulfill(ctx, event.Order)
}
```

## Span Error Recording

`utils.RecordSpanError(span, err)` marks the span as errored in Datadog. Always call it before returning an error from a spanned function:

```go
span := utils.StartSpan(ctx)
defer span.Finish()

result, err := s.externalAPI.Call(ctx, req)
if err != nil {
    utils.RecordSpanError(span, err) // span shows as red in Datadog APM
    return nil, errors.Wrap(err, "external api call")
}
```

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Forgot `defer span.Finish()` | Span never sent to Datadog — always defer immediately after creating |
| Raw `sql.Open("pgx", dsn)` | No DB query tracing — use `utils.NewTracedDBConnection` |
| Raw `redis.NewClient(opts)` | No cache tracing — use `utils.NewTracedCacheConnection` |
| Raw `http.Client{}` | No outbound HTTP tracing — use `utils.NewTracedHTTPClient` |
| `db.Query(...)` instead of `db.QueryContext(ctx, ...)` | Context not propagated, span not linked |
| Context dropped mid-call-chain | Trace chain broken — propagate `ctx` through every function |
