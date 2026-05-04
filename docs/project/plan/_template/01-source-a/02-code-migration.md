# Code Migration — {SOURCE_A_NAME}

<!-- Scope migrasi kode dari source A. Ini yang dibaca doc-reader untuk extract API contract.
     Rename folder ini sesuai nama source: contoh "01-jr-web-partner" atau "01-monolith" -->

---

## Service Name

<!-- Nama service baru yang akan menghandle scope ini.
     Contoh: otp-general -->

---

## Entities In Scope

<!-- Entity yang masuk scope epic ini (bukan semua entity di service).
     Contoh:
     - OTP
     - OTPBlacklist -->

---

## API Routes

<!-- Semua endpoint yang masuk scope, dari legacy source ini.
     | Method | Path              | Auth     | Description              |
     |--------|-------------------|----------|--------------------------|
     | POST   | /otp/send         | JWT      | Kirim OTP ke nomor phone |
     | POST   | /otp/verify       | JWT      | Verifikasi OTP           |
     | POST   | /otp/resend       | JWT      | Kirim ulang OTP          | -->

---

## DTO Definitions

<!-- Request dan response schema per endpoint.
     Nama field HARUS sama persis dengan legacy — ini yang dipakai untuk backward compat.

     ### POST /otp/send
     Request:
     ```json
     {
       "phone": "08123456789",
       "type": "LOGIN"
     }
     ```
     Response:
     ```json
     {
       "code": "123456",
       "expired_at": "2024-01-01T10:05:00Z"
     }
     ``` -->

---

## Business Logic Notes

<!-- Logika bisnis penting yang tidak obvious dari schema — baca dari legacy code.
     Contoh:
     - OTP di-generate 6 digit numerik, random
     - Sebelum send, invalidate semua OTP aktif untuk nomor yang sama
     - Rate limit dicek sebelum generate (3 request per 10 menit) -->

---

## Auth & Middleware

<!-- Chain middleware untuk service ini.
     Contoh:
     - JWT middleware: extract user_id + app_origin dari token
     - Rate limit middleware: per phone number -->
