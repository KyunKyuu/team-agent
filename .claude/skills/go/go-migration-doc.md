---
name: go-migration-doc
description: |
  Write GORM AutoMigrate reference and SQL migration files for Go microservices in the jr-external/jr-web-partner rebuild project.
  Use this skill when creating migration documentation or SQL files for database schema changes.
  Covers two migration approaches (GORM reference doc + SQL files), table creation, index creation, app_origin defaults, and data migration SQL.
  Trigger when: writing SQL migration, creating GORM AutoMigrate reference, adding database indexes, writing data migration scripts.
---

# Go Migration Doc Skill

## Principles

- **Two migration approaches**: GORM AutoMigrate reference (documentation only) + SQL files (manual execution).
- **No AutoMigrate in main.go**: GORM AutoMigrate is for reference only. Real migrations are SQL files.
- **SQL files are the source of truth**: They're versioned, reviewable, and executed manually by DBAs.
- **app_origin defaults**: Unified tables get `DEFAULT 'web_partner'` or `DEFAULT 'external'`.
- **Indexes for frequent queries**: Every field used in WHERE clauses should have an index.
- **Numbered SQL files**: `00001_create_notifications_table.sql`, `00002_create_indexes.sql`, etc.

## File Structure

```
├── docs/
│   └── gorm_automigrate_reference.go    ← GORM reference (NOT executed)
└── migrations/
    ├── 00001_create_notifications_table.sql
    ├── 00002_create_device_tokens_table.sql
    ├── 00003_create_preferences_table.sql
    └── 00004_create_indexes.sql
```

## GORM AutoMigrate Reference (Documentation Only)

This file is for REFERENCE only — it documents what the GORM structs produce. It is NEVER executed in main.go.

```go
// docs/gorm_automigrate_reference.go
// THIS FILE IS FOR REFERENCE ONLY — DO NOT EXECUTE
// Actual migrations are in migrations/ directory as SQL files

package docs

import (
    "myapp/internal/entity"
    "gorm.io/gorm"
)

// AutoMigrateReference shows what GORM AutoMigrate would produce.
// Compare with SQL files in migrations/ to verify schema correctness.
func AutoMigrateReference(db *gorm.DB) error {
    return db.AutoMigrate(
        &entity.Notification{},
        &entity.DeviceToken{},
        &entity.PartnerNotificationPreference{},
    )
}
```

## SQL Migration Files

### Table Creation

```sql
-- migrations/00001_create_notifications_table.sql
-- Notification table for jr-web-partner notification service
-- Unified table: shared with jr-external (differentiated by app_origin)

CREATE TABLE IF NOT EXISTS notifications (
    id           UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id      VARCHAR(255) NOT NULL,
    event_type   VARCHAR(100) NOT NULL,
    title        VARCHAR(255),
    body         VARCHAR(1000),
    company_id   VARCHAR(255),
    data_id      VARCHAR(255),
    icon_type    VARCHAR(50),
    status       VARCHAR(20)  NOT NULL DEFAULT 'unread',
    is_read      BOOLEAN      NOT NULL DEFAULT false,
    read_at      TIMESTAMPTZ,
    created_by   VARCHAR(255),
    data         JSONB,
    app_origin   VARCHAR(20)  NOT NULL DEFAULT 'web_partner',
    created_at   TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ
);

-- Comment for documentation
COMMENT ON TABLE notifications IS 'Unified notification table shared by jr-web-partner and jr-external';
COMMENT ON COLUMN notifications.app_origin IS 'Discriminator: web_partner or external';
```

```sql
-- migrations/00002_create_device_tokens_table.sql
-- Device token table for FCM push notifications

CREATE TABLE IF NOT EXISTS device_tokens (
    id         UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id    VARCHAR(255) NOT NULL,
    token      VARCHAR(500) NOT NULL,
    platform   VARCHAR(20),
    app_origin VARCHAR(20)  NOT NULL DEFAULT 'web_partner',
    created_at TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ,

    UNIQUE(user_id, app_origin)
);

COMMENT ON TABLE device_tokens IS 'FCM device tokens for push notifications';
```

