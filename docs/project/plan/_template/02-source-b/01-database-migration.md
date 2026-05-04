# Database Migration — {SOURCE_B_NAME}

<!-- Sama seperti 01-source-a/01-database-migration.md tapi untuk source B.
     Rename folder ini sesuai nama source: contoh "02-jr-external" atau "02-external-api"
     Hapus folder ini jika SOURCE_COUNT = 1 (hanya satu legacy source). -->

---

## Entity List

<!-- Daftar entity dari source B yang masuk scope epic ini. -->

---

## Entity: {EntityName}

### Legacy Schema

<!-- Struktur tabel di legacy source B. -->

### Column Mapping

<!-- Mapping kolom legacy B → kolom baru (mungkin berbeda dari source A). -->

### GORM Struct (target)

<!-- Struct Go target — biasanya sama dengan source A, kecuali ada perbedaan field. -->
