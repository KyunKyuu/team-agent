---
name: go-repository
description: |
  Write GORM repository implementations for Go microservices in the jr-external/jr-web-partner rebuild project.
  Use this skill when creating or modifying repository files in internal/repository/.
  Covers GORM query patterns, app_origin filtering, error wrapping, clause.OnConflict upsert, pagination, and compile-time interface checks.
  Trigger when: writing repository implementation, creating GORM queries, implementing data access layer, building CRUD operations with GORM.
---

# Go Repository Skill

## Principles

- **Thin data access**: Repository does CRUD + queries only. No business logic, no service calls.
- **Error wrapping**: All errors wrapped with `fmt.Errorf("repository: %w", err)` for traceability.
- **app_origin filter**: Every query on a unified table MUST filter by `app_origin`. Non-negotiable.
- **Context propagation**: Every method takes `ctx context.Context` as first param. Use it for GORM's `WithContext(ctx)`.
- **Max 300-500 lines per file**. Split by entity if needed.
- **Compile-time interface check**: `var _ Interface = (*Impl)(nil)` at top of file.

## File Structure

```
internal/repository/
├── notification_repository.go    ← NotificationRepository implementation
├── device_token_repository.go    ← DeviceTokenRepository implementation
├── preference_repository.go     ← PreferenceRepository implementation
└── health_repository.go         ← HealthRepository implementation
```

## Implementation Pattern

```go
package repository

import (
    "context"
    "fmt"

    "gorm.io/gorm"
    "gorm.io/gorm/clause"

    domainRepo "myapp/internal/domain/repository"
    "myapp/internal/entity"
)

// Compile-time interface check
var _ domainRepo.NotificationRepository = (*notificationRepository)(nil)

type notificationRepository struct {
    db        *gorm.DB
    appOrigin string // "web_partner" or "external"
}

func NewNotificationRepository(db *gorm.DB, appOrigin string) domainRepo.NotificationRepository {
    return &notificationRepository{db: db, appOrigin: appOrigin}
}
```

## CRUD Operations

### Create

```go
func (r *notificationRepository) Create(ctx context.Context, notif *entity.Notification) error {
    if err := r.db.WithContext(ctx).Create(notif).Error; err != nil {
        return fmt.Errorf("repository: create notification: %w", err)
    }
    return nil
}
```

### FindByID

```go
func (r *notificationRepository) FindByID(ctx context.Context, id string) (*entity.Notification, error) {
    var notif entity.Notification
    err := r.db.WithContext(ctx).
        Where("id = ? AND app_origin = ?", id, r.appOrigin).
        First(&notif).Error
    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, fmt.Errorf("repository: notification not found: %w", err)
        }
        return nil, fmt.Errorf("repository: find notification by id: %w", err)
    }
    return &notif, nil
}
```

### FindByUserID (Paginated)

```go
func (r *notificationRepository) FindByUserID(ctx context.Context, userID string, page, limit int) ([]entity.Notification, int64, error) {
    var notifs []entity.Notification
    var total int64

    query := r.db.WithContext(ctx).
        Model(&entity.Notification{}).
        Where("user_id = ? AND app_origin = ?", userID, r.appOrigin)

    if err := query.Count(&total).Error; err != nil {
        return nil, 0, fmt.Errorf("repository: count notifications: %w", err)
    }

    offset := (page - 1) * limit
    if err := query.Order("created_at DESC").Offset(offset).Limit(limit).Find(&notifs).Error; err != nil {
        return nil, 0, fmt.Errorf("repository: find notifications by user: %w", err)
    }

    return notifs, total, nil
}
```

### Update

```go
func (r *notificationRepository) MarkRead(ctx context.Context, id string) error {
    now := time.Now().UTC()
    result := r.db.WithContext(ctx).
        Model(&entity.Notification{}).
        Where("id = ? AND app_origin = ?", id, r.appOrigin).
        Updates(map[string]interface{}{
            "is_read":   true,
            "status":    "read",
            "read_at":   &now,
            "updated_at": &now,
        })
    if result.Error != nil {
        return fmt.Errorf("repository: mark notification read: %w", result.Error)
    }
    if result.RowsAffected == 0 {
        return fmt.Errorf("repository: notification not found for mark read")
    }
    return nil
}
```

