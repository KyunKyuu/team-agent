---
name: go-domain-interface
description: |
  Write repository and service interfaces for Go microservices in the jr-external/jr-web-partner rebuild project.
  Use this skill when creating or modifying interface files in internal/domain/repository/ or internal/domain/service/.
  Covers Interface Segregation Principle (ISP), small focused interfaces, Go interface conventions, and consuming interfaces where they're used.
  Trigger when: writing interface definitions, creating domain contracts, defining repository or service interfaces, setting up dependency injection contracts.
---

# Go Domain Interface Skill

## Principles

- **Interface Segregation Principle (ISP)**: Max 5 methods per interface. Split if more.
- **Consume interfaces where they're used**: Repository interfaces are consumed by the service layer, so they live in `domain/repository/`. Service interfaces are consumed by handlers, so they live in `domain/service/`.
- **Go convention**: Interfaces should be small and focused. "The bigger the interface, the weaker the abstraction." — Rob Pike
- **Return structs, accept interfaces**: Factory functions return concrete types for implementations, but accept interfaces as dependencies.
- **Context-first**: Every method takes `ctx context.Context` as the first parameter.

## File Structure

```
internal/domain/
├── repository/
│   ├── notification_repository.go   ← NotificationRepository interface
│   ├── device_token_repository.go  ← DeviceTokenRepository interface
│   └── preference_repository.go    ← PreferenceRepository interface
└── service/
    ├── notification_service.go     ← NotificationService interface
    ├── device_token_service.go     ← DeviceTokenService interface
    └── preference_service.go       ← PreferenceService interface
```

## Repository Interfaces

Repository interfaces define data access contracts. They are consumed by the service layer.

```go
package repository

import (
    "context"

    "myapp/internal/entity"
)

// NotificationRepository defines data access for notifications.
// Consumed by: NotificationService
type NotificationRepository interface {
    Create(ctx context.Context, notif *entity.Notification) error
    FindByID(ctx context.Context, id string) (*entity.Notification, error)
    FindByUserID(ctx context.Context, userID string, page, limit int) ([]entity.Notification, int64, error)
    MarkRead(ctx context.Context, id string) error
    CountUnread(ctx context.Context, userID string) (int64, error)
}

// NotificationBulkRepository for bulk operations — split when >5 methods.
type NotificationBulkRepository interface {
    CreateBulk(ctx context.Context, notifs []entity.Notification) error
}
```

**Rules for repository interfaces:**
- Method names: `Create`, `FindByID`, `FindByUserID`, `Update`, `Delete`, `Count`, `Exists`
- Prefix `Find` for queries returning entities, `Count` for counts, `Exists` for booleans
- Pagination: `page, limit int` as last params, return `([]entity, int64, error)` where int64 = total count
- Always include `ctx context.Context` as first param
- Use entity types from `internal/entity/`, NOT DTOs
- If an entity needs >5 repository methods, split into two interfaces (e.g., `Read` + `Write`)

## Service Interfaces

Service interfaces define business logic contracts. They are consumed by handlers.

```go
package service

import (
    "context"

    "myapp/internal/dto"
)

// NotificationService defines business logic for notifications.
// Consumed by: HTTP handlers, RPC handlers
type NotificationService interface {
    SendNotification(ctx context.Context, req *dto.SendNotificationManifestRequest) error
    SendNotificationBulk(ctx context.Context, req *dto.SendNotificationBoronganRequest) error
    GetNotifications(ctx context.Context, userID string, req *dto.GetNotificationsRequest) (*dto.NotificationListResponse, error)
    MarkRead(ctx context.Context, notifID string) error
    CountUnread(ctx context.Context, userID string) (int64, error)
}

// DeviceTokenService defines business logic for device tokens.
// Consumed by: RPC handlers
type DeviceTokenService interface {
    SaveToken(ctx context.Context, userID, token string) error
    DeleteToken(ctx context.Context, token string) error
}

// PreferenceService defines business logic for notification preferences.
// Consumed by: HTTP handlers, RPC handlers, NotificationService (cross-entity)
type PreferenceService interface {
    GetPreferences(ctx context.Context, userID string) (*dto.PreferenceResponse, error)
    UpdatePreferences(ctx context.Context, userID string, req *dto.UpdatePreferenceRequest) error
}
```

