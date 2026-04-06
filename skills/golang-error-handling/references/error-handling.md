# Error Handling Patterns and Logging

## The Single Handling Rule

An error MUST be handled exactly once: either log it or return it, never both. Doing both causes duplicate log entries and makes debugging harder.

```go
// ✗ Bad — logs AND returns (duplicate noise)
func processOrder(id string) error {
    err := chargeCard(id)
    if err != nil {
        log.Printf("failed to charge card: %v", err)
        return errors.Wrap(err, "charging card")
    }
    return nil
}

// ✓ Good — wrap with context and return, let the caller decide
func processOrder(id string) error {
    err := chargeCard(id)
    if err != nil {
        return errors.Wrapf(err, "charging card for order %s", id)
    }
    return nil
}

// ✓ Good — handle at the top level (HTTP handler, main, etc.)
func handleOrder(w http.ResponseWriter, r *http.Request) {
    err := processOrder(r.FormValue("id"))
    if err != nil {
        slog.Error("order failed", "error", fmt.Sprintf("%+v", err)) // %+v prints stack trace
        http.Error(w, "internal error", http.StatusInternalServerError)
        return
    }
    w.WriteHeader(http.StatusOK)
}
```

## Stack Traces with `github.com/pkg/errors`

Use `errors.Wrap` or `errors.WithStack` at the origin of an error to capture a stack trace. The stack is printed with `%+v`:

```go
import "github.com/pkg/errors"

func (r *UserRepository) FindByID(id string) (*User, error) {
    row := r.db.QueryRow("SELECT * FROM users WHERE id = $1", id)
    if err := row.Scan(&user); err != nil {
        return nil, errors.Wrap(err, "scanning user row") // stack captured here
    }
    return &user, nil
}

// At the logging boundary:
if err != nil {
    slog.Error("request failed", "error", fmt.Sprintf("%+v", err)) // prints full stack
}
```

Only wrap once per error origin — avoid calling `errors.WithStack` on an error already wrapped by `errors.Wrap`, as it duplicates the stack trace.

## Panic and Recover

### When to panic

Panic MUST only be used for truly unrecoverable states — programmer errors, impossible conditions, or corrupt invariants. NEVER use panic for expected failures like network timeouts or missing files.

```go
// ✓ Acceptable — programmer error in initialization
func MustCompileRegex(pattern string) *regexp.Regexp {
    re, err := regexp.Compile(pattern)
    if err != nil {
        panic(fmt.Sprintf("invalid regex %q: %v", pattern, err))
    }
    return re
}

// ✗ Bad — panic for a normal failure
func GetUser(id string) *User {
    user, err := db.Find(id)
    if err != nil {
        panic(err) // callers cannot recover gracefully
    }
    return user
}
```

### Recovering from panics

Use `recover` in deferred functions at goroutine boundaries (HTTP handlers, worker goroutines) to prevent one panic from crashing the entire process.

```go
func safeHandler(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if r := recover(); r != nil {
                slog.Error("panic recovered",
                    "panic", r,
                    "stack", string(debug.Stack()),
                )
                http.Error(w, "internal error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

## Logging Errors with `slog`

→ See `samber/cc-skills-golang@golang-observability` skill for comprehensive structured logging guidance, including `slog` setup, log levels, log handlers, HTTP middleware, and cost considerations.
