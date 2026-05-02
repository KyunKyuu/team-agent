---
name: go-entity
description: |
  Write GORM entity structs for Go microservices in the jr-external/jr-web-partner rebuild project.
  Use this skill when creating or modifying entity/model Go files in internal/entity/.
  Covers GORM tags, TableName(), helper methods, app_origin discriminator, backward-compatible JSON, and entity splitting for large files.
  Trigger when: writing entity.go, model.go, or any GORM struct; when user says "create entity", "write model", "define struct for table"; when implementing a repository that needs an entity first.
---

# Go Entity Skill

## Principles

- **Keep it simple**: Entity files hold GORM structs + TableName + thin helpers. No business logic, no database queries, no service calls.
- **One struct per file** when the entity is large (>200 lines with helpers). Small related structs (like filter params) can share a file.
- **Max 300-500 lines per file**. If helpers grow beyond that, split into `{entity}_helpers.go`.
- **Consume interfaces where they're used**: Entity doesn't consume repository or service interfaces. It's pure data.
- **Go idioms**: Prefer `errors.New` for sentinel errors, `fmt.Errorf` for wrapped errors, no panic in libraries.

## GORM Tags

Every field MUST have `column`, `type`, and appropriate constraints. No bare structs.

```go
type Notification struct {
    // Primary key: UUID with database default
    ID string `gorm:"column:id;type:uuid;primaryKey;default:gen_random_uuid()" json:"id"`

    // Required string fields: type + not null
    UserID    string  `gorm:"column:user_id;type:varchar(255);not null;index" json:"user_id"`
    EventType string  `gorm:"column:event_type;type:varchar(100);not null" json:"type"`

    // Nullable string fields: pointer type
    Title    *string `gorm:"column:title;type:varchar(255)" json:"title,omitempty"`
    Body     *string `gorm:"column:body;type:varchar(1000)" json:"body,omitempty"`
    CompanyID *string `gorm:"column:company_id;type:varchar(255)" json:"templated_title,omitempty"`

    // Boolean with default
    IsRead bool `gorm:"column:is_read;type:boolean;not null;default:false" json:"is_read"`

    // Discriminator: always include for unified tables
    AppOrigin string `gorm:"column:app_origin;type:varchar(20);not null;default:'web_partner'" json:"-"`

    // Timestamps
    CreatedAt time.Time  `gorm:"column:created_at;type:timestamptz;not null;default:now()" json:"created_at"`
    UpdatedAt *time.Time `gorm:"column:updated_at;type:timestamptz" json:"updated_at,omitempty"`

    // JSONB data
    Data datatypes.JSON `gorm:"column:data;type:jsonb" json:"data,omitempty"`

    // Computed fields: NOT stored in DB
    FullName string `gorm:"-" json:"full_name,omitempty"`
}
```

**Tag rules:**
- `column` — always explicit, snake_case matching DB column name
- `type` — always explicit, use PostgreSQL types: `uuid`, `varchar(N)`, `text`, `boolean`, `timestamptz`, `jsonb`, `integer`, `bigint`
- `not null` — for required fields
- `default` — for database-side defaults
- `index` — for frequently queried fields
- `uniqueIndex` — for unique constraints
- `json` — legacy format for backward compatibility (see mapping rules below)
- `gorm:"-"` — for computed fields not stored in DB

## TableName()

Always explicit. No magic.

```go
func (Notification) TableName() string {
    return "notifications"
}
```

## Helper Methods

Keep helpers thin. They map between internal and legacy formats, or compute derived values. No DB access, no side effects.

```go
// GetLegacyType maps internal event_type to legacy Bahasa Indonesia format.
// Internal: "transaction_created" → Legacy: "pembayaran_dibuat"
func (n *Notification) GetLegacyType() string {
    return EventTypeToLegacy(n.EventType)
}

// IsRead reports whether the notification has been read.
func (n *Notification) IsRead() bool {
    return n.Status == "read"
}
```

If the mapping table is large (>10 entries), put it in a separate file:

```go
// internal/entity/event_type_mapping.go
package entity

// EventTypeMapping maps internal snake_case English to legacy locale format.
var EventTypeMapping = map[string]string{
    "transaction_created":              "pembayaran_dibuat",
    "transaction_success":             "pembayaran_diterima",
    "transaction_canceled":           "pembayaran_dibatalkan",
    // ...
}

// EventTypeToLegacy converts internal event type to legacy format.
func EventTypeToLegacy(eventType string) string {
    if legacy, ok := EventTypeMapping[eventType]; ok {
        return legacy
    }
    return eventType
}

// LegacyToEventType converts legacy format to internal event type.
func LegacyToEventType(legacy string) string {
    for internal, l := range EventTypeMapping {
        if l == legacy {
            return internal
        }
    }
    return legacy
}
```

## app_origin Discriminator

For **unified tables** (shared with jr-external), every entity MUST have:

```go
AppOrigin string `gorm:"column:app_origin;type:varchar(20);not null;default:'web_partner'" json:"-"`
```

The `json:"-"` tag means it's hidden from API responses. The default value matches the scope:
- `jr-web-partner` → `default:'web_partner'`
- `jr-external` → `default:'external'`

Every repository query on a unified table MUST filter by `app_origin`. This is non-negotiable.

## Nullable Fields

Use Go pointers for nullable DB columns. This distinguishes "not set" from "zero value":

```go
// Nullable string
Title *string `gorm:"column:title;type:varchar(255)" json:"title,omitempty"`

// Nullable timestamp
ReadAt *time.Time `gorm:"column:read_at;type:timestamptz" json:"read_at,omitempty"`

// Required boolean with default
IsRead bool `gorm:"column:is_read;type:boolean;not null;default:false" json:"is_read"`
```

## JSON Tags for Backward Compatibility

Response JSON tags MUST match the legacy monolith format. This is critical — frontend/mobile clients depend on these exact field names.

```go
// GOOD: legacy field names preserved
type NotificationResponse struct {
    TipeNotif    string `json:"tipe_notif"`      // NOT "notification_type"
    Judul        string `json:"judul"`            // NOT "title"
    Dibaca       bool   `json:"dibaca"`           // NOT "is_read"
    TanggalBuat  string `json:"tanggal_dibuat"`  // NOT "created_at"
}

// BAD: using internal English names
type NotificationResponse struct {
    NotificationType string `json:"notification_type"` // ❌ breaks frontend
    Title           string `json:"title"`            // ❌ breaks frontend
    IsRead          bool   `json:"is_read"`          // ❌ breaks frontend
}
```

## File Splitting

When an entity file grows beyond 300 lines, split it:

```
internal/entity/
├── notification.go              ← GORM struct + TableName + small helpers
├── notification_helpers.go      ← GetLegacyType, ToLegacyResponse, IsRead
├── event_type_mapping.go        ← Mapping tables (if shared across entities)
├── device_token.go              ← Separate entity
└── preference.go                ← Separate entity
```

## Checklist

Before finishing an entity file, verify:
- [ ] Every field has `column`, `type`, and constraint GORM tags
- [ ] `TableName()` is explicit
- [ ] Nullable fields use Go pointers
- [ ] `app_origin` discriminator present for unified tables
- [ ] JSON tags match legacy format for response structs
- [ ] Helper methods are thin (no DB access, no side effects)
- [ ] File is under 300-500 lines (split if needed)
- [ ] Mapping tables are in separate file if >10 entries