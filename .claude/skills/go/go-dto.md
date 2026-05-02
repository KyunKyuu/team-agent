---
name: go-dto
description: |
  Write request and response DTO structs for Go microservices in the jr-external/jr-web-partner rebuild project.
  Use this skill when creating or modifying DTO files in internal/dto/.
  Covers validation (go-playground/validator), backward-compatible JSON tags, event type mapping, preference field mapping, partial updates with *bool, and DTO-to-entity mappers.
  Trigger when: writing DTO, request struct, response struct, creating mapper functions, defining API contract, implementing handler that needs request/response types.
---

# Go DTO Skill

## Principles

- **Separate request and response DTOs** — never combine them in one struct.
- **Request DTOs** validate input. **Response DTOs** match legacy format exactly.
- **Keep mappers thin** — `ToResponse()` methods or standalone mapper functions that do field-by-field assignment, no business logic.
- **Max 300-500 lines per file**. Split by entity: `notification.go`, `preference.go`.
- **Consume interfaces where they're used**: DTOs are consumed by handlers and services. They don't consume repositories.

## File Structure

```
internal/dto/
├── common.go           ← CustomValidator, shared types
├── pagination.go       ← Pagination helper (shared)
├── mapper.go           ← Entity-to-DTO mapper functions
├── notification.go      ← Notification request/response DTOs
├── device_token.go     ← DeviceToken DTOs
└── preference.go       ← Preference DTOs
```

## Request DTOs

Use `go-playground/validator` for validation. Tags: `json` for field names, `validate` for rules, `query` for query params, `param` for path params.

```go
package dto

// --- S2S Request DTOs ---

// SendNotificationManifestRequest untuk POST /internal/notifications/send-manifest
type SendNotificationManifestRequest struct {
    CreatedBy        string `json:"created_by" validate:"required"`
    NotificationType string `json:"notification_type" validate:"required,oneof=pembayaran_dibuat pembayaran_diterima"`
    TransactionCode  string `json:"transaction_code" validate:"required"`
}

// --- User Request DTOs ---

// GetNotificationsRequest untuk GET /partner/notifications
type GetNotificationsRequest struct {
    Page      int    `query:"page" validate:"min=1"`
    Limit     int    `query:"limit" validate:"min=1,max=100"`
    CompanyID string `query:"company_id"`
}

// MarkNotificationReadRequest untuk PUT /partner/notifications/:id/read
type MarkNotificationReadRequest struct {
    NotificationID string `param:"id" validate:"required,uuid"`
}

// --- Preference Request DTOs ---

// UpdatePreferenceRequest untuk PUT /partner/notifications/preferences
// Use *bool for partial updates — only fields explicitly sent are updated
type UpdatePreferenceRequest struct {
    MenungguPembayaranNotif       *bool `json:"menunggu_pembayaran_notif"`
    PembayaranSelesaiNotif        *bool `json:"pembayaran_selesai_notif"`
    LaporanDiterimaNotif          *bool `json:"laporan_diterima_notif"`
    InvoiceCanceledEnabled        *bool `json:"invoice_canceled_enabled,omitempty"`
    EmailEnabled                  *bool `json:"email_enabled,omitempty"`
}
```

**Key rules for request DTOs:**
- `json` tags use **internal/legacy field names** that the monolith API expects
- `validate` tags use `go-playground/validator` syntax
- `query` tags for GET parameters, `param` for path parameters
- Use `*bool` for partial updates — `nil` means "don't change", `false` means "set to false"
- Use `omitempty` for optional fields only

## Response DTOs

Response JSON tags MUST match the legacy monolith format. This is non-negotiable.

```go
package dto

// NotificationListResponse untuk GET /partner/notifications
type NotificationListResponse struct {
    Data  []NotificationItemResponse `json:"data"`
    Total int64                      `json:"total"`
    Page  int                        `json:"page"`
    Limit int                        `json:"limit"`
}

// NotificationItemResponse — field names MUST match monolith exactly
type NotificationItemResponse struct {
    ID             string `json:"id"`
    UserID         string `json:"user_id"`
    Type           string `json:"type"`               // event_type → legacy type
    DataID         string `json:"data_id,omitempty"`
    Title          string `json:"title,omitempty"`
    TemplatedTitle string `json:"templated_title,omitempty"` // company_id → templated_title
    Body           string `json:"body,omitempty"`
    Status         string `json:"status"`              // "read" or "unread"
    IconType       string `json:"icon_type,omitempty"`
    CreatedBy      string `json:"created_by,omitempty"`
    CreatedAt      string `json:"created_at"`
    UpdatedAt      string `json:"updated_at,omitempty"`
}

// PreferenceResponse untuk GET /partner/notifications/preferences
type PreferenceResponse struct {
    UserID                         string `json:"user_id"`
    MenungguPembayaranNotif       bool   `json:"menunggu_pembayaran_notif"`
    PembayaranSelesaiNotif        bool   `json:"pembayaran_selesai_notif"`
    LaporanDiterimaNotif          bool   `json:"laporan_diterima_notif"`
    LaporanSelesaiNotif           bool   `json:"laporan_selesai_notif"`
    TrxVerifikasiPetugasNotif    bool   `json:"trx_verifikasi_petugas_notif"`
    LaporanVerifikasiPetugasNotif bool   `json:"laporan_verifikasi_petugas_notif"`
    // New fields with omitempty (not in legacy)
    InvoiceCanceledEnabled *bool `json:"invoice_canceled_enabled,omitempty"`
    EmailEnabled           *bool `json:"email_enabled,omitempty"`
    InAppEnabled           *bool `json:"in_app_enabled,omitempty"`
}
```

