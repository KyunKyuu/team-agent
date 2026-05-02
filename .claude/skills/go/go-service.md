---
name: go-service
description: |
  Write service layer implementations for Go microservices in the jr-external/jr-web-partner rebuild project.
  Use this skill when creating or modifying service files in internal/service/.
  Covers business logic patterns, cross-entity dependencies, error handling with HTTPError, logger usage, DTO mapping, and compile-time interface checks.
  Trigger when: writing service implementation, adding business logic, implementing service interface, creating service constructor, handling cross-entity dependencies.
---

# Go Service Skill

## Principles

- **Business logic lives here**: Validation, orchestration, preference checking, dispatch logic.
- **Thin handlers, fat services**: Handlers parse request and call service. Services do the real work.
- **Error handling**: Use `pkg/errors` HTTPError for domain errors, `fmt.Errorf` for internal wrapping.
- **Logger**: Use `*logrus.Logger` for structured logging. Log at service boundaries (entry/exit, errors).
- **Max 300-500 lines per file**. Split by entity or by operation group if needed.
- **Cross-entity via interface**: Inject `PreferenceService` and `DeviceTokenService` as interfaces, not concrete types.
- **Compile-time interface check**: `var _ Interface = (*Impl)(nil)` at top of file.

## File Structure

```
internal/service/
├── notification_service.go       ← NotificationService implementation
├── notification_service_send.go  ← Send/dispatch logic (if main file >300 lines)
├── device_token_service.go       ← DeviceTokenService implementation
├── preference_service.go         ← PreferenceService implementation
└── health_service.go             ← HealthService implementation
```

## Implementation Pattern

```go
package service

import (
    "context"
    "fmt"

    "github.com/sirupsen/logrus"

    domainRepo "myapp/internal/domain/repository"
    domainSvc "myapp/internal/domain/service"
    "myapp/internal/dto"
    appErrors "myapp/pkg/errors"
)

// Compile-time interface check
var _ domainSvc.NotificationService = (*notificationService)(nil)

type notificationService struct {
    notifRepo  domainRepo.NotificationRepository
    prefSvc    domainSvc.PreferenceService
    tokenSvc   domainSvc.DeviceTokenService
    logger     *logrus.Logger
}

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

## Method Patterns

### Simple Get (Delegating to Repository)

```go
func (s *notificationService) GetNotifications(ctx context.Context, userID string, req *dto.GetNotificationsRequest) (*dto.NotificationListResponse, error) {
    notifs, total, err := s.notifRepo.FindByUserID(ctx, userID, req.Page, req.Limit)
    if err != nil {
        s.logger.Errorf("Failed to get notifications for user %s: %v", userID, err)
        return nil, appErrors.NewInternalServer("Failed to get notifications")
    }

    items := make([]dto.NotificationItemResponse, len(notifs))
    for i := range notifs {
        items[i] = dto.NotificationToResponse(&notifs[i])
    }

    return &dto.NotificationListResponse{
        Data:  items,
        Total: total,
        Page:  req.Page,
        Limit: req.Limit,
    }, nil
}
```

### Business Logic with Cross-Entity Dependency

```go
func (s *notificationService) SendNotification(ctx context.Context, req *dto.SendNotificationManifestRequest) error {
    // 1. Determine event type
    eventType := entity.LegacyToEventType(req.NotificationType)

    // 2. Build notification entity
    notif := &entity.Notification{
        UserID:    req.UserID,
        EventType: eventType,
        Status:    "unread",
        AppOrigin: "web_partner",
    }

    // 3. Save to database
    if err := s.notifRepo.Create(ctx, notif); err != nil {
        s.logger.Errorf("Failed to create notification: %v", err)
        return appErrors.NewInternalServer("Failed to send notification")
    }

    // 4. Check preferences before sending push/email
    prefs, err := s.prefSvc.GetPreferences(ctx, req.UserID)
    if err != nil {
        s.logger.Warnf("Failed to get preferences for user %s, skipping push: %v", req.UserID, err)
        // Don't fail the whole operation if preferences check fails
    }

    // 5. Send FCM push if preference enabled
    if prefs != nil && prefs.PushEnabled {
        if err := s.sendPushNotification(ctx, req.UserID, notif); err != nil {
            s.logger.Warnf("Failed to send push notification: %v", err)
            // Don't fail the whole operation if push fails
        }
    }

    return nil
}
```

### Update with Validation

```go
func (s *preferenceService) UpdatePreferences(ctx context.Context, userID string, req *dto.UpdatePreferenceRequest) error {
    // 1. Check if preferences exist
    pref, err := s.prefRepo.FindByUserID(ctx, userID)
    if err != nil {
        s.logger.Errorf("Failed to find preferences for user %s: %v", userID, err)
        return appErrors.NewNotFound("Preferences not found")
    }

    // 2. Apply partial updates (*bool pattern)
    if req.MenungguPembayaranNotif != nil {
        pref.InvoiceCreatedEnabled = *req.MenungguPembayaranNotif
    }
    if req.PembayaranSelesaiNotif != nil {
        pref.InvoicePaymentSuccessEnabled = *req.PembayaranSelesaiNotif
    }
    if req.EmailEnabled != nil {
        pref.EmailEnabled = req.EmailEnabled // *bool stays pointer
    }

    // 3. Save
    if err := s.prefRepo.Update(ctx, pref); err != nil {
        s.logger.Errorf("Failed to update preferences for user %s: %v", userID, err)
        return appErrors.NewInternalServer("Failed to update preferences")
    }

    return nil
}
```

## Error Handling

Use `pkg/errors` HTTPError for domain-level errors that map to HTTP responses:

```go
// Domain errors — these will be returned to the client
return nil, appErrors.NewBadRequest("Invalid notification type")
return nil, appErrors.NewUnauthorized("Invalid token")
return nil, appErrors.NewNotFound("Notification not found")
return nil, appErrors.NewForbidden("Insufficient permissions")
return nil, appErrors.NewInternalServer("Failed to process request")

// Internal wrapping — for logging and debugging
s.logger.Errorf("Failed to create notification: %v", err)
return appErrors.NewInternalServer("Failed to send notification")
```

**Rules:**
- Never return raw `err` from repository — always wrap with an HTTPError or log it
- Log the internal error for debugging, return a sanitized error to the client
- `BadRequest` for validation errors (400)
- `NotFound` for missing resources (404)
- `Unauthorized` for auth failures (401)
- `Forbidden` for permission failures (403)
- `InternalServer` for unexpected failures (500)

## File Splitting

When service file grows >300 lines, split by operation group:

```
internal/service/
├── notification_service.go        ← Struct definition + constructor + simple methods
├── notification_service_send.go  ← Send/Dispatch logic
├── notification_service_query.go ← Get/List/Count/MarkRead
└── notification_service_email.go ← Email dispatch (if applicable)
```

All split files share the same package and operate on the same `notificationService` struct:

```go
// notification_service_send.go
package service

func (s *notificationService) SendNotification(ctx context.Context, req *dto.SendNotificationManifestRequest) error {
    // ...
}

func (s *notificationService) sendPushNotification(ctx context.Context, userID string, notif *entity.Notification) error {
    // ...
}
```

## Checklist

- [ ] Compile-time interface check at top of file
- [ ] Struct holds interfaces (not concrete types) for cross-entity deps
- [ ] Constructor injects all dependencies
- [ ] `*logrus.Logger` for structured logging
- [ ] Domain errors use `pkg/errors` HTTPError
- [ ] Repository errors are logged then wrapped with HTTPError
- [ ] Partial updates use `*bool` pattern (nil = don't change)
- [ ] DTO mapping via mapper functions, not inline
- [ ] `ctx context.Context` propagated to all repository calls
- [ ] File under 300-500 lines (split by operation group if needed)