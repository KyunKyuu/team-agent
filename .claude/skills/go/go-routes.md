---
name: go-routes
description: |
  Write Echo route registration for Go microservices in the jr-external/jr-web-partner rebuild project.
  Use this skill when creating or modifying route files in internal/routes/.
  Covers route grouping, middleware application per group, handler registration, and API path conventions.
  Trigger when: writing route registration, setting up API groups, applying middleware to routes, registering handlers with Echo.
---

# Go Routes Skill

## Principles

- **One SetupRoutes function**: Takes Echo instance, config, and all handlers. Wires everything.
- **Middleware per group**: Apply auth middleware to route groups, not individual routes.
- **Path convention**: `/api/v1/...` for versioned APIs, `/internal/...` for S2S, `/partner/...` for user-facing.
- **Group naming**: Use descriptive group variables (e.g., `usr`, `role`, `team`, `s2s`).
- **Route order**: Public → S2S → User JWT → Admin/Role-protected.

## File Structure

```
internal/routes/
└── routes.go    ← Single file, SetupRoutes function
```

## SetupRoutes Function

```go
package routes

import (
    "github.com/labstack/echo/v4"

    "myapp/internal/config"
    "myapp/internal/handler"
    "myapp/internal/middleware"
)

func SetupRoutes(
    e *echo.Echo,
    cfg *config.Config,
    healthHandler *handler.HealthHandler,
    notifHandler *handler.NotificationHandler,
    prefHandler *handler.PreferenceHandler,
) {
    // Setup global middleware first
    middleware.SetupMiddleware(e, cfg.Security.JWTSecret, cfg.Security.EnableCORS, cfg.Security.AllowedOrigins)

    // API version group
    v1 := e.Group("/api/v1")

    // ── Public endpoints (no auth) ────────────────────────────
    v1.GET("/health", healthHandler.GetHealth)

    // ── S2S endpoints (x-api-key auth) ────────────────────────
    s2s := e.Group("/internal", middleware.S2SAuthMiddleware(cfg.Security.S2SAPIKey))
    s2s.POST("/notifications/send-manifest", notifHandler.SendManifest)
    s2s.POST("/notifications/send-borongan", notifHandler.SendBorongan)

    // ── User endpoints (JWT auth) ─────────────────────────────
    partner := e.Group("/partner", middleware.AllLoginUserMiddleware)

    // Notifications
    partner.GET("/notifications", notifHandler.GetNotifications)
    partner.GET("/notifications/count", notifHandler.CountUnread)
    partner.PUT("/notifications/:id/read", notifHandler.MarkRead)

    // Preferences
    partner.GET("/notifications/preferences", prefHandler.GetPreferences)
    partner.PUT("/notifications/preferences", prefHandler.UpdatePreferences)
}
```

## Route Group Patterns

### S2S Group (Internal API)

All endpoints under `/internal/` require `x-api-key` header:

```go
s2s := e.Group("/internal", middleware.S2SAuthMiddleware(cfg.Security.S2SAPIKey))
s2s.POST("/notifications/send-manifest", notifHandler.SendManifest)
s2s.POST("/notifications/send-borongan", notifHandler.SendBorongan)
```

### User Group (Partner API)

All endpoints under `/partner/` require `accesstoken` JWT:

```go
partner := e.Group("/partner", middleware.AllLoginUserMiddleware)
partner.GET("/notifications", notifHandler.GetNotifications)
partner.GET("/notifications/count", notifHandler.CountUnread)
partner.PUT("/notifications/:id/read", notifHandler.MarkRead)
```

### Role-Protected Group

Sub-group requiring specific roles:

```go
admin := e.Group("/partner/admin", middleware.AllLoginUserMiddleware, middleware.RoleMiddleware([]string{"super_admin", "verificator"}))
admin.GET("/notifications", notifHandler.GetAdminNotifications)
```

### Optional Auth Group

Endpoints that work with or without JWT:

```go
public := e.Group("/public", middleware.OptionalLoginUserMiddleware)
public.GET("/notifications/count", notifHandler.CountUnread)
```

## Path Convention

| Pattern | Auth | Description |
|---------|------|-------------|
| `/api/v1/health` | None | Health check (public) |
| `/internal/{entity}/...` | S2S (x-api-key) | Service-to-service endpoints |
| `/partner/{entity}/...` | JWT (accesstoken) | User-facing partner endpoints |
| `/partner/admin/...` | JWT + Role | Admin-only endpoints |

## Scope-Specific Routes

### jr-web-partner

```
S2S:
  POST /internal/notifications/send-manifest
  POST /internal/notifications/send-borongan

User:
  GET  /partner/notifications
  GET  /partner/notifications/count
  PUT  /partner/notifications/:id/read
  GET  /partner/notifications/preferences
  PUT  /partner/notifications/preferences
```

### jr-external

```
(Check external code migration doc for specific routes)
```

## Checklist

- [ ] `SetupRoutes` function takes Echo + config + all handlers
- [ ] Global middleware applied via `middleware.SetupMiddleware()`
- [ ] S2S group uses `S2SAuthMiddleware(cfg.Security.S2SAPIKey)`
- [ ] User group uses `AllLoginUserMiddleware`
- [ ] Role-protected sub-groups use `RoleMiddleware`
- [ ] Path params use `:id` convention (Echo standard)
- [ ] Handler methods match the service interface
- [ ] All routes from code migration doc are registered
- [ ] No business logic in routes — only middleware + handler binding