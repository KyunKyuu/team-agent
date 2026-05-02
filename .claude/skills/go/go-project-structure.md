---
name: go-project-structure
description: |
  Define Go microservice folder structure and shared packages for the jr-external/jr-web-partner rebuild project.
  Use this skill when creating a new service, setting up project directories, or writing shared package code (pkg/errors, pkg/response, pkg/logger, pkg/pagination).
  Covers directory layout, shared package patterns, and project initialization.
  Trigger when: creating new service scaffold, setting up folder structure, writing pkg/errors, writing pkg/response, writing pkg/logger, initializing Go module.
---

# Go Project Structure Skill

## Principles

- **Clean Architecture**: Entity → Domain (interfaces) → Repository → Service → Handler → Routes.
- **Shared packages in pkg/**: Error, response, logger, pagination — reusable across services.
- **Domain interfaces consumed where needed**: Repo interfaces in `domain/repository/` (consumed by service), Service interfaces in `domain/service/` (consumed by handler).
- **Max 300-500 lines per file**.
- **Module path**: `repository.jasaraharja.co.id/jasaraharja/jrku/service/{scope}-{service}`.

## Directory Structure

```
apps/be/{scope}/rebuild/{service}/
├── main.go                              ← Entry point (numbered steps)
├── go.mod                               ← Module definition
├── go.sum                               ← Dependency checksums
├── .env.example                         ← Environment variable template
├── Dockerfile                           ← Container build
├── docker-compose.yml                   ← Local development
│
├── internal/
│   ├── config/
│   │   └── config.go                    ← Config struct + Load() + helpers
│   │
│   ├── database/
│   │   └── database.go                  ← GORM init + connection pool
│   │
│   ├── domain/                          ← INTERFACE ONLY (no implementations)
│   │   ├── repository/
│   │   │   ├── notification_repository.go
│   │   │   ├── device_token_repository.go
│   │   │   └── preference_repository.go
│   │   └── service/
│   │       ├── notification_service.go
│   │       ├── device_token_service.go
│   │       └── preference_service.go
│   │
│   ├── entity/                          ← GORM models + helpers
│   │   ├── notification.go
│   │   ├── notification_helpers.go      ← (if needed, split from main)
│   │   ├── event_type_mapping.go       ← (if shared across entities)
│   │   ├── device_token.go
│   │   ├── preference.go
│   │   └── test/
│   │       └── notification_test.go     ← White-box: package entity
│   │
│   ├── dto/                             ← Request/Response DTOs + mappers
│   │   ├── common.go                    ← CustomValidator
│   │   ├── pagination.go                ← Pagination helper
│   │   ├── mapper.go                    ← Entity-to-DTO mapping
│   │   ├── notification.go
│   │   ├── device_token.go
│   │   ├── preference.go
│   │   └── test/
│   │       └── notification_test.go     ← White-box: package dto
│   │
│   ├── repository/                      ← Repository IMPLEMENTATIONS
│   │   ├── notification_repository.go
│   │   ├── device_token_repository.go
│   │   ├── preference_repository.go
│   │   └── test/
│   │       └── notification_repository_test.go  ← Black-box: package repository_test
│   │
│   ├── service/                         ← Service IMPLEMENTATIONS
│   │   ├── notification_service.go
│   │   ├── device_token_service.go
│   │   ├── preference_service.go
│   │   └── test/
│   │       └── notification_service_test.go      ← Black-box: package service_test
│   │
│   ├── handler/                         ← HTTP handlers (Echo)
│   │   ├── notification_handler.go
│   │   ├── preference_handler.go
│   │   ├── health_handler.go
│   │   └── test/
│   │       └── notification_handler_test.go      ← Black-box: package handler_test
│   │
│   ├── rpc/                             ← RabbitMQ RPC handlers
│   │   ├── server.go                    ← RPC server setup
│   │   ├── handlers.go                  ← Individual RPC handlers
│   │   └── test/
│   │       └── handlers_test.go         ← Black-box: package rpc_test
│   │
│   ├── middleware/                       ← Auth + security middleware
│   │   ├── middleware.go
│   │   └── test/
│   │       └── middleware_test.go       ← Black-box: package middleware_test
│   │
│   └── routes/
│       └── routes.go                    ← Route registration
│
│   └── testutil/                         ← Shared test utilities
│       ├── db.go                         ← Test DB setup/teardown (SQLite in-memory)
│       ├── fixture.go                    ← Test data factories
│       └── mock_repository.go            ← Generated or hand-written mocks
│
├── pkg/                                 ← Shared packages
│   ├── errors/
│   │   └── errors.go                    ← HTTPError + factory methods
│   ├── response/
│   │   └── response.go                  ← BaseResponse + helpers
│   ├── logger/
│   │   └── logger.go                    ← logrus + lumberjack
│   └── utils/
│       └── utils.go                     ← Shared utilities
│
├── docs/                                ← Documentation
│   └── gorm_automigrate_reference.go    ← GORM reference (not executed)
│
└── migrations/                          ← SQL migration files
    ├── 00001_create_notifications_table.sql
    ├── 00002_create_device_tokens_table.sql
    ├── 00003_create_preferences_table.sql
    └── 00004_create_indexes.sql
```

## Shared Packages

### pkg/errors — HTTPError

```go
package errors

import (
    "fmt"
    "net/http"
)

type HTTPError struct {
    Code    int
    Message string
}

func (e *HTTPError) Error() string {
    return e.Message
}

func NewBadRequest(message string) *HTTPError {
    return &HTTPError{Code: http.StatusBadRequest, Message: message}
}

func NewUnauthorized(message string) *HTTPError {
    return &HTTPError{Code: http.StatusUnauthorized, Message: message}
}

func NewForbidden(message string) *HTTPError {
    return &HTTPError{Code: http.StatusForbidden, Message: message}
}

func NewNotFound(message string) *HTTPError {
    return &HTTPError{Code: http.StatusNotFound, Message: message}
}

func NewConflict(message string) *HTTPError {
    return &HTTPError{Code: http.StatusConflict, Message: message}
}

func NewInternalServer(message string) *HTTPError {
    return &HTTPError{Code: http.StatusInternalServerError, Message: message}
}

func NewServiceUnavailable(message string) *HTTPError {
    return &HTTPError{Code: http.StatusServiceUnavailable, Message: message}
}
```

### pkg/response — BaseResponse

```go
package response

import (
    "net/http"

    "github.com/labstack/echo/v4"
)

type BaseResponse struct {
    Code    int         `json:"code"`
    Status  string      `json:"status"`
    Message string      `json:"message"`
    Data    interface{} `json:"data"`
}

func Success(c echo.Context, data interface{}) error {
    return c.JSON(http.StatusOK, BaseResponse{
        Code: http.StatusOK, Status: http.StatusText(http.StatusOK), Data: data,
    })
}

func SuccessWithMessage(c echo.Context, message string, data interface{}) error {
    return c.JSON(http.StatusOK, BaseResponse{
        Code: http.StatusOK, Status: http.StatusText(http.StatusOK), Message: message, Data: data,
    })
}

func Created(c echo.Context, message string, data interface{}) error {
    return c.JSON(http.StatusCreated, BaseResponse{
        Code: http.StatusCreated, Status: http.StatusText(http.StatusCreated), Message: message, Data: data,
    })
}

func Error(c echo.Context, code int, message string) error {
    return c.JSON(code, BaseResponse{
        Code: code, Status: http.StatusText(code), Message: message, Data: struct{}{},
    })
}

func BadRequest(c echo.Context, message string) error { return Error(c, http.StatusBadRequest, message) }
func Unauthorized(c echo.Context, message string) error { return Error(c, http.StatusUnauthorized, message) }
func Forbidden(c echo.Context, message string) error { return Error(c, http.StatusForbidden, message) }
func NotFound(c echo.Context, message string) error { return Error(c, http.StatusNotFound, message) }
func Conflict(c echo.Context, message string) error { return Error(c, http.StatusConflict, message) }
func InternalServerError(c echo.Context, message string) error { return Error(c, http.StatusInternalServerError, message) }
```

### pkg/logger — Logrus + Lumberjack

Singleton pattern with `sync.Once`:

```go
package logger

import (
    "io"
    "os"
    "path/filepath"
    "sync"

    "github.com/sirupsen/logrus"
    "gopkg.in/natefinch/lumberjack.v2"
)

var (
    logger *logrus.Logger
    once   sync.Once
)

type Config struct {
    Level      string
    Format     string
    Output     string
    FilePath   string
    MaxSize    int
    MaxBackups int
    MaxAge     int
    Compress   bool
}

func Init(cfg Config) error {
    var initErr error
    once.Do(func() {
        logger = logrus.New()

        level, _ := logrus.ParseLevel(cfg.Level)
        logger.SetLevel(level)

        if cfg.Format == "json" {
            logger.SetFormatter(&logrus.JSONFormatter{
                TimestampFormat: "2006-01-02T15:04:05.999Z07:00",
            })
        } else {
            logger.SetFormatter(&logrus.TextFormatter{
                FullTimestamp:   true,
                TimestampFormat: "2006-01-02T15:04:05.999Z07:00",
            })
        }

        var output io.Writer
        if cfg.Output == "stdout" || cfg.Output == "" {
            output = os.Stdout
        } else {
            _ = os.MkdirAll(filepath.Dir(cfg.FilePath), 0755)
            output = &lumberjack.Logger{
                Filename: cfg.FilePath, MaxSize: cfg.MaxSize,
                MaxBackups: cfg.MaxBackups, MaxAge: cfg.MaxAge, Compress: cfg.Compress,
            }
        }
        logger.SetOutput(output)
    })
    return initErr
}

func GetLogger() *logrus.Logger {
    if logger == nil {
        return logrus.StandardLogger()
    }
    return logger
}
```

## Go Module

```
module repository.jasaraharja.co.id/jasaraharja/jrku/service/jrku-wp-notification
```

Key dependencies:
- `github.com/labstack/echo/v4` — HTTP framework
- `gorm.io/gorm` + `gorm.io/driver/postgres` — ORM
- `github.com/golang-jwt/jwt/v5` — JWT
- `github.com/go-playground/validator/v10` — Validation
- `github.com/rabbitmq/amqp091-go` — RabbitMQ
- `github.com/sirupsen/logrus` — Logging
- `gopkg.in/natefinch/lumberjack.v2` — Log rotation
- `github.com/joho/godotenv` — Env loading

## Test Directory Convention

Tests live in separate `test/` directories, NOT alongside source files. This keeps production code clean and follows Go best practices for both white-box and black-box testing.

| Layer | Test Directory | Package | Access | Why |
|-------|---------------|---------|--------|-----|
| Entity helpers | `internal/entity/test/` | `package entity` | Private + Public | Need access to unexported helpers like `getLegacyType()` |
| DTO mappers | `internal/dto/test/` | `package dto` | Private + Public | Need access to unexported mapper functions |
| Repository | `internal/repository/test/` | `package repository_test` | Public only | Test public interface, real DB (SQLite in-memory) |
| Service | `internal/service/test/` | `package service_test` | Public only | Test public interface, mock repos |
| Handler (HTTP) | `internal/handler/test/` | `package handler_test` | Public only | Test HTTP API via httptest |
| RPC handlers | `internal/rpc/test/` | `package rpc_test` | Public only | Test RPC message handling |
| Middleware | `internal/middleware/test/` | `package middleware_test` | Public only | Test middleware chain |

**Shared test utilities** go in `internal/testutil/`:
- `db.go` — SQLite in-memory setup/teardown for integration tests
- `fixture.go` — Test data factory functions
- `mock_repository.go` — Hand-written or generated mocks for service tests

**Run commands:**
```bash
# Run all tests
cd {BASE_PATH} && go test ./... -v

# Run specific layer
cd {BASE_PATH} && go test ./internal/entity/... -v
cd {BASE_PATH} && go test ./internal/repository/... -v
cd {BASE_PATH} && go test ./internal/service/... -v

# Run with coverage
cd {BASE_PATH} && go test ./... -cover -coverprofile=coverage.out
```

## Checklist

- [ ] Directory structure matches the layout above
- [ ] `internal/domain/` contains ONLY interfaces (no implementations)
- [ ] `internal/repository/` and `internal/service/` contain implementations
- [ ] `pkg/` contains shared packages (errors, response, logger, utils)
- [ ] `internal/testutil/` contains shared test utilities (db.go, fixture.go, mocks)
- [ ] Tests in separate `test/` directories (NOT alongside source files)
- [ ] White-box tests use same package name (entity, dto)
- [ ] Black-box tests use `_test` package suffix (repository_test, service_test, handler_test, middleware_test)
- [ ] `docs/gorm_automigrate_reference.go` exists (documentation only)
- [ ] `migrations/` contains numbered SQL files
- [ ] `.env.example` matches config struct
- [ ] `go.mod` has correct module path
- [ ] No AutoMigrate in main.go
- [ ] Each file under 300-500 lines