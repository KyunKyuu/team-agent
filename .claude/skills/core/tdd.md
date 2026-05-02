---
name: tdd
description: Universal TDD patterns — write failing test first, verify it fails, implement, verify it passes.
---

# TDD — Test-Driven Development

## The Loop (Non-Negotiable)

```
1. Write the failing test
2. Run it → MUST fail (if it passes, the test is wrong)
3. Write the minimal implementation to make it pass
4. Run it → MUST pass
5. Commit
```

Never write implementation before the test. Never skip step 2.

---

## Test Structure

```
test/
├── entity/       ← entity helpers, DTO mappers (white-box: package entity)
├── repository/   ← repository impl (black-box: package repository_test)
├── service/      ← service impl (black-box: package service_test)
├── handler/      ← HTTP and RPC handlers (black-box: package handler_test)
└── middleware/   ← middleware (black-box: package middleware_test)
```

Source files live in `internal/` — tests live in `test/`, not alongside source.

---

## White-box vs Black-box

| Layer | Package declaration | Why |
|-------|---------------------|-----|
| Entity helpers, DTO mappers | `package entity` (same as source) | Need access to unexported helpers |
| Repository, service, handler, middleware | `package {layer}_test` (external) | Test the public contract only |

---

## Test Setup Pattern

```go
// test/{layer}/{entity}_test.go
package {layer}_test

import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/suite"
)

type {Entity}Suite struct {
    suite.Suite
    // dependencies (use interfaces, not concrete)
}

func (s *{Entity}Suite) SetupTest() {
    // reset state before each test
}

func Test{Entity}Suite(t *testing.T) {
    suite.Run(t, new({Entity}Suite))
}
```

---

## Mocking Pattern

Use interfaces — inject mocks at test time.

```go
// mock the interface, not the concrete type
type mock{Entity}Repository struct {
    mock.Mock
}

func (m *mock{Entity}Repository) FindByID(ctx context.Context, id string, appOrigin string) (*entity.{Entity}, error) {
    args := m.Called(ctx, id, appOrigin)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*entity.{Entity}), args.Error(1)
}
```

---

## Test Naming

```
Test{What}_{Scenario}_{ExpectedResult}

Examples:
TestSendOTP_ValidRequest_ReturnsOTPCode
TestSendOTP_ExpiredOTP_ReturnsError
TestFindByID_NotFound_ReturnsHTTPError404
```

---

## What to Test Per Layer

**Entity:** helper methods, type mappings, default values
**DTO:** mapper functions (entity→response), validation rules
**Repository:** each query method, pagination, app_origin filter, error propagation
**Service:** business logic paths, error handling, cross-entity interactions
**Handler:** request binding, validation errors, service error → HTTP status mapping
**Middleware:** auth extraction, rejection on invalid token, passthrough on valid

---

## Run Commands

```bash
# Run specific test
go test ./test/{layer}/... -run Test{Name} -v

# Run all tests
go test ./test/... -v

# Run with coverage
go test ./test/... -cover -coverprofile=coverage.out
go tool cover -html=coverage.out
```
