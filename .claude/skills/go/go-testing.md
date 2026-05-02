---
name: go-testing
description: |
  Write Go tests following TDD (Test-Driven Development) for microservices in the jr-external/jr-web-partner rebuild project.
  Use this skill when writing tests for any layer: entity helpers, repository (integration), service (mock-based), handler (httptest), middleware, and config.
  Covers test directory structure, testify setup, per-layer test patterns, TDD step template, and verification commands.
  Trigger when: writing tests, following TDD, verifying implementation, setting up test infrastructure, creating test helpers.
---

# Go Testing Skill

## Principles

- **TDD is mandatory**: Write failing test FIRST, verify it fails, then implement, then verify it passes.
- **Two test package types**: White-box (same package) for unit tests, Black-box (`_test` suffix) for integration/API tests.
- **Tests in separate directory from production code**: Tests live in their own directory, not alongside the source files.
- **Testing library**: `github.com/stretchr/testify/assert` for assertions, `github.com/stretchr/testify/mock` for mocks.
- **Max 300 lines per test file**: Split by entity or by operation group if needed.
- **Test helpers in `internal/testutil/`**: Shared fixtures, DB setup, mock factories.
- **No test skips**: Every test must run. Use `t.Skip()` only for known external dependencies that are unavailable in CI.

## White-Box vs Black-Box Testing

| Type | Package Declaration | Access | Use When |
|------|-------------------|--------|---------|
| **White-box** | `package user` (same as source) | Private + Public | Unit tests for internal logic, helper methods, unexported functions |
| **Black-box** | `package user_test` (separate) | Public only | Integration tests, API contract tests, testing as a consumer would |

**Rule of thumb:**
- Entity helper tests → White-box (need to test private `getLegacyType()` etc.)
- DTO mapper tests → White-box (need to test private `toResponse()` etc.)
- Repository tests → Black-box (test public interface only, real DB)
- Service tests → Black-box (test public interface, mock repos)
- Handler tests → Black-box (test public HTTP API, httptest)
- Middleware tests → Black-box (test public middleware chain)

## Test Directory Structure

Tests live in their own directory, separate from production code:

```
internal/
├── entity/
│   ├── notification.go
│   ├── notification_helpers.go
│   └── test/
│       └── notification_test.go          ← White-box: package entity
├── dto/
│   ├── notification.go
│   ├── mapper.go
│   └── test/
│       └── notification_test.go          ← White-box: package dto
├── domain/
│   └── repository/
│       └── notification_repository.go
├── repository/
│   ├── notification_repository.go
│   └── test/
│       └── notification_repository_test.go  ← Black-box: package repository_test
├── service/
│   ├── notification_service.go
│   └── test/
│       └── notification_service_test.go    ← Black-box: package service_test
├── handler/
│   ├── notification_handler.go
│   └── test/
│       └── notification_handler_test.go    ← Black-box: package handler_test
├── middleware/
│   ├── middleware.go
│   └── test/
│       └── middleware_test.go              ← Black-box: package middleware_test
└── testutil/
    ├── db.go                          ← Test DB setup/teardown
    ├── fixture.go                     ← Test data factories
    └── mock_repository.go             ← Generated or hand-written mocks
```

## Dependencies

Add to `go.mod`:

```
github.com/stretchr/testify v1.9.0
github.com/testcontainers/testcontainers-go  (optional, for integration tests)
```

## TDD Step Template

Every implementation task follows this template:

### Step 1: Write the failing test

**White-box example (entity helper — same package):**

```go
// internal/entity/test/notification_test.go
package entity

import (
    "testing"
    "github.com/stretchr/testify/assert"
)

func TestNotification_GetLegacyType(t *testing.T) {
    tests := []struct {
        name      string
        eventType string
        expected  string
    }{
        {"transaction_created maps to legacy", "transaction_created", "pembayaran_dibuat"},
        {"payment_received maps to legacy", "payment_received", "pembayaran_diterima"},
        {"unknown type returns as-is", "unknown_event", "unknown_event"},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            notif := Notification{EventType: tt.eventType}
            assert.Equal(t, tt.expected, notif.GetLegacyType())
        })
    }
}
```

**Black-box example (repository — separate package):**

```go
// internal/repository/test/notification_repository_test.go
package repository_test

import (
    "context"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"

    "myapp/internal/entity"
    "myapp/internal/repository"
    "myapp/internal/testutil"
)

func TestNotificationRepository_FindByID(t *testing.T) {
    db := testutil.SetupTestDB(t)
    defer testutil.TeardownTestDB(t, db)

    repo := repository.NewNotificationRepository(db, "web_partner")

    t.Run("should find notification by id", func(t *testing.T) {
        // Arrange
        notif := &entity.Notification{
            ID:         "notif-001",
            UserID:     "user-001",
            AppOrigin:  "web_partner",
            // ... required fields
        }
        require.NoError(t, repo.Create(context.Background(), notif))

        // Act
        result, err := repo.FindByID(context.Background(), "notif-001")

        // Assert
        assert.NoError(t, err)
        assert.Equal(t, "notif-001", result.ID)
        assert.Equal(t, "user-001", result.UserID)
    })

    t.Run("should return error when notification not found", func(t *testing.T) {
        // Act
        result, err := repo.FindByID(context.Background(), "nonexistent")

        // Assert
        assert.Error(t, err)
        assert.Nil(t, result)
    })
}
```