**Rules for service interfaces:**
- Method names: verbs describing the business operation (`Send`, `Get`, `Mark`, `Save`, `Update`)
- Input: DTO request types from `internal/dto/`
- Output: DTO response types from `internal/dto/` or `error`
- `userID string` as explicit param when needed from auth context (not inside DTO)
- Cross-entity dependencies: `NotificationService` may call `PreferenceService` and `DeviceTokenService` — inject via interface, not concrete

## Interface Splitting Strategy

When an entity needs many operations, split by responsibility:

```go
// GOOD: Small focused interfaces
type NotificationReader interface {
    FindByID(ctx context.Context, id string) (*entity.Notification, error)
    FindByUserID(ctx context.Context, userID string, page, limit int) ([]entity.Notification, int64, error)
    CountUnread(ctx context.Context, userID string) (int64, error)
}

type NotificationWriter interface {
    Create(ctx context.Context, notif *entity.Notification) error
    MarkRead(ctx context.Context, id string) error
}

// BAD: One giant interface
type NotificationRepository interface {
    Create(ctx context.Context, notif *entity.Notification) error
    FindByID(ctx context.Context, id string) (*entity.Notification, error)
    FindByUserID(ctx context.Context, userID string, page, limit int) ([]entity.Notification, int64, error)
    MarkRead(ctx context.Context, id string) error
    CountUnread(ctx context.Context, userID string) (int64, error)
    CreateBulk(ctx context.Context, notifs []entity.Notification) error
    DeleteOlderThan(ctx context.Context, before time.Time) error
    // ... keeps growing
}
```

**When to split:**
- >5 methods → split immediately
- Read vs Write separation → natural split point
- Bulk operations → separate interface
- Admin/management operations → separate interface

## Implementation Binding

Implementations in `internal/repository/` and `internal/service/` satisfy these interfaces:

```go
// internal/repository/notification_repository.go
type notificationRepository struct {
    db *gorm.DB
}

// Compile-time check: interface satisfaction
var _ domainRepo.NotificationRepository = (*notificationRepository)(nil)

func NewNotificationRepository(db *gorm.DB) domainRepo.NotificationRepository {
    return &notificationRepository{db: db}
}
```

```go
// internal/service/notification_service.go
type notificationService struct {
    notifRepo  domainRepo.NotificationRepository
    prefSvc    domainSvc.PreferenceService   // cross-entity: accept interface
    tokenSvc   domainSvc.DeviceTokenService   // cross-entity: accept interface
    logger     *logrus.Logger
}

// Compile-time check
var _ domainSvc.NotificationService = (*notificationService)(nil)

func NewNotificationService(
    notifRepo domainRepo.NotificationRepository,
    prefSvc domainSvc.PreferenceService,
    tokenSvc domainSvc.DeviceTokenService,
    logger *logrus.Logger,
) domainSvc.NotificationService {
    return &notificationService{
        notifRepo: notifRepo,
        prefSvc:   prefSvc,
        tokenSvc:  tokenSvc,
        logger:    logger,
    }
}
```

## Checklist

- [ ] Max 5 methods per interface (ISP)
- [ ] Repository interfaces in `internal/domain/repository/`
- [ ] Service interfaces in `internal/domain/service/`
- [ ] Every method has `ctx context.Context` as first param
- [ ] Repository methods use entity types, NOT DTOs
- [ ] Service methods use DTO types for input/output
- [ ] Cross-entity dependencies via interface injection
- [ ] Compile-time interface satisfaction check (`var _ Interface = (*Impl)(nil)`)
- [ ] Factory functions return interface type
- [ ] No business logic in interfaces (only signatures)