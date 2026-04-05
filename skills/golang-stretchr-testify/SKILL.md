---
name: golang-stretchr-testify
description: "Comprehensive guide to stretchr/testify for Golang testing. Covers assert, require, mock, and suite packages in depth. Use whenever writing tests with testify, creating mocks, setting up test suites, or choosing between assert and require. Essential for testify assertions, mock expectations, argument matchers, call verification, suite lifecycle, and advanced patterns like Eventually, JSONEq, and custom matchers. Trigger on any Go test file importing testify."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.2"
  openclaw:
    emoji: "✅"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
        - gotests
    install:
      - kind: go
        package: github.com/cweill/gotests/...@latest
        bins: [gotests]
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent WebFetch mcp__context7__resolve-library-id mcp__context7__query-docs Bash(gotests:*)
---

**Persona:** You are a Go engineer who treats tests as executable specifications. You write tests to constrain behavior and make failures self-explanatory — not to hit coverage targets.

**Modes:**

- **Write mode** — adding new tests or mocks to a codebase.
- **Review mode** — auditing existing test code for testify misuse.

# stretchr/testify

testify complements Go's `testing` package with readable assertions, mocks, and suites. It does not replace `testing` — always use `*testing.T` as the entry point.

This skill is not exhaustive. Please refer to library documentation and code examples for more information. Context7 can help as a discoverability platform.

## require everywhere

Always use `s.Require()` for all assertions in suite tests — this fails fast on any unexpected state and prevents misleading failures cascading through the rest of the test.

**Rule**: use `s.Require()` for all assertions — setup, error checks, and verifications alike.

## Core Assertions

Always call assertions via `s.Require()` in suite tests:

```go
// Equality
s.Require().Equal(expected, actual)
s.Require().NotEqual(unexpected, actual)
s.Require().EqualValues(expected, actual)        // converts to common type first
s.Require().EqualExportedValues(expected, actual)

// Nil / Bool / Emptiness
s.Require().Nil(obj)               s.Require().NotNil(obj)
s.Require().True(cond)             s.Require().False(cond)
s.Require().Empty(collection)      s.Require().NotEmpty(collection)
s.Require().Len(collection, n)

// Contains (strings, slices, map keys)
s.Require().Contains("hello world", "world")
s.Require().Contains([]int{1, 2, 3}, 2)
s.Require().Contains(map[string]int{"a": 1}, "a")

// Comparison
s.Require().Greater(actual, threshold)     s.Require().Less(actual, ceiling)
s.Require().Positive(val)                  s.Require().Negative(val)
s.Require().Zero(val)

// Errors
s.Require().Error(err)                     s.Require().NoError(err)
s.Require().ErrorIs(err, ErrNotFound)      // walks error chain
s.Require().ErrorAs(err, &target)
s.Require().ErrorContains(err, "not found")

// Type
s.Require().IsType(&User{}, obj)
s.Require().Implements((*IReader)(nil), obj)
```

**Argument order**: always `(expected, actual)` — swapping produces confusing diff output.

## Advanced Assertions

```go
s.Require().ElementsMatch([]string{"b", "a", "c"}, result)             // unordered comparison
s.Require().InDelta(3.14, computedPi, 0.01)                            // float tolerance
s.Require().JSONEq(`{"name":"alice"}`, `{"name": "alice"}`)             // ignores whitespace/key order
s.Require().WithinDuration(expected, actual, 5*time.Second)
s.Require().Regexp(`^user-[a-f0-9]+$`, userID)

// Async polling
s.Require().Eventually(func() bool {
    status, _ := client.GetJobStatus(jobID)
    return status == "completed"
}, 5*time.Second, 100*time.Millisecond)

// Async polling with rich assertions
s.Require().EventuallyWithT(func(c *assert.CollectT) {
    resp, err := client.GetOrder(orderID)
    assert.NoError(c, err)
    assert.Equal(c, "shipped", resp.Status)
}, 10*time.Second, 500*time.Millisecond)
```

## testify/mock

Mock interfaces to isolate the unit under test. Embed `mock.Mock`, implement methods with `m.Called()`, always verify with `AssertExpectations(t)`.

Key matchers: `mock.Anything`, `mock.AnythingOfType("T")`, `mock.MatchedBy(func)`. Call modifiers: `.Once()`, `.Times(n)`, `.Maybe()`, `.Run(func)`.

For defining mocks, argument matchers, call modifiers, return sequences, and verification, see [Mock reference](./references/mock.md).

## testify/suite

Suites group related tests with shared setup/teardown.

### Lifecycle

```
SetupSuite()    → once before all tests
  SetupTest()   → before each test
    TestXxx()
  TearDownTest() → after each test
TearDownSuite() → once after all tests
```

### Example

```go
type TokenServiceSuite struct {
    suite.Suite
    store   *MockTokenStore
    service *TokenService
}

func (s *TokenServiceSuite) SetupTest() {
    s.store = new(MockTokenStore)
    s.service = NewTokenService(s.store)
}

func (s *TokenServiceSuite) TestGenerate_ReturnsValidToken() {
    s.store.On("Save", mock.Anything, mock.Anything).Return(nil)
    token, err := s.service.Generate("user-42")
    s.Require().NoError(err)
    s.Require().NotEmpty(token)
    s.store.AssertExpectations(s.T())
}

// Required launcher
func TestTokenServiceSuite(t *testing.T) {
    suite.Run(t, new(TokenServiceSuite))
}
```

Always use `s.Require()` for all assertions — never use bare `s.Equal()` or `s.NoError()` as they continue on failure and can produce misleading results.

## Common Mistakes

- **Forgetting `AssertExpectations(t)`** — mock expectations silently pass without verification
- **`s.Require().Equal(ErrNotFound, err)`** — fails on wrapped errors. Use `s.Require().ErrorIs` to walk the chain
- **Swapped argument order** — testify assumes `(expected, actual)`. Swapping produces backwards diffs
- **Using bare `s.Equal()` / `s.NoError()`** — these behave like `assert` and continue on failure, masking downstream panics. Always use `s.Require()`
- **Missing `suite.Run()`** — without the launcher function, zero tests execute silently
- **Comparing pointers** — `s.Require().Equal(ptr1, ptr2)` compares addresses. Dereference or use `EqualExportedValues`

## Linters

Use `testifylint` to catch wrong argument order, assert/require misuse, and more. See `samber/cc-skills-golang@golang-linter` skill.

## Cross-References

- → See `samber/cc-skills-golang@golang-testing` skill for general test patterns, table-driven tests, and CI
- → See `samber/cc-skills-golang@golang-linter` skill for testifylint configuration
