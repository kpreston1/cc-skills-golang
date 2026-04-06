# Tests, Benchmarks, and Examples

## File Naming Conventions

Go uses suffix-based naming for test-related files:

| Suffix | Purpose | Build Tag |
| --- | --- | --- |
| `_test.go` | Tests | Not included in normal builds |
| `_mock.go` | Mock implementations | Excluded from production builds via `//go:build !mock` |
| `_bench_test.go` | Benchmarks | Not included in normal builds |
| `_example_test.go` | Examples that verify output | Not included in normal builds |
| No suffix | Regular code | Included in all builds |

## Mock Files

Mock files MUST be co-located with the interface they implement, using the `_mock.go` suffix. They MUST include the `//go:build !mock` build tag so they are excluded from production builds but available during tests.

```
internal/
└── adbroker/
    ├── client.go           # Production code + IClient interface
    └── client_mock.go      # MockClient — co-located, excluded from prod builds
```

**Mock file conventions:**

- Filename: `{file}_mock.go` (e.g., `client_mock.go` for `client.go`)
- Same package as the interface — mock lives in the same package, not a separate `mock/` directory
- Build tag `//go:build !mock` on the first line — excludes from production builds
- Mocks MUST use `github.com/stretchr/testify/mock` — embed `mock.Mock`, use `b.Called(...)` pattern
- Compile-time interface assertion: `var _ IClient = &MockClient{}` — catches drift immediately

**Example:**

```go
//go:build !mock

// Package adbroker provides test doubles for the adbroker client.
package adbroker

import (
    "context"

    "github.com/stretchr/testify/mock"
)

// MockClient is a testify mock for IClient.
type MockClient struct {
    mock.Mock
}

var _ IClient = &MockClient{} // assert MockClient adheres to IClient

func (m *MockClient) DoSomething(ctx context.Context, id string) (*Result, error) {
    args := m.Called(ctx, id)
    result, ok := args.Get(0).(*Result)
    if !ok {
        return nil, args.Error(1)
    }
    return result, args.Error(1)
}
```

## Where to Place Tests

**Co-locate tests with the code they test:**

```
internal/
├── handler/
│   ├── handler.go          # Production code
│   ├── handler_test.go     # Tests for handler
│   └── handler_bench_test.go  # Benchmarks (optional)
├── service/
│   ├── service.go
│   └── service_test.go
└── model/
    ├── user.go
    └── user_test.go

pkg/
└── logger/
    ├── logger.go
    └── logger_test.go
```

**Key principles:**

- Tests live in the **same package** as the code (e.g., `package handler`)
- Test files are in the **same directory** as the code they test
- Use `_test.go` suffix for all test files

## Test Package Options

When writing tests, you have two options for the package declaration:

**Option 1: Same package (white-box testing)**

```go
package handler  // Same package, can access unexported

import "testing"

func TestHandler(t *testing.T) {
    // Can access unexported functions and types
    internalFunction()
}
```

**Option 2: Package with `_test` suffix (black-box testing)**

```go
package handler_test  // Different package, only exported API

import "testing"

func TestHandler(t *testing.T) {
    // Can only access exported functions and types
    handler.PublicMethod()
}
```

**When to use each:**

- Use **same package** for unit tests that need to test internals
- Use **`_test` suffix** for integration/behavioral tests

## Benchmarks

Benchmarks use the `_bench_test.go` suffix and contain functions with the `Benchmark` prefix.

## Examples

Examples serve two purposes: documentation and verification.

**In libraries** - use `*_example_test.go` files:

```
pkg/
└── logger/
    ├── logger.go
    ├── logger_test.go
    └── logger_example_test.go     # Examples
```

**Example function format:**

```go
package logger

import "fmt"

func ExampleLogger_Info() {
    log := New()
    log.Info("processing started")
    log.Info("processing complete")
    // Output:
    // INFO: processing started
    // INFO: processing complete
}
```

**Key points:**

- Example functions must start with `Example`
- The `// Output:` comment verifies the output
- Examples are runnable tests: `go test` will fail if output doesn't match
- `godoc` displays examples as documentation
- File name format: `{package}_example_test.go` (e.g., `logger_example_test.go`)

**For executable examples** (standalone demo programs):

```
examples/
└── basic-usage/
    └── main.go                    # Executable example
```

## Test Utilities

When you have shared test helpers, use a dedicated package:

```
test/
└── testutils/
    ├── mock.go
    └── fixtures.go
```

Or use the `internal/testutil` pattern:

```
internal/
└── testutil/
    ├── mock.go
    └── fixtures.go
```

## Test Fixtures

Fixtures are test data files used across multiple tests. Use one of these patterns:

**Option 1: Local testdata directory** (package-specific fixtures)

```
internal/
└── handler/
    ├── handler.go
    ├── handler_test.go
    └── testdata/
        ├── users.json
        ├── request_valid.json
        └── request_invalid.json
```

**Option 2: Global test directory** (shared across packages)

```
test/
└── fixtures/
    ├── users.json
    ├── products.json
    └── responses/
        ├── success.json
        └── error.json
```

**Option 3: Embedded fixtures** (Go 1.16+, use `//go:embed`)

```
internal/
└── handler/
    ├── handler.go
    ├── handler_test.go
    └── testdata/
        └── users.json
```

**Important notes:**

- Go ignores the `testdata` directory when building regular packages
- Use `testdata/` for package-specific test data
- Use `test/fixtures/` for cross-package shared fixtures
- Don't put `.go` files in `testdata/` - they will be ignored

## Running Tests

```bash
go test ./...                    # Run all tests
go test ./internal/handler       # Test specific package
go test -v ./...                 # Verbose output
go test -race ./...              # Race detection
go test -cover ./...             # Coverage report
go test -short ./...             # Skip long-running tests
```

## Test File Summary

| File Type | Suffix | Package | Purpose |
| --- | --- | --- | --- |
| Test | `*_test.go` | `package X` or `package X_test` | Unit/integration tests |
| Mock | `*_mock.go` | Same as interface | Testify mocks, excluded from prod via `//go:build !mock` |
| Benchmark | `*_bench_test.go` | Same as code | Performance tests |
| Example (godoc) | `*_example_test.go` | Same as code | Documentation + verification |
| Executable example | No suffix | `package main` | Standalone demo programs |
| Test utilities | `*_test.go` | `package testutil` | Shared test helpers |