### Upsert with clause.OnConflict

For entities that need insert-or-update (e.g., DeviceToken, Preference):

```go
func (r *deviceTokenRepository) Upsert(ctx context.Context, token *entity.DeviceToken) error {
    err := r.db.WithContext(ctx).
        Clauses(clause.OnConflict{
            Columns:   []clause.Column{{Name: "user_id"}, {Name: "app_origin"}},
            DoUpdates: clause.AssignmentColumns([]string{"token", "platform", "updated_at"}),
        }).
        Create(token).Error
    if err != nil {
        return fmt.Errorf("repository: upsert device token: %w", err)
    }
    return nil
}
```

### Count

```go
func (r *notificationRepository) CountUnread(ctx context.Context, userID string) (int64, error) {
    var count int64
    err := r.db.WithContext(ctx).
        Model(&entity.Notification{}).
        Where("user_id = ? AND is_read = false AND app_origin = ?", userID, r.appOrigin).
        Count(&count).Error
    if err != nil {
        return 0, fmt.Errorf("repository: count unread notifications: %w", err)
    }
    return count, nil
}
```

### Exists

```go
func (r *deviceTokenRepository) ExistsByToken(ctx context.Context, token string) (bool, error) {
    var count int64
    err := r.db.WithContext(ctx).
        Model(&entity.DeviceToken{}).
        Where("token = ? AND app_origin = ?", token, r.appOrigin).
        Count(&count).Error
    if err != nil {
        return false, fmt.Errorf("repository: check token exists: %w", err)
    }
    return count > 0, nil
}
```

## Key Patterns

### app_origin Filter (MANDATORY)

Every query on a unified table MUST include `app_origin` filter:

```go
// CORRECT — always filter by app_origin
r.db.Where("user_id = ? AND app_origin = ?", userID, r.appOrigin)

// WRONG — missing app_origin filter
r.db.Where("user_id = ?", userID)
```

The `appOrigin` field is set in the constructor:
- `jr-web-partner` → `appOrigin = "web_partner"`
- `jr-external` → `appOrigin = "external"`

### Error Wrapping

All repository errors use `fmt.Errorf("repository: <operation>: %w", err)`:

```go
// CORRECT
return nil, fmt.Errorf("repository: find notification by id: %w", err)

// WRONG — no wrapping, loses traceability
return nil, err

// WRONG — wrong prefix
return nil, fmt.Errorf("db error: %w", err)
```

### GORM ErrRecordNotFound

Distinguish between "not found" (expected) and real errors:

```go
if errors.Is(err, gorm.ErrRecordNotFound) {
    return nil, fmt.Errorf("repository: notification not found: %w", err)
}
return nil, fmt.Errorf("repository: find notification: %w", err)
```

### Transactions

Use transactions only when multiple writes must be atomic:

```go
func (r *notificationRepository) CreateWithPreference(ctx context.Context, notif *entity.Notification, pref *entity.Preference) error {
    return r.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        if err := tx.Create(notif).Error; err != nil {
            return fmt.Errorf("repository: create notification in tx: %w", err)
        }
        if err := tx.Save(pref).Error; err != nil {
            return fmt.Errorf("repository: save preference in tx: %w", err)
        }
        return nil
    })
}
```

### Nullable Fields

When querying nullable fields, use the column name not the Go field:

```go
// Find records where read_at IS NOT NULL
r.db.Where("read_at IS NOT NULL AND app_origin = ?", r.appOrigin)

// Find records where company_id IS NOT NULL
r.db.Where("company_id IS NOT NULL AND app_origin = ?", r.appOrigin)
```

## Checklist

- [ ] Compile-time interface check at top of file
- [ ] `appOrigin` field in struct, set via constructor
- [ ] Every unified table query filters by `app_origin`
- [ ] All errors wrapped with `fmt.Errorf("repository: <op>: %w", err)`
- [ ] `ctx context.Context` propagated via `r.db.WithContext(ctx)`
- [ ] GORM `ErrRecordNotFound` handled separately from other errors
- [ ] Pagination returns `([]entity, int64, error)` with total count
- [ ] `clause.OnConflict` for upsert operations
- [ ] Transactions only when atomic multi-write needed
- [ ] File under 300-500 lines (split by entity if needed)