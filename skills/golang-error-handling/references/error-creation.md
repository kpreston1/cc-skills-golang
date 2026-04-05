# Error Creation

## Errors as Values

Go treats errors as ordinary values implementing the `error` interface:

```go
type error interface {
    Error() string
}
```

This means errors are returned, not thrown. Every function that can fail returns an `error` as its last return value, and every caller must check it.

```go
// ✗ Bad — silently discarding errors
data, _ := os.ReadFile("config.yaml")

// ✗ Bad — only checking in some branches
result, err := doSomething()
fmt.Println(result) // using result without checking err

// ✓ Good — always check before using other return values
data, err := os.ReadFile("config.yaml")
if err != nil {
    return errors.Wrap(err, "reading config")
}
```

## Error String Conventions

Error strings MUST be lowercase, without trailing punctuation, and should not duplicate the context that wrapping will add.

```go
// ✗ Bad — capitalized, punctuation, redundant prefix
return errors.New("Failed to connect to database.")
return errors.Wrap(err, "UserService: failed to fetch user")

// ✓ Good — lowercase, no punctuation, concise
return errors.New("connection refused")
return errors.Wrap(err, "fetching user")
```

When errors are wrapped through multiple layers, each layer adds its own prefix. The result reads like a chain:

```
creating order: charging card: connecting to payment gateway: connection refused
```

## Creating Errors

### `errors.New` — static error messages

```go
import "github.com/pkg/errors"

var ErrNotFound = errors.New("not found")
var ErrUnauthorized = errors.New("unauthorized")
```

### `errors.Wrap` / `errors.Wrapf` — wrap with context and stack trace

Use `errors.Wrap` from `github.com/pkg/errors` to attach a message and capture a stack trace at the point of origin.

```go
import "github.com/pkg/errors"

// ✓ Good — wraps with context, captures stack trace
return errors.Wrap(err, "fetching user")

// ✓ Good — wraps with formatted context
return errors.Wrapf(err, "fetching user %s", userID)
```

### `errors.WithStack` — stack trace only, no new message

Use when you want to capture a stack trace without adding a new message layer:

```go
// ✓ Good — adds stack trace to a sentinel or stdlib error
return errors.WithStack(sql.ErrNoRows)
```

### `errors.WithMessage` — message only, no new stack trace

Use when adding context mid-chain where a stack trace was already captured:

```go
// ✓ Good — adds context without duplicating stack
return errors.WithMessage(err, "retrying after timeout")
```

### `errors.Cause` — unwrap to root cause

Use `errors.Cause` to get the original unwrapped error:

```go
if errors.Cause(err) == sql.ErrNoRows {
    return ErrNotFound
}
```

### Decision table: which error strategy to use

| Situation | Strategy | Example |
| --- | --- | --- |
| Caller needs to match a specific condition | Sentinel error (`errors.New` as package var) | `var ErrNotFound = errors.New("not found")` |
| Caller needs to extract structured data | Custom error type | `type ValidationError struct { Field, Msg string }` |
| Wrapping an error with context at a layer boundary | `errors.Wrap` / `errors.Wrapf` | `errors.Wrap(err, "fetching user")` |
| Adding stack trace to an error that has none | `errors.WithStack` | `errors.WithStack(sql.ErrNoRows)` |
| Error is purely informational, not matched on | `fmt.Errorf` or `errors.New` | `fmt.Errorf("connecting to %s: %w", addr, err)` |

## Low-Cardinality Error Messages

APM and log aggregation tools (Datadog, Loki, Sentry) group errors by message. When you interpolate variable data into error strings, every unique combination creates a separate group — dashboards become unusable and alerting breaks.

```go
// ✗ Bad — high cardinality: each file/line combo creates a unique error message
errors.Wrapf(err, "error in %s at line %d of the csv", csvPath, line)

// ✓ Good — static message, structured attributes at the log site
err := errors.Wrap(err, "csv parsing error")
// ... later, at the logging boundary:
slog.Error("csv parsing failed", "error", err, "csv_file_path", csvPath, "csv_file_line", line)
```

**Static wrapping prefixes are fine** — `errors.Wrap(err, "fetching user")` is low-cardinality because the prefix never changes. What to avoid is interpolating IDs, paths, counts, or other variable data into the message itself.

## Custom Error Types

Create custom error types when callers need to extract structured data from errors.

```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on %s: %s", e.Field, e.Message)
}

// Usage
func validateAge(age int) error {
    if age < 0 {
        return &ValidationError{Field: "age", Message: "must be non-negative"}
    }
    return nil
}
```

### Custom types that wrap other errors

Implement `Unwrap()` so `errors.Is` and `errors.As` can traverse the chain:

```go
type QueryError struct {
    Query string
    Err   error
}

func (e *QueryError) Error() string {
    return fmt.Sprintf("query %q: %v", e.Query, e.Err)
}

func (e *QueryError) Unwrap() error {
    return e.Err
}
```
