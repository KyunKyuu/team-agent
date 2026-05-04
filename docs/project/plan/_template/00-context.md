# Context — {SERVICE_NAME}

<!-- Ditulis oleh Tech team. Ini patokan tertinggi — override migration docs jika ada konflik. -->
<!-- Semua section opsional, tapi makin lengkap makin baik hasilnya. -->

---

## Situasi

<!-- Jelaskan kondisi saat ini: kenapa service ini perlu dibangun / direbuild / di-extend.
     Contoh: legacy pakai Node.js 8 EOL + MongoDB, perlu migrasi ke Go + PostgreSQL.
             Dua service terpisah (WP dan EXT) harus digabung karena duplikasi logic. -->

---

## Yang Dibangun

<!-- Target state setelah selesai: service ini mengerjakan apa.
     Contoh: unified OTP service yang handle send, verify, resend untuk kedua scope (WP + EXT).
             Satu service, dua channel: REST API + RabbitMQ RPC. -->

---

## Constraints Khusus

<!-- Rules spesifik untuk service ini — yang tidak ada di template generik.
     Contoh:
     - Rate limit: max 3 OTP request per nomor per 10 menit
     - OTP expire: 5 menit untuk WP, 10 menit untuk EXT (beda per scope)
     - SMS gateway: pakai Twilio untuk WP, Nexmo untuk EXT -->

---

## Integrasi

<!-- Service lain yang terlibat, tipe auth, pola komunikasi.
     Contoh:
     - Auth: JWT dari identity-service (header Authorization: Bearer ...)
     - Notifikasi: publish event ke notification-service via RabbitMQ exchange "notifications"
     - Internal API: dipanggil oleh auth-service untuk verify OTP -->

---

## Keputusan Pre-set

<!-- Keputusan arsitektur yang sudah fixed sebelum brainstorming — tidak perlu didiskusikan lagi.
     Brainstormer akan skip section yang sudah ada di sini.
     Contoh:
     - Pakai Redis untuk cache OTP, TTL = expire time OTP
     - Error format pakai BaseResponse{code, status, message, data}
     - app_origin diambil dari JWT claims, bukan dari request body -->

---

## Hal yang Perlu Diperhatikan

<!-- Edge cases, gotchas, hal yang tidak obvious dari legacy code atau epic.
     Contoh:
     - WP punya endpoint /otp/resend yang tidak ada di EXT — tetap diimplementasi untuk WP only
     - Field "expired_at" di WP adalah ISO 8601 string, di EXT adalah Unix timestamp — jangan diubah
     - Legacy WP ada bug di validasi phone: menerima format tanpa +62, harus dipertahankan (backward compat) -->