### Step 2: Run test to verify it fails

```bash
cd {BASE_PATH} && go test ./internal/repository/ -v -run TestNotificationRepository_FindByID
```

Expected: FAIL (method not yet implemented)

### Step 3: Write minimal implementation

(Implemented in the corresponding skill — go-repository, go-service, etc.)

### Step 4: Run test to verify it passes

```bash
cd {BASE_PATH} && go test ./internal/repository/ -v -run TestNotificationRepository_FindByID
```

Expected: PASS

### Step 5: Commit

```bash
cd {BASE_PATH} && git add internal/repository/notification_repository.go internal/repository/notification_repository_test.go && git commit -m "feat({SERVICE}): add notification repository FindByID"
```

## Per-Layer Test Patterns

### Entity Helper Tests (White-box — same package)

```go
// internal/entity/test/notification_test.go
package entity

import (
    "testing"
    "github.com/stretchr/testify/assert"
)

func TestNotification_GetLegacyType(t *testing.T) {
    tests := []struct {
        name      string
        eventType string
        expected  string
    }{
        {"transaction_created maps to legacy", "transaction_created", "pembayaran_dibuat"},
        {"payment_received maps to legacy", "payment_received", "pembayaran_diterima"},
        {"unknown type returns as-is", "unknown_event", "unknown_event"},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            notif := Notification{EventType: tt.eventType}
            assert.Equal(t, tt.expected, notif.GetLegacyType())
        })
    }
}

func TestNotification_IsRead(t *testing.T) {
    notif := Notification{IsRead: false}
    assert.False(t, notif.IsRead())

    notif.IsRead = true
    assert.True(t, notif.IsRead())
}
```

### Repository Integration Tests (Black-box — separate package)

```go
// internal/repository/test/notification_repository_test.go
package repository_test

import (
    "context"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"

    "myapp/internal/entity"
    "myapp/internal/repository"
    "myapp/internal/testutil"
)

func TestNotificationRepository_Create(t *testing.T) {
    db := testutil.SetupTestDB(t)
    defer testutil.TeardownTestDB(t, db)
    repo := repository.NewNotificationRepository(db, "web_partner")

    t.Run("should create notification successfully", func(t *testing.T) {
        notif := testutil.NewNotificationFixture(t)
        err := repo.Create(context.Background(), notif)
        assert.NoError(t, err)
    })

    t.Run("should filter by app_origin", func(t *testing.T) {
        notif := testutil.NewNotificationFixture(t)
        notif.AppOrigin = "external"

        // Wrong app_origin should not find it
        wpRepo := repository.NewNotificationRepository(db, "web_partner")
        result, err := wpRepo.FindByID(context.Background(), notif.ID)
        assert.Error(t, err)
        assert.Nil(t, result)
    })
}
```

### Service Unit Tests (Black-box — separate package, mock repos)

```go
// internal/service/test/notification_service_test.go
package service_test

import (
    "context"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"

    "myapp/internal/service"
    "myapp/internal/testutil"
)

func TestNotificationService_GetByID(t *testing.T) {
    mockRepo := new(testutil.MockNotificationRepository)
    svc := service.NewNotificationService(mockRepo)

    t.Run("should return notification when found", func(t *testing.T) {
        expected := testutil.NewNotificationFixture(t)
        mockRepo.On("FindByID", mock.Anything, "notif-001").Return(expected, nil)

        result, err := svc.GetByID(context.Background(), "notif-001")

        assert.NoError(t, err)
        assert.Equal(t, "notif-001", result.ID)
        mockRepo.AssertExpectations(t)
    })

    t.Run("should return error when not found", func(t *testing.T) {
        mockRepo.On("FindByID", mock.Anything, "nonexistent").Return(nil, appErrors.NewNotFoundError("notification not found"))

        result, err := svc.GetByID(context.Background(), "nonexistent")

        assert.Error(t, err)
        assert.Nil(t, result)
        mockRepo.AssertExpectations(t)
    })
}
```

### Handler HTTP Tests (Black-box — separate package, httptest)