**Key rules for response DTOs:**
- JSON tags use **legacy field names** (Bahasa Indonesia where the monolith uses them)
- New fields not present in legacy use `omitempty`
- `*bool` for fields that may not be present in legacy responses

## Mapper Functions

Put entity-to-DTO mapping in `mapper.go` or in the DTO file if it's small. Mappers are thin — just field assignment, no business logic.

```go
package dto

import "myapp/internal/entity"

// NotificationToResponse maps entity to legacy response format.
// This is the ONLY place where field name mapping happens.
func NotificationToResponse(n *entity.Notification) NotificationItemResponse {
    return NotificationItemResponse{
        ID:             n.ID,
        UserID:         n.UserID,
        Type:           n.GetLegacyType(),    // uses entity helper
        DataID:         derefString(n.DataID),
        Title:          derefString(n.Title),
        TemplatedTitle: derefString(n.CompanyID),
        Body:           derefString(n.Body),
        Status:         n.Status,
        IconType:       derefString(n.IconType),
        CreatedBy:      n.CreatedBy,
        CreatedAt:       n.CreatedAt.Format(time.RFC3339),
        UpdatedAt:       derefTime(n.UpdatedAt),
    }
}

// PreferenceToResponse maps preference entity to legacy response format.
func PreferenceToResponse(p *entity.PartnerNotificationPreference) PreferenceResponse {
    return PreferenceResponse{
        UserID:                         p.UserID,
        MenungguPembayaranNotif:       p.InvoiceCreatedEnabled,
        PembayaranSelesaiNotif:        p.InvoicePaymentSuccessEnabled,
        LaporanDiterimaNotif:          p.LaporanReceivedEnabled,
        LaporanSelesaiNotif:           p.LaporanProcessedEnabled,
        TrxVerifikasiPetugasNotif:    p.VerificationPendingEnabled,
        LaporanVerifikasiPetugasNotif: p.VerificationCompletedEnabled,
    }
}

// Helper: dereference pointer strings
func derefString(s *string) string {
    if s == nil {
        return ""
    }
    return *s
}

// Helper: dereference pointer timestamps
func derefTime(t *time.Time) string {
    if t == nil {
        return ""
    }
    return t.Format(time.RFC3339)
}
```

## Validation Setup (common.go)

```go
package dto

import "github.com/go-playground/validator/v10"

// CustomValidator wraps the validator for Echo
type CustomValidator struct {
    Validator *validator.Validate
}

func (cv *CustomValidator) Validate(i interface{}) error {
    return cv.Validator.Struct(i)
}
```

## Pagination (pagination.go)

```go
package dto

import "math"

type Pagination struct {
    Limit     int         `json:"limit" query:"limit"`
    Page      int         `json:"page" query:"page"`
    Sort      string      `json:"sort,omitempty" query:"sort"`
    Search    string      `json:"search,omitempty" query:"search"`
    TotalRows int64       `json:"total_rows"`
    TotalPage int         `json:"total_pages"`
    Rows      interface{} `json:"rows"`
    CompanyID string      `json:"company_id,omitempty" query:"company_id"`
}

func (p *Pagination) GetOffset() int {
    return (p.GetPage() - 1) * p.GetLimit()
}

func (p *Pagination) GetLimit() int {
    if p.Limit == 0 {
        p.Limit = 10
    }
    return p.Limit
}

func (p *Pagination) GetPage() int {
    if p.Page == 0 || p.Page < 0 {
        p.Page = 1
    }
    return p.Page
}

func (p *Pagination) Calculate(totalRows int64) {
    p.TotalRows = totalRows
    p.TotalPage = int(math.Ceil(float64(totalRows) / float64(p.GetLimit())))
}
```

## RPC Request/Response DTOs

For RabbitMQ RPC handlers, use separate DTOs in the same file or a `dto/rpc.go` file:

```go
package dto

// RPC request DTOs (matching RabbitMQ message format)
type SendNotifRPCRequest struct {
    UserID    string `json:"user_id" validate:"required"`
    NotifType string `json:"notif_type" validate:"required"`
    Title     string `json:"title"`
    Body      string `json:"body"`
    DataID    string `json:"data_id"`
}

// RPC response DTOs
type RPCResponse struct {
    Data  interface{} `json:"data"`
    Error string      `json:"error,omitempty"`
}
```

## Checklist

- [ ] Separate request and response DTOs
- [ ] Request DTOs: `validate` tags with `go-playground/validator`
- [ ] Response DTOs: JSON tags match legacy monolith exactly
- [ ] `*bool` for partial update fields
- [ ] `omitempty` for new fields not in legacy
- [ ] Mapper functions in `mapper.go` or same file if small
- [ ] No business logic in DTOs or mappers
- [ ] Pagination helper follows boilerplate pattern
- [ ] CustomValidator follows boilerplate pattern
- [ ] RPC DTOs in same package or `dto/rpc.go`
- [ ] File under 300-500 lines (split by entity if needed)