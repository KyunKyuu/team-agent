---
name: go-middleware
description: |
  Write Echo middleware implementations for Go microservices in the jr-external/jr-web-partner rebuild project.
  Use this skill when creating or modifying middleware files in internal/middleware/.
  Covers JWT auth (accesstoken header), S2S auth (x-api-key header), role-based access, team-based access, CORS, security headers, and context helpers.
  Trigger when: writing auth middleware, creating JWT validation, implementing S2S API key check, adding role middleware, setting up CORS, creating security middleware.
---

# Go Middleware Skill

## Principles

- **Two auth patterns**: JWT for user endpoints, S2S API key for internal endpoints.
- **Header convention**: `accesstoken` (lowercase) for JWT, `x-api-key` (lowercase) for S2S.
- **Context storage**: Set user/team info in Echo context via `c.Set()`, retrieve via helpers.
- **Package-level secret**: JWT secret set during `SetupMiddleware`, accessible to all middleware funcs.
- **Global middleware**: Applied in `SetupMiddleware()` — RequestID, Logger, Recover, CORS, Secure, BodyLimit, Gzip.
- **Per-route middleware**: Applied in routes.go — `AllLoginUserMiddleware`, `RoleMiddleware`, `S2SAuthMiddleware`.

## File Structure

```
internal/middleware/
├── middleware.go       ← SetupMiddleware + JWT helpers + auth middleware + context helpers
├── s2s.go             ← S2S API key middleware (if separate file preferred)
└── cors.go            ← CORS configuration (if complex, otherwise in middleware.go)
```

## Global Middleware Setup

```go
package middleware

import (
    "github.com/labstack/echo/v4"
    "github.com/labstack/echo/v4/middleware"
)

var jwtSecretKey string

func SetupMiddleware(e *echo.Echo, jwtSecret string, enableCORS bool, allowedOrigins []string) {
    jwtSecretKey = jwtSecret

    // Built-in Echo middleware (order matters)
    e.Use(middleware.RequestID())
    e.Use(middleware.Logger())
    e.Use(middleware.Recover())

    if enableCORS {
        e.Use(middleware.CORSWithConfig(middleware.CORSConfig{
            AllowOrigins:     allowedOrigins,
            AllowMethods:     []string{echo.GET, echo.POST, echo.PUT, echo.DELETE, echo.PATCH, echo.OPTIONS},
            AllowHeaders:     []string{echo.HeaderOrigin, echo.HeaderContentType, echo.HeaderAccept, echo.HeaderAuthorization, "accesstoken", "x-api-key"},
            AllowCredentials: true,
            MaxAge:           86400,
        }))
    }

    e.Use(middleware.SecureWithConfig(middleware.SecureConfig{
        XSSProtection:      "1; mode=block",
        ContentTypeNosniff: "nosniff",
        XFrameOptions:      "DENY",
        HSTSMaxAge:         31536000,
    }))

    e.Use(middleware.BodyLimit("10MB"))
    e.Use(middleware.Gzip())
}
```

## JWT Auth Middleware

### Token Types

```go
type UserMiddleware struct {
    UserID string
    Name   string
    Role   []string
    Phone  string
    Email  string
}

type TeamMiddleware struct {
    TeamID string   `json:"tid"`
    Roles  []string `json:"roles"`
}
```

### Required Auth (AllLoginUserMiddleware)

Returns 403 if no valid JWT. Used for endpoints requiring authentication.

```go
func AllLoginUserMiddleware(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        token := GetToken(c)
        if token == nil {
            return c.String(403, "Unauthorized")
        }

        u, t, err := ValidateToken(jwtSecretKey, *token)
        if err != nil {
            return c.String(403, err.Error())
        }

        c.Set("user", u)
        c.Set("team", t)
        return next(c)
    }
}
```

### Optional Auth (OptionalLoginUserMiddleware)

Proceeds even without JWT. Sets user/team in context if token is valid.

```go
func OptionalLoginUserMiddleware(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        token := GetToken(c)
        if token == nil {
            return next(c)
        }

        u, t, err := ValidateToken(jwtSecretKey, *token)
        if err != nil {
            return c.String(403, err.Error())
        }

        c.Set("user", u)
        c.Set("team", t)
        return next(c)
    }
}
```

## S2S Auth Middleware

Validates `x-api-key` header against configured API key.