```go
// internal/handler/test/notification_handler_test.go
package handler_test

import (
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/labstack/echo/v4"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"

    "myapp/internal/dto"
    "myapp/internal/handler"
    "myapp/pkg/response"
)

func TestNotificationHandler_GetNotifications(t *testing.T) {
    e := echo.New()
    mockSvc := new(testutil.MockNotificationService)
    handler := handler.NewNotificationHandler(mockSvc)

    t.Run("should return notifications list", func(t *testing.T) {
        expected := []dto.NotificationResponse{
            {ID: "notif-001", Title: "Test"},
        }
        mockSvc.On("GetByUserID", mock.Anything, "user-001", 1, 10).Return(expected, int64(1), nil)

        req := httptest.NewRequest(http.MethodGet, "/partner/notifications?page=1&limit=10", nil)
        req.Header.Set("accesstoken", "valid-token")
        rec := httptest.NewRecorder()

        ctx := e.NewContext(req, rec)
        ctx.SetPath("/partner/notifications")

        err := handler.GetNotifications(ctx)

        assert.NoError(t, err)
        assert.Equal(t, http.StatusOK, rec.Code)

        var resp response.BaseResponse
        json.Unmarshal(rec.Body.Bytes(), &resp)
        assert.Equal(t, 200, resp.Code)
    })
}
```

## Test Utilities (internal/testutil/)

### db.go — Test Database Setup

```go
package testutil

import (
    "testing"

    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
    "myapp/internal/entity"
)

func SetupTestDB(t *testing.T) *gorm.DB {
    t.Helper()
    db, err := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
    if err != nil {
        t.Fatalf("failed to connect test database: %v", err)
    }

    // Auto-migrate all entities for test
    err = db.AutoMigrate(
        &entity.Notification{},
        &entity.DeviceToken{},
        &entity.Preference{},
    )
    if err != nil {
        t.Fatalf("failed to migrate test database: %v", err)
    }

    return db
}

func TeardownTestDB(t *testing.T, db *gorm.DB) {
    t.Helper()
    sqlDB, err := db.DB()
    if err != nil {
        t.Fatalf("failed to get underlying sql.DB: %v", err)
    }
    sqlDB.Close()
}
```

### fixture.go — Test Data Factories

```go
package testutil

import (
    "testing"

    "myapp/internal/entity"
)

func NewNotificationFixture(t *testing.T) *entity.Notification {
    t.Helper()
    return &entity.Notification{
        ID:        "notif-001",
        UserID:    "user-001",
        Title:     "Test Notification",
        AppOrigin: "web_partner",
        IsRead:    false,
    }
}

func NewDeviceTokenFixture(t *testing.T) *entity.DeviceToken {
    t.Helper()
    return &entity.DeviceToken{
        ID:        "token-001",
        UserID:    "user-001",
        Token:     "fcm-test-token",
        Platform:  "android",
        AppOrigin: "web_partner",
    }
}
```

## Verification Commands

### Run all tests

```bash
cd {BASE_PATH} && go test ./... -v
```

### Run tests for specific layer

```bash
cd {BASE_PATH} && go test ./internal/repository/... -v
cd {BASE_PATH} && go test ./internal/service/... -v
cd {BASE_PATH} && go test ./internal/handler/... -v
cd {BASE_PATH} && go test ./internal/entity/... -v
cd {BASE_PATH} && go test ./internal/dto/... -v
cd {BASE_PATH} && go test ./internal/middleware/... -v
```

### Run single test

```bash
cd {BASE_PATH} && go test ./internal/repository/ -v -run TestNotificationRepository_FindByID
```

### Run with coverage

```bash
cd {BASE_PATH} && go test ./... -cover -coverprofile=coverage.out
cd {BASE_PATH} && go tool cover -html=coverage.out -o coverage.html
```

### Run with race detector

```bash
cd {BASE_PATH} && go test ./... -race -v
```

## Checklist

- [ ] Every implementation file has a corresponding `*_test.go` file
- [ ] Tests written BEFORE implementation (TDD: red → green → commit)
- [ ] `github.com/stretchr/testify/assert` used for assertions
- [ ] `github.com/stretchr/testify/mock` used for mock objects
- [ ] Entity helpers tested with table-driven tests
- [ ] Repository tests use real DB (SQLite in-memory) for integration
- [ ] Service tests use mock repositories (no DB dependency)
- [ ] Handler tests use `httptest.NewRecorder()` and Echo context
- [ ] Middleware tests verify JWT and S2S token validation
- [ ] `app_origin` filter tested — wrong app_origin returns not found
- [ ] Error wrapping tested — verify error messages contain "repository:", "service:"
- [ ] Test utilities in `internal/testutil/` shared across test files
- [ ] All tests pass: `cd {BASE_PATH} && go test ./... -v`
- [ ] No race conditions: `cd {BASE_PATH} && go test ./... -race`
- [ ] Coverage > 70%: `cd {BASE_PATH} && go test ./... -cover`