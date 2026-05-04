# Database Migration — {SOURCE_A_NAME}

<!-- Schema dan mapping dari source A (legacy) ke struktur baru.
     Rename folder ini sesuai nama source: contoh "01-jr-web-partner" atau "01-monolith" -->

---

## Entity List

<!-- Daftar entity dari source ini yang masuk scope epic.
     Contoh:
     - OTP: tabel otp_codes
     - OTPLog: tabel otp_logs -->

---

## Entity: {EntityName}

<!-- Satu section per entity. Duplikasi section ini untuk tiap entity. -->

### Legacy Schema

<!-- Struktur tabel di legacy (nama kolom, tipe data, constraint).
     Contoh:
     ```sql
     CREATE TABLE otp_codes (
         id         INT AUTO_INCREMENT,
         phone      VARCHAR(20),
         code       VARCHAR(6),
         expired_at DATETIME,
         created_at DATETIME
     );
     ``` -->

### Column Mapping

<!-- Mapping kolom legacy → kolom baru.
     | Legacy Column | New Column   | Type        | Notes                    |
     |---------------|--------------|-------------|--------------------------|
     | id            | id           | BIGSERIAL   |                          |
     | phone         | phone_number | VARCHAR(20) | rename                   |
     | expired_at    | expired_at   | TIMESTAMPTZ | convert to UTC           | -->

### GORM Struct (target)

<!-- Struct Go yang akan dibuat di service baru.
     ```go
     type OTP struct {
         ID        uint      `gorm:"primaryKey"`
         Phone     string    `gorm:"column:phone_number"`
         ExpiredAt time.Time `gorm:"column:expired_at"`
     }
     ``` -->
