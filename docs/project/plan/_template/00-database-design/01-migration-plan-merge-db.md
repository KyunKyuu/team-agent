# Database Migration Plan — Merge DB

<!-- Dibuat oleh Tech Lead / Architect berdasarkan schema legacy WP dan EXT.
     Dokumen ini mendefinisikan bagaimana tabel dari dua source digabungkan. -->

---

## Overview

<!-- Ringkasan strategi merge database.
     Contoh: tabel otp_codes digabung menjadi satu tabel unified dengan kolom app_origin
             sebagai partisi. Tabel otp_blacklist tetap terpisah per scope. -->

---

## Entity List

<!-- Daftar semua entity yang terlibat dalam epic ini.
     Format:
     | Entity       | Table Name      | Strategy              | app_origin default |
     |------------- |-----------------|-----------------------|--------------------|
     | OTP          | otp_codes       | Unified (shared)      | —                  |
     | OTPBlacklist | otp_blacklists  | Separate per scope    | —                  | -->

---

## Unified Tables

<!-- Tabel yang digabung menjadi satu, dipartisi oleh kolom app_origin.
     Setiap query WAJIB include WHERE app_origin = ?

     Contoh:
     ### otp_codes
     Gabungan dari: wp.otp_codes + ext.otp_tokens
     Kolom baru: app_origin VARCHAR(20) NOT NULL DEFAULT ''
     app_origin value: 'web_partner' (dari WP) | 'external' (dari EXT) -->

---

## Separate Tables

<!-- Tabel yang tetap terpisah per scope (tidak digabung).
     Contoh:
     ### otp_rate_limits (WP only)
     Tidak ada equivalent di EXT — tetap ada, hanya diakses oleh WP scope. -->

---

## Schema DDL

<!-- SQL DDL untuk tabel-tabel baru / yang diubah.
     Contoh:
     ```sql
     CREATE TABLE otp_codes (
         id          BIGSERIAL PRIMARY KEY,
         app_origin  VARCHAR(20) NOT NULL DEFAULT '',
         ...
     );
     ``` -->