```sql
-- migrations/00003_create_preferences_table.sql
-- Partner notification preferences

CREATE TABLE IF NOT EXISTS partner_notification_preferences (
    id                              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id                         VARCHAR(255) NOT NULL UNIQUE,
    invoice_created_enabled         BOOLEAN     NOT NULL DEFAULT true,
    invoice_payment_success_enabled  BOOLEAN     NOT NULL DEFAULT true,
    laporan_received_enabled        BOOLEAN     NOT NULL DEFAULT true,
    laporan_processed_enabled       BOOLEAN     NOT NULL DEFAULT true,
    verification_pending_enabled    BOOLEAN     NOT NULL DEFAULT true,
    verification_completed_enabled  BOOLEAN     NOT NULL DEFAULT true,
    invoice_canceled_enabled        BOOLEAN     DEFAULT true,
    email_enabled                   BOOLEAN     DEFAULT true,
    in_app_enabled                  BOOLEAN     DEFAULT true,
    push_enabled                    BOOLEAN     DEFAULT true,
    app_origin                      VARCHAR(20) NOT NULL DEFAULT 'web_partner',
    created_at                      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at                      TIMESTAMPTZ
);

COMMENT ON TABLE partner_notification_preferences IS 'Notification preference settings per user';
```

### Index Creation

```sql
-- migrations/00004_create_indexes.sql
-- Indexes for frequently queried fields

-- Notifications: user_id + app_origin (most common query pattern)
CREATE INDEX IF NOT EXISTS idx_notifications_user_app_origin
    ON notifications (user_id, app_origin);

-- Notifications: unread count query
CREATE INDEX IF NOT EXISTS idx_notifications_user_unread
    ON notifications (user_id, is_read, app_origin)
    WHERE is_read = false;

-- Notifications: created_at for ordering
CREATE INDEX IF NOT EXISTS idx_notifications_created_at
    ON notifications (created_at DESC);

-- Notifications: event_type for filtering
CREATE INDEX IF NOT EXISTS idx_notifications_event_type
    ON notifications (event_type, app_origin);

-- Device tokens: token lookup for deletion
CREATE INDEX IF NOT EXISTS idx_device_tokens_token
    ON device_tokens (token, app_origin);

-- Preferences: user lookup (already UNIQUE but explicit index)
CREATE INDEX IF NOT EXISTS idx_preferences_user_id
    ON partner_notification_preferences (user_id, app_origin);
```

## Data Migration SQL (If Needed)

For migrating data from legacy monolith tables:

```sql
-- migrations/00005_wp_data_migration.sql
-- Migrate data from legacy webpartner monolith to new schema

-- Step 1: Migrate notifications
INSERT INTO notifications (user_id, event_type, title, body, company_id, status, is_read, read_at, created_by, app_origin, created_at)
SELECT
    user_id,
    legacy_type_to_event(type),        -- map legacy type to internal event_type
    title,
    body,
    company_id,
    CASE WHEN is_read = 1 THEN 'read' ELSE 'unread' END,
    is_read = 1,
    read_at,
    created_by,
    'web_partner',
    created_at
FROM legacy_wp.notifications
WHERE app_name = 'web_partner';

-- Step 2: Migrate preferences
INSERT INTO partner_notification_preferences (user_id, invoice_created_enabled, invoice_payment_success_enabled, laporan_received_enabled, app_origin)
SELECT
    user_id,
    menunggu_pembayaran_notif,
    pembayaran_selesai_notif,
    laporan_diterima_notif,
    'web_partner'
FROM legacy_wp.user_preferences
WHERE app_name = 'web_partner';
```

## Scope-Specific Defaults

| Scope | app_origin default | Queue name |
|-------|-------------------|------------|
| jr-web-partner | `web_partner` | `wp_service_notification` |
| jr-external | `external` | `ext_service_notification` |

## Checklist

- [ ] GORM AutoMigrate reference file in `docs/` (NOT in main.go)
- [ ] SQL migration files numbered sequentially
- [ ] Every table has `app_origin` with correct default
- [ ] `app_origin` default matches scope (`web_partner` or `external`)
- [ ] Indexes for all frequently queried fields
- [ ] Partial index for `is_read = false` (unread query optimization)
- [ ] `COMMENT ON TABLE` and `COMMENT ON COLUMN` for documentation
- [ ] UUID primary keys with `gen_random_uuid()`
- [ ] `TIMESTAMPTZ` for all timestamps (not `TIMESTAMP`)
- [ ] Data migration SQL in separate numbered file