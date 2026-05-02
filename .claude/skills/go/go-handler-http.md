---
name: go-handler-http
description: |
  Write Echo HTTP handler implementations for Go microservices in the jr-external/jr-web-partner rebuild project.
  Use this skill when creating or modifying handler files in internal/handler/.
  Covers thin handler pattern, request binding/validation, response formatting with BaseResponse, context extraction, and error handling.
  Trigger when: writing HTTP handler, creating Echo route handler, implementing REST API endpoint, binding request DTOs, formatting API responses.
---

# Go HTTP Handler Skill

## Principles

- **Thin handlers**: Parse request → call service → return response. No business logic.
- **Two auth types**: S2S (x-api-key) for internal endpoints, JWT (accesstoken) for user endpoints.
- **Response format**: Always use `pkg/response` helpers — `Success`, `Created`, `Error`, etc.
- **Error handling**: Let HTTPError propagate to Echo's global error handler. Use `pkg/response` for known errors.
- **Request binding**: Use Echo's `Bind` for path/query params, manual `json.Unmarshal` only if needed.
- **Max 300-500 lines per file**. Split by entity if needed.

## File Structure

```
internal/handler/
├── notification_handler.go   ← S2S + User notification endpoints
├── preference_handler.go     ← User preference endpoints
├── health_handler.go         ← Health check (public)
└── middleware.go              ← Not middleware — just helpers if needed
```

## Handler Struct Pattern

```go
package handler

import (
    "github.com/labstack/echo/v4"

    "myapp/internal/domain/service"
    "myapp/internal/dto"
    "myapp/internal/middleware"
    appErrors "myapp/pkg/errors"
    "myapp/pkg/response"
)

type NotificationHandler struct {
    notifService service.NotificationService
}

func NewNotificationHandler(notifService service.NotificationService) *NotificationHandler {
    return &NotificationHandler{notifService: notifService}
}
```

## S2S Endpoints (Internal API)

S2S endpoints use `x-api-key` header for auth (enforced by route middleware).

### POST — Create/Trigger

```go
// SendManifest handles POST /internal/notifications/send-manifest
// Auth: S2S (x-api-key)
func (h *NotificationHandler) SendManifest(c echo.Context) error {
    ctx := c.Request().Context()

    var req dto.SendNotificationManifestRequest
    if err := c.Bind(&req); err != nil {
        return response.BadRequest(c, "Invalid request body")
    }

    if err := c.Validate(&req); err != nil {
        return response.BadRequest(c, err.Error())
    }

    if err := h.notifService.SendNotification(ctx, &req); err != nil {
        return response.InternalServerError(c, "Failed to send notification")
    }

    return response.Created(c, "Notification sent successfully", nil)
}
```

### POST — Bulk

```go
// SendBorongan handles POST /internal/notifications/send-borongan
// Auth: S2S (x-api-key)
func (h *NotificationHandler) SendBorongan(c echo.Context) error {
    ctx := c.Request().Context()

    var req dto.SendNotificationBoronganRequest
    if err := c.Bind(&req); err != nil {
        return response.BadRequest(c, "Invalid request body")
    }

    if err := c.Validate(&req); err != nil {
        return response.BadRequest(c, err.Error())
    }

    if err := h.notifService.SendNotificationBulk(ctx, &req); err != nil {
        return response.InternalServerError(c, "Failed to send bulk notifications")
    }

    return response.Created(c, "Bulk notifications sent successfully", nil)
}
```

## User Endpoints (Partner API)

User endpoints use `accesstoken` header for auth (enforced by route middleware).

### GET — List with Query Params

```go
// GetNotifications handles GET /partner/notifications
// Auth: JWT (accesstoken)
func (h *NotificationHandler) GetNotifications(c echo.Context) error {
    ctx := c.Request().Context()
    user := middleware.GetUser(c)
    if user == nil {
        return response.Unauthorized(c, "User not authenticated")
    }

    var req dto.GetNotificationsRequest
    if err := c.Bind(&req); err != nil {
        return response.BadRequest(c, "Invalid query parameters")
    }

    // Set defaults if not provided
    if req.Page == 0 {
        req.Page = 1
    }
    if req.Limit == 0 {
        req.Limit = 10
    }

    result, err := h.notifService.GetNotifications(ctx, user.UserID, &req)
    if err != nil {
        return response.InternalServerError(c, "Failed to get notifications")
    }

    return response.Success(c, result)
}
```

### GET — Count

```go
// CountUnread handles GET /partner/notifications/count
// Auth: JWT (accesstoken)
func (h *NotificationHandler) CountUnread(c echo.Context) error {
    ctx := c.Request().Context()
    user := middleware.GetUser(c)
    if user == nil {
        return response.Unauthorized(c, "User not authenticated")
    }

    count, err := h.notifService.CountUnread(ctx, user.UserID)
    if err != nil {
        return response.InternalServerError(c, "Failed to count unread")
    }

    return response.Success(c, map[string]int64{"unread_count": count})
}
```

### PUT — Update with Path Param

```go
// MarkRead handles PUT /partner/notifications/:id/read
// Auth: JWT (accesstoken)
func (h *NotificationHandler) MarkRead(c echo.Context) error {
    ctx := c.Request().Context()

    var req dto.MarkNotificationReadRequest
    if err := c.Bind(&req); err != nil {
        return response.BadRequest(c, "Invalid notification ID")
    }

    if err := c.Validate(&req); err != nil {
        return response.BadRequest(c, err.Error())
    }

    if err := h.notifService.MarkRead(ctx, req.NotificationID); err != nil {
        return response.InternalServerError(c, "Failed to mark notification as read")
    }

    return response.SuccessWithMessage(c, "Notification marked as read", nil)
}
```

### GET + PUT — Preferences

```go
// GetPreferences handles GET /partner/notifications/preferences
func (h *PreferenceHandler) GetPreferences(c echo.Context) error {
    ctx := c.Request().Context()
    user := middleware.GetUser(c)
    if user == nil {
        return response.Unauthorized(c, "User not authenticated")
    }

    result, err := h.prefService.GetPreferences(ctx, user.UserID)
    if err != nil {
        return response.InternalServerError(c, "Failed to get preferences")
    }

    return response.Success(c, result)
}

// UpdatePreferences handles PUT /partner/notifications/preferences
func (h *PreferenceHandler) UpdatePreferences(c echo.Context) error {
    ctx := c.Request().Context()
    user := middleware.GetUser(c)
    if user == nil {
        return response.Unauthorized(c, "User not authenticated")
    }

    var req dto.UpdatePreferenceRequest
    if err := c.Bind(&req); err != nil {
        return response.BadRequest(c, "Invalid request body")
    }

    if err := h.prefService.UpdatePreferences(ctx, user.UserID, &req); err != nil {
        return response.InternalServerError(c, "Failed to update preferences")
    }

    return response.SuccessWithMessage(c, "Preferences updated successfully", nil)
}
```

## Handler-to-Service Contract

Every handler method follows this pattern:

```
1. Extract context:   ctx := c.Request().Context()
2. Get user (JWT):    user := middleware.GetUser(c)
3. Bind request:      c.Bind(&req) or c.Bind(&pathParamReq)
4. Validate:          c.Validate(&req)
5. Call service:      h.service.Method(ctx, ...)
6. Return response:   response.Success/Created/Error
```

**Never:**
- Put business logic in handlers
- Access repositories directly from handlers
- Format raw JSON responses (use `pkg/response`)
- Return raw errors to client (use `pkg/response` helpers)

## Checklist

- [ ] Handler struct holds service interfaces (not concrete types)
- [ ] Constructor: `NewXxxHandler(xxxService service.XxxService) *XxxHandler`
- [ ] Every method follows: bind → validate → service → response
- [ ] JWT endpoints extract user via `middleware.GetUser(c)`
- [ ] S2S endpoints don't need user extraction (no JWT context)
- [ ] All responses via `pkg/response` helpers
- [ ] Errors use `response.BadRequest/Unauthorized/etc` (not raw error)
- [ ] `ctx context.Context` propagated to all service calls
- [ ] File under 300-500 lines