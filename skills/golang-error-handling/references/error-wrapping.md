# Error Wrapping and Inspection

## Error Wrapping with `github.com/pkg/errors`

`github.com/pkg/errors` provides `Wrap`, `Wrapf`, `WithStack`, and `WithMessage` to attach context and stack traces. Errors SHOULD be wrapped at each layer to build a readable chain.

```go
import "github.com/pkg/errors"

// ✓ Good — wraps with context and captures a stack trace
func (s *UserService) GetUser(id string) (*User, error) {
    user, err := s.repo.FindByID(id)
    if err != nil {
        return nil, errors.Wrapf(err, "getting user %s", id)
    }
    return user, nil
}
```

### Controlling exposure at system boundaries

Use `errors.Wrap` within your module to preserve the error chain. At public API / system boundaries, use `fmt.Errorf` with `%v` (not `%w`) to prevent callers from depending on internal error types.

```go
// Internal layer — wrap to preserve chain and capture stack
func (r *repo) fetch(id string) error {
    return errors.Wrap(err, "querying database")
}

// Public API boundary — break chain to hide internals
func (s *PublicService) GetItem(id string) error {
    err := s.repo.fetch(id)
    if err != nil {
        return fmt.Errorf("item unavailable: %v", err) // %v — callers cannot unwrap
    }
    return nil
}
```

### `errors.Cause` — get the root error

```go
// Unwrap to the original error
root := errors.Cause(err)
if root == sql.ErrNoRows {
    return ErrNotFound
}
```

## Inspecting Errors: `errors.Is` and `errors.As`

### `errors.Is` — match against a sentinel value

```go
// ✗ Bad — direct comparison breaks on wrapped errors
if err == sql.ErrNoRows {

// ✓ Good — traverses the entire error chain
if errors.Is(err, sql.ErrNoRows) {
    return nil, ErrNotFound
}
```

### `errors.As` — extract a typed error from the chain

```go
// ✗ Bad — type assertion breaks on wrapped errors
if ve, ok := err.(*ValidationError); ok {

// ✓ Good — traverses the entire error chain
var ve *ValidationError
if errors.As(err, &ve) {
    log.Printf("validation failed on field %s: %s", ve.Field, ve.Msg)
}
```

## Combining Errors with `errors.Join`

`errors.Join` (Go 1.20+) combines multiple independent errors into one. The combined error works with `errors.Is` and `errors.As` — each inner error is inspectable.

### Use case: validating multiple fields

```go
func validateUser(u User) error {
    var errs []error

    if u.Name == "" {
        errs = append(errs, errors.New("name is required"))
    }
    if u.Email == "" {
        errs = append(errs, errors.New("email is required"))
    }

    return errors.Join(errs...) // returns nil if errs is empty
}
```

### Use case: parallel operations with independent failures

```go
func closeAll(closers ...io.Closer) error {
    var errs []error
    for _, c := range closers {
        if err := c.Close(); err != nil {
            errs = append(errs, err)
        }
    }
    return errors.Join(errs...)
}
```

### `errors.Is` works through joined errors

```go
err := errors.Join(ErrNotFound, ErrUnauthorized)

errors.Is(err, ErrNotFound)    // true
errors.Is(err, ErrUnauthorized) // true
```