```go
// S2SAuthMiddleware validates x-api-key header for internal endpoints.
func S2SAuthMiddleware(validAPIKey string) echo.MiddlewareFunc {
    return func(next echo.HandlerFunc) echo.HandlerFunc {
        return func(c echo.Context) error {
            apiKey := c.Request().Header.Get("x-api-key")
            if apiKey == "" {
                return c.String(403, "Missing API key")
            }
            if apiKey != validAPIKey {
                return c.String(403, "Invalid API key")
            }
            return next(c)
        }
    }
}
```

## Role-Based Middleware

Checks if the authenticated user has one of the required roles.

```go
func RoleMiddleware(roles []string) echo.MiddlewareFunc {
    return func(next echo.HandlerFunc) echo.HandlerFunc {
        return func(c echo.Context) error {
            user := c.Get("user").(*UserMiddleware)
            for _, name := range roles {
                if contains(user.Role, name) {
                    return next(c)
                }
            }
            return c.String(403, "Insufficient permissions")
        }
    }
}

func contains(slice []string, item string) bool {
    for _, s := range slice {
        if s == item {
            return true
        }
    }
    return false
}
```

## Team-Based Middleware

Checks team membership roles for team-specific operations.

```go
func TeamMiddlewareHandler(expectedRoles []string) echo.MiddlewareFunc {
    return func(next echo.HandlerFunc) echo.HandlerFunc {
        return func(c echo.Context) error {
            team := c.Get("team").(*TeamMiddleware)
            for _, expected := range expectedRoles {
                for _, current := range team.Roles {
                    if expected == current {
                        return next(c)
                    }
                }
            }
            return c.String(403, "Insufficient team permissions")
        }
    }
}
```

## JWT Token Helpers

```go
import (
    "errors"
    "fmt"
    "strings"
    "time"

    "github.com/golang-jwt/jwt/v5"
    "github.com/labstack/echo/v4"
)

func GetToken(c echo.Context) *string {
    // Check accesstoken header first (project convention)
    if token := c.Request().Header.Get("accesstoken"); token != "" {
        return &token
    }
    // Fallback: Authorization Bearer
    header := c.Request().Header.Get("Authorization")
    if len(header) > 0 {
        withoutBearer := strings.ReplaceAll(header, "Bearer ", "")
        return &withoutBearer
    }
    return nil
}

func CreateLoginToken(secret string, user UserMiddleware, team TeamMiddleware, roles []string) (*string, error) {
    now := time.Now().UTC()
    claims := jwt.MapClaims{
        "sub":    user.UserID,
        "userid": user.UserID,
        "name":   user.Name,
        "phone":  user.Phone,
        "email":  user.Email,
        "iss":    "jrku_auth",
        "roles":  roles,
        "iat":    now.Unix(),
        "exp":    now.Add(24 * time.Hour).Unix(),
    }

    t, _ := StructToMap(team)
    claims["team"] = t

    at := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    token, err := at.SignedString([]byte(secret))
    if err != nil {
        return nil, err
    }
    return &token, nil
}

func ValidateToken(secret string, tokenStr string) (*UserMiddleware, *TeamMiddleware, error) {
    claims := jwt.MapClaims{}
    t, err := jwt.ParseWithClaims(tokenStr, claims, func(token *jwt.Token) (interface{}, error) {
        return []byte(secret), nil
    })
    if err != nil {
        return nil, nil, err
    }
    if !t.Valid {
        return nil, nil, errors.New("invalid token")
    }

    return getUserFromToken(claims), getTeamFromToken(claims), nil
}
```

## Context Helpers

```go
func GetUser(c echo.Context) *UserMiddleware {
    if user := c.Get("user"); user != nil {
        return user.(*UserMiddleware)
    }
    return nil
}

func GetTeam(c echo.Context) *TeamMiddleware {
    if team := c.Get("team"); team != nil {
        return team.(*TeamMiddleware)
    }
    return nil
}
```

## Checklist

- [ ] JWT secret set via `SetupMiddleware` (package-level var)
- [ ] `accesstoken` header checked first, then `Authorization: Bearer`
- [ ] `x-api-key` header for S2S auth
- [ ] `AllLoginUserMiddleware` for required JWT endpoints
- [ ] `OptionalLoginUserMiddleware` for optional JWT endpoints
- [ ] `RoleMiddleware` for role-based access
- [ ] `TeamMiddlewareHandler` for team-based access
- [ ] `GetUser`/`GetTeam` context helpers
- [ ] Global middleware: RequestID, Logger, Recover, CORS, Secure, BodyLimit, Gzip
- [ ] CORS includes custom headers: `accesstoken`, `x-api-key`