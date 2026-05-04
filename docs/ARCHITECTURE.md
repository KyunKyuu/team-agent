# Claude Agent Team — Architecture & Flow Documentation

> Untuk: Tech Lead & PM  
> Konteks: Sistem ini menggantikan proses manual "baca doc → desain → planning → implement → review → QA" menjadi pipeline otomatis berbasis AI agent team.

---

## Daftar Isi

1. [Masalah yang Diselesaikan](#1-masalah-yang-diselesaikan)
2. [Gambaran Besar](#2-gambaran-besar)
3. [Alur dari Sisi User](#3-alur-dari-sisi-user)
4. [Alur dari Sisi Sistem](#4-alur-dari-sisi-sistem)
5. [Kenapa Arsitektur Ini?](#5-kenapa-arsitektur-ini)
6. [Kenapa CLAUDE.md Singkat?](#6-kenapa-claudemd-singkat)
7. [Struktur File dan Fungsinya](#7-struktur-file-dan-fungsinya)
8. [Manfaat untuk PM](#8-manfaat-untuk-pm)
9. [Manfaat untuk Tech Lead](#9-manfaat-untuk-tech-lead)

---

## 1. Masalah yang Diselesaikan

### Sebelum sistem ini

Untuk setiap service rebuild (WP + EXT → unified Go service):

| Aktivitas | Waktu Manual | Risiko |
|-----------|-------------|--------|
| Baca migration docs WP + EXT | 2–4 jam | Missed requirement |
| Identifikasi conflict WP vs EXT | 1–2 jam | Salah mapping field → prod bug |
| Desain arsitektur + review | 2–4 jam | Inconsistent pattern antar developer |
| Tulis implementation plan | 3–5 jam | Placeholder, task kurang detail |
| Implement per entity | varies | Salah urutan, lupa app_origin filter |
| Review code | 1–2 jam per entity | Review manual, bias |
| QA backward compat | 1–3 jam | Field rename tidak ketahuan sampai FE komplain |

**Total: 10–20 jam per service sebelum QA selesai.**

### Setelah sistem ini

- Tulis `00-context.md` (penjelasan situasi, constraints khusus, keputusan pre-set)
- Taruh migration docs + Taiga stories dari PM di PLAN_PATH
- Jawab parameter di `/jr-init` atau `/jr-resume`
- Jalankan `/continue` — sistem jalan otomatis sampai QA selesai

**Total keterlibatan user aktif: < 30 menit per service.**

---

## 2. Gambaran Besar

Sistem ini adalah **pipeline AI multi-agent** yang dibagi menjadi 3 tim:

```
┌─────────────────────────────────────────────────────────────────┐
│                        LEAD SESSION (kamu)                       │
│  Mengatur flow, membaca output antar agent, dispatch subagents   │
└───────────────────────────┬─────────────────────────────────────┘
                            │ dispatch
           ┌────────────────┼────────────────┐
           ▼                ▼                ▼
   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
   │ PLANNING TEAM│ │   DEV TEAM   │ │   QA TEAM    │
   │  (sequential)│ │  (parallel)  │ │ (sequential) │
   └──────────────┘ └──────────────┘ └──────────────┘
```

Setiap "agent" adalah instance Claude yang di-spawn fresh dengan instruksi spesifik. Mereka tidak tahu tentang satu sama lain — komunikasi lewat file di disk (output dari satu agent = input untuk agent berikutnya).

---

## 3. Alur dari Sisi User

### Skenario A: Service baru (belum ada kode)

```
[Tech Lead siapkan PLAN_PATH]
  docs/project/plan/E05-otp/
    00-context.md              ← tulis situasi, constraints, keputusan pre-set
    00-database-design/        ← schema + merge strategy
    01-jr-web-partner/         ← migration docs WP
    02-jr-external/            ← migration docs EXT
    03-taiga-stories.md        ← dari PM (opsional)
        ↓
Jalankan: /jr-init
        ↓
Jawab 4 pertanyaan:
  1. Nama service → "otp-general"
  2. Path folder legacy (WP + EXT source code) → "legacy/"
  3. Path target project (boilerplate) → "services/otp-general/"
  4. Path epic plan folder → "docs/project/plan/E05-otp/"
        ↓
Sistem jalan otomatis (planning team ~15-30 menit)
        ↓
STOP — Review output:
  ✓ contract-matrix.md — apakah semua endpoint tercakup?
  ✓ design-decisions.md — apakah backward compat terjaga?
  ✓ implementation-plan.md — apakah tasks detail dan ada kode?
        ↓
Kalau ada konflik WP vs EXT (misal: nama field beda) → pilih resolusi
        ↓
Ketik: "lanjut"
        ↓
Jalankan: /continue
        ↓
Sistem jalan otomatis sampai selesai (dev + QA, tidak perlu standby)
        ↓
Selesai — lihat laporan QA di REPORT_PATH
```

### Skenario B: Tambah fitur ke service yang sudah ada

```
Jalankan: /jr-resume
        ↓
Jawab 5 pertanyaan:
  1. Nama service yang sudah ada → "otp-general"
  2. Nama fitur baru → "blacklist"
  3. Path folder legacy → "legacy/"
  4. Path project yang sudah ada → "services/otp-general/"
  5. Path epic plan folder → "docs/project/plan/E06-blacklist/"
        ↓
(sama seperti Skenario A setelah ini)
```

### Skenario C: Perubahan requirement di tengah jalan

```
Jalankan: /change
        ↓
Jelaskan apa yang berubah
        ↓
Sistem assess dampak: task mana aman vs perlu diubah
        ↓
Pilih cara penanganan (re-plan / adjust manual / batalkan)
        ↓
Lanjut otomatis dari titik yang sama
```

---

## 4. Alur dari Sisi Sistem

### 4.1 Command Flow

```
User ketik /jr-init
      │
      ▼
jr-init.md (command) — berjalan di lead session
  │  Kumpulkan: SERVICE_NAME, SUBMODULE_ROOT, BASE_PATH, PLAN_PATH
  │  Derive: EPIC, OUTPUT_PATH, WP_LEGACY_PATH, EXT_LEGACY_PATH, CONTEXT_PATH, TAIGA_STORIES
  │  Pre-read: {PLAN_PATH}00-context.md → CONTEXT (jika ada)
  │  Tulis: {OUTPUT_PATH}session/current.md ← state awal
  │
  ├─→ spawn Agent(doc-reader)
  │     Phase 0: proses CONTEXT (constraints khusus, keputusan pre-set)
  │     Baca: WP legacy code + EXT legacy code + migration docs + boilerplate
  │     Tulis: contract-matrix.md, entities.md, doc-reader-summary.md
  │     Kembali ke lead session
  │
  │  Lead baca contract-matrix.md
  │  [jika ada conflict] → tanya user → tulis conflict-resolutions.md
  │  [jika tidak ada conflict] → tulis conflict-resolutions.md (kosong)
  │
  ├─→ spawn Agent(brainstormer)
  │     Baca: CONTEXT (di-inject langsung — prioritas tertinggi)
  │     Baca: doc-reader output (di-inject via prompt, bukan baca file sendiri)
  │     Tanya clarifying questions (skip section yang ada di Keputusan Pre-set)
  │     Propose 2-3 approach per section
  │     Self-review
  │     Tulis: design-decisions.md
  │     Kembali ke lead session
  │
  ├─→ spawn Agent(planner)
  │     Baca: design-decisions.md, contract-matrix.md, entities.md (di-inject)
  │     Tulis: implementation-plan.md
  │     (setiap task: actual code, test, commit message — no placeholder)
  │     Kembali ke lead session
  │
  ├─→ spawn Agent(task-breaker)
  │     Baca: implementation-plan.md (di-inject)
  │     Tulis: tasks-superpowers.md ← routing map untuk /continue
  │     Tulis: tasks-taiga.md ← mapping ke PM stories atau generate baru
  │     Kembali ke lead session
  │
  │  Update session/current.md: STATUS=waiting-review
  │  Print: planning selesai, list output files
  │
  └─→ STOP — tunggu user review + "lanjut"
```

```
User ketik "lanjut"
      │
      ▼
jr-init.md (masih berjalan)
  │  git worktree add ... -b feat/{SERVICE_NAME}
  │  Update session/current.md: PHASE=development
  │  Print: "jalankan /continue"
  │
  └─→ selesai
```

```
User ketik /continue
      │
      ▼
continue.md — baca session/current.md → PHASE=development
  │
  │  Baca: tasks-superpowers.md → build task map
  │
  ├── PHASE 1: Shared Components (sequential)
  │   └── untuk setiap shared task:
  │       [pre-read] baca skill files sesuai layer task
  │       [pre-read] baca design-decisions.md
  │       spawn implementer (konten skill + design di-inject ke prompt)
  │       tunggu → spawn reviewer → tunggu
  │       jika NEEDS_FIX → spawn implementer lagi → re-review
  │       jika APPROVED → update session, next task
  │
  ├── PHASE 2: Entity Implementation (PARALLEL)
  │   └── Round 1:
  │       [pre-read] baca skill files untuk semua entity di round ini
  │       [pre-read] baca design-decisions.md + contract-matrix.md (sekali, pakai ulang)
  │       spawn impl-entity_A + impl-entity_B + impl-entity_C (SERENTAK)
  │         masing-masing prompt berisi: task content + skill content + design
  │       ← tunggu semua selesai
  │       spawn reviewer-A + reviewer-B + reviewer-C (SERENTAK)
  │       ← tunggu semua selesai
  │       handle NEEDS_FIX per entity (tidak blokir entity lain)
  │       update session → Round 2 → dst
  │
  ├── PHASE 3: Wiring (lead session langsung)
  │   └── tulis internal/routes/routes.go
  │       tulis main.go
  │       commit
  │
  ├── QA VERIFY
  │   └── spawn qa-verify
  │       jika FAIL → spawn implementer untuk fix → retry
  │       jika PASS → lanjut
  │
  ├── QA COMPAT
  │   └── spawn qa-compat
  │       verifikasi JSON tags match legacy contract persis
  │
  ├── QA SECURITY
  │   └── spawn qa-security
  │       cek: auth, injection, tenant isolation, goroutine leak,
  │            timeout, panic recovery, N+1, Redis TTL
  │       CRITICAL/HIGH → fix dulu sebelum lanjut
  │       MEDIUM/LOW → advisory (masuk report, tidak blokir)
  │
  ├── QA REPORT
  │   └── spawn qa-report → agregasi ketiga laporan → final verdict
  │
  └── Print: DONE
```

### 4.2 Bagaimana Agent Berkomunikasi

Agent tidak saling bicara langsung. Mereka berkomunikasi lewat **file di disk**:

```
00-context.md (ditulis user)
  └──pre-read by lead──→ CONTEXT variable
                              │
                              ├──inject ke──→ doc-reader (Phase 0)
                              └──inject ke──→ brainstormer (prioritas tertinggi)

doc-reader
  └──writes──→ contract-matrix.md
                entities.md
                doc-reader-summary.md (includes context summary)
                     │
                     └──read by──→ brainstormer (via lead injection)
                                        │
                                        └──writes──→ design-decisions.md
                                                          │
                                                          └──read by──→ planner
                                                                            │
                                                                            └──writes──→ implementation-plan.md
                                                                                               │
                                                                                               └──read by──→ task-breaker
                                                                                                                 │
                                                                                                                 └──writes──→ tasks-superpowers.md
                                                                                                                              tasks-taiga.md
```

**session/current.md** adalah state machine — setiap agent dan lead session update file ini setelah setiap checkpoint. Ini yang membuat `/continue` bisa resume dari mana saja.

### 4.3 Paralelisme Development

Tanpa sistem ini, entity dikerjakan satu per satu:

```
Tanpa paralel (3 entities × 8 tasks = 24 steps sequential):
  otp → [task1, task2, ..., task8] → blacklist → [task1, ..., task8] → config → [task1, ..., task8]
  Estimasi: ~4 jam
```

Dengan sistem ini, entity berjalan paralel dalam round:

```
Dengan paralel (round-based):
  Round 1: otp-task1 ║ blacklist-task1 ║ config-task1  (parallel)
  Round 2: otp-task2 ║ blacklist-task2 ║ config-task2  (parallel)
  ...
  Round 8: otp-task8 ║ blacklist-task8 ║ config-task8  (parallel)
  Estimasi: ~45 menit
```

---

## 5. Kenapa Arsitektur Ini?

### 5.1 Native Claude Agent Teams vs Plugin Eksternal

Dua pilihan yang dievaluasi: native agent teams vs plugin seperti ClaudeFlow/Ruflo.

| Aspek | Native | Plugin Eksternal |
|-------|--------|-----------------|
| Setup | Edit file `.md` di folder `.claude/` | Install, config, dependency |
| Maintenance | Kamu kontrol penuh | Tergantung upstream |
| Debug | Baca file `.md` langsung | Black box |
| Customization | Ubah instruksi langsung | Terbatas ke API plugin |
| Biaya | Token Claude saja | Token + biaya plugin |
| Stabilitas | Stabil (file lokal) | Bergantung availability plugin |

**Pilihan: Native.** Tidak ada dependency eksternal, tidak ada lock-in.

### 5.2 Kenapa Planning Sequential, Development Parallel?

**Planning harus sequential** karena setiap agent butuh output dari agent sebelumnya:
- brainstormer tidak bisa desain tanpa contract matrix dari doc-reader
- planner tidak bisa tulis kode tanpa design decisions dari brainstormer
- task-breaker tidak bisa buat routing tanpa implementation plan dari planner

**Development bisa parallel** karena setiap entity independen:
- Entity `otp` tidak bergantung pada progress entity `blacklist`
- Mereka share database layer tapi masing-masing punya file sendiri
- Dependency hanya di Phase 1 (shared components harus selesai dulu)

### 5.3 Kenapa Fresh Agent per Task, Bukan Agent Persisten?

Setiap task di-implement oleh fresh general-purpose agent (bukan agent yang sama terus). Alasannya:

1. **Context window**: agent yang mengerjakan 20 task berturut-turut akan penuh context, mulai "lupa" instruksi awal
2. **Fokus**: fresh agent hanya tahu task ini — tidak ada noise dari task sebelumnya
3. **Parallelism**: bisa spawn 10 agent sekaligus karena masing-masing fresh dan isolated
4. **Cost**: agent yang selesai langsung dibuang, tidak ada biaya idle

### 5.4 Kenapa Skills Dipisah dari Agent Definition?

Skills (`go-entity.md`, `go-repository.md`, dll) adalah **knowledge base** yang bisa dipakai oleh agent manapun. Dipisah karena:

- Implementer entity butuh `go-entity` + `go-dto` + `go-testing`
- Implementer repository butuh `go-repository` + `go-domain-interface` + `go-testing`
- Reviewer butuh semua skills untuk cek
- Kalau digabung ke dalam agent definition → setiap agent bawa semua knowledge padahal hanya pakai sebagian

---

## 6. Kenapa CLAUDE.md Singkat?

### CLAUDE.md adalah Root Node dari Knowledge Graph

Sistem ini menggunakan pola **knowledge graph** — mirip seperti Obsidian graph view, di mana setiap file adalah node, dan referensi antar file membentuk edge. CLAUDE.md bukan tempat menyimpan semua instruksi, tapi **titik masuk (root node)** yang menunjuk ke node lain.

```
                        CLAUDE.md (root)
                       /       |        \
              commands/     agents/    skills/
             /    |    \    /  |  \    /     \
         jr-init  │  change  │   │  go/    core/
       jr-resume  │          │   │   │       │
        continue  │       doc-  │  go-entity  tdd
                  │      reader │  go-repo    git
                  │    brainstorm  go-service  ...
                  │      planner
                  │    task-breaker
                  │      reviewer
                  │      qa-*
```

Ketika agent di-spawn, dia tidak perlu tahu semua hal — dia hanya perlu tahu **node yang relevan dengan tugasnya**. Implementer entity cukup membaca `go-entity.md` + `go-dto.md`. Dia tidak perlu membaca instruksi QA atau cara kerja brainstormer.

### Analoginya: Buku vs Ensiklopedia

| Pendekatan | Deskripsi | Masalah |
|------------|-----------|---------|
| CLAUDE.md panjang | Semua instruksi di satu file | Setiap agent baca semua, padahal hanya butuh sebagian |
| CLAUDE.md sebagai index | Pointer ke file yang relevan | Agent baca hanya yang dibutuhkan ✅ |

Ini persis seperti Obsidian graph — dokumen tidak menyimpan semua konten, tapi saling terhubung. Yang membuka dokumen hanya membaca node yang relevan untuk konteksnya.

### Masalah dengan CLAUDE.md Panjang

`CLAUDE.md` dimuat ke context window **setiap kali agent di-spawn**. Semakin panjang, semakin banyak token terbuang sebagai repetisi.

```
CLAUDE.md 225 baris × 15 agent spawns per service
= ~150 tokens/baris × 225 × 15
= ~506.000 token ekstra per service — semua repetisi
```

Dengan CLAUDE.md 55 baris sebagai root node:
```
= ~150 tokens/baris × 55 × 15
= ~123.750 token
  Hemat: ~382.250 token per service
```

Detail instruksi hanya dibaca oleh agent yang memang butuh:

| Informasi | Disimpan di | Dibaca oleh |
|-----------|------------|-------------|
| Cara kerja doc-reader | `agents/doc-reader.md` | doc-reader saja |
| Pattern Go repository | `skills/go/go-repository.md` | implementer repo saja |
| TDD workflow | `skills/core/tdd.md` | semua implementer |
| Flow planning | `commands/jr-init.md` | lead session saat /jr-init |

### Graf Dependency antar Node — Model Push, Bukan Pull

Penting: agent **tidak** mencari file skill sendiri. Lead session yang membaca lalu meng-inject konten ke dalam prompt agent sebelum dispatch. Ini model **push**, bukan pull.

Kenapa push lebih aman:
- Pull: agent harus tahu path file → bisa salah tebak, bisa skip diam-diam
- Push: lead baca dulu, kalau file tidak ketemu → error di lead, ketahuan lebih awal

```
jr-init.md (lead session)
  │
  ├── pre-read output files → inject ke prompt brainstormer
  │     (doc-reader-summary.md, contract-matrix.md, conflict-resolutions.md)
  ├── pre-read output files → inject ke prompt planner
  │     (design-decisions.md, contract-matrix.md, entities.md)
  └── pre-read output files → inject ke prompt task-breaker
        (implementation-plan.md)

continue.md (lead session)
  │
  ├── baca tasks-superpowers.md → routing map (siapa kerjakan apa)
  │
  ├── per task sebelum dispatch implementer:
  │     baca skill files sesuai layer → inject ke prompt
  │     baca design-decisions.md → inject ke prompt
  │     baca contract-matrix.md → inject ke prompt (jika dto/handler)
  │
  │     Layer → Skill files yang di-inject:
  │       entity      → go-entity.md + tdd.md
  │       dto         → go-dto.md + tdd.md
  │       repository  → go-repository.md + go-domain-interface.md + tdd.md
  │       service     → go-service.md + go-domain-interface.md + tdd.md
  │       handler-http → go-handler-http.md + tdd.md
  │       handler-rpc  → go-handler-rpc.md + tdd.md
  │
  └── spawn reviewer → reviewer.md
        (reviewer punya tools Read sendiri, baca file hasil implementasi langsung)

QA phase
  └── spawn qa-verify   → qa-verify.md   (Bash, Read, Grep — build + test + pattern)
  └── spawn qa-compat   → qa-compat.md   (Bash, Grep — JSON tags vs contract matrix)
  └── spawn qa-security → qa-security.md (Bash, Grep — auth, injection, leak, N+1, Redis)
  └── spawn qa-report   → qa-report.md   (haiku — agregasi 3 laporan, final verdict)
```

Hasilnya: setiap agent menerima **persis konten yang dia butuhkan** di dalam promptnya — tidak lebih, tidak kurang. Graph tetap terjaga karena resolusi path terjadi di lead session, bukan di dalam agent yang bisa saja gagal diam-diam.

### Prompt Caching

Claude Code mendukung **prompt caching** — context yang identik dalam window 5 menit di-cache, tidak dihitung ulang. CLAUDE.md singkat + stabil = kandidat cache yang ideal.

```
Tanpa caching (CLAUDE.md panjang):  setiap spawn hitung semua token dari nol
Dengan caching (CLAUDE.md singkat): CLAUDE.md di-cache, hanya instruksi task yang fresh
```

Hasilnya: spawn agent ke-2, ke-3, ke-10 jauh lebih murah dan lebih cepat dari spawn pertama.

---

## 7. Struktur File dan Fungsinya

```
.claude/
│
├── CLAUDE.md                    ← Index singkat (55 baris). Dimuat di setiap spawn.
│
├── settings.local.json          ← Enable agent teams + permission Bash commands
│
├── commands/                    ← Slash commands (dijalankan langsung oleh user)
│   ├── jr-init.md               ← /jr-init  : service baru dari nol
│   ├── jr-resume.md             ← /jr-resume: tambah fitur ke service yang ada
│   └── continue.md              ← /continue : lanjut development + QA otomatis
│   └── change.md                ← /change   : handle perubahan requirement
│
├── agents/                      ← Definisi subagent (dipanggil via Agent())
│   ├── doc-reader.md            ← Baca context + legacy code + docs, extract contracts
│   ├── brainstormer.md          ← Desain arsitektur, 2-3 approach per section
│   ├── planner.md               ← Tulis implementation plan dengan actual code
│   ├── task-breaker.md          ← Generate routing map + Taiga task mapping
│   ├── reviewer.md              ← Two-stage review: spec compliance + code quality
│   ├── qa-verify.md             ← Build, vet, test, pattern checks
│   ├── qa-compat.md             ← Backward compat: JSON tags match legacy contract
│   ├── qa-security.md           ← Security, leak, concurrency, perf analysis
│   └── qa-report.md             ← Aggregate 3 laporan QA → final verdict
│
├── skills/
│   ├── go/                      ← Knowledge base Go patterns (14 files)
│   │   ├── go-entity.md         ← GORM entity struct patterns
│   │   ├── go-repository.md     ← Repository implementation patterns
│   │   ├── go-service.md        ← Service layer patterns
│   │   ├── go-handler-http.md   ← Echo HTTP handler patterns
│   │   └── ...
│   └── core/                    ← Knowledge base universal
│       ├── tdd.md               ← TDD workflow
│       ├── git-workflow.md      ← Worktree, branch, commit patterns
│       ├── change-management.md ← Mid-flow change protocol
│       └── taiga.md             ← Taiga ticket format
│
└── session/
    └── current.md               ← State machine: phase, completed tasks, entity status
                                    (gitignored — lokal only, untuk cross-session resume)
```

### Alur Data antar File

```
[User siapkan PLAN_PATH]
  PLAN_PATH/00-context.md       ← penjelasan situasi, constraints, keputusan pre-set
  PLAN_PATH/00-database-design/ ← schema + merge strategy
  PLAN_PATH/01-{source-a}/      ← migration docs WP
  PLAN_PATH/02-{source-b}/      ← migration docs EXT
  PLAN_PATH/03-taiga-stories.md ← dari PM (opsional)
     ↓
jr-init.md → pre-read 00-context.md → CONTEXT variable
           → session/current.md (state awal)
     ↓
doc-reader.md → Phase 0: proses CONTEXT (constraints + pre-set)
              → contract-matrix.md
                entities.md
                doc-reader-summary.md
     ↓
brainstormer.md (menerima CONTEXT + doc-reader output via injection)
              → design-decisions.md
     ↓
planner.md      → implementation-plan.md
     ↓
task-breaker.md → tasks-superpowers.md  ← dibaca /continue untuk routing
                  tasks-taiga.md        ← dibaca PM untuk import ke Taiga
     ↓
/continue membaca tasks-superpowers.md
     → dispatch implementer per task
     → implementer baca implementation-plan.md + skill files
     → reviewer baca hasil implementasi + contract-matrix.md
     ↓
qa-verify → qa-compat → qa-security → qa-report (final verdict)
```

---

## 8. Manfaat untuk PM

### Input PM ke Sistem

PM hanya perlu menulis user stories dalam format markdown dan menaruhnya di:
```
docs/project/plan/E05-otp/03-taiga-stories.md
```

Format bebas — sistem membaca dan memetakan ke implementation tasks.

### Output yang PM Terima

1. **`tasks-taiga.md`** — setiap PM user story dipetakan ke sub-tasks implementasi. PM bisa lihat apakah semua acceptance criteria tercakup, dan ada flag ⚠️ kalau ada US yang tidak punya task atau task yang tidak punya US.

2. **QA Report** — laporan otomatis setelah development selesai berisi:
   - Build status
   - Test coverage
   - Backward compat check (field names match legacy atau tidak)

3. **Tidak ada kejutan di FE** — sistem enforce bahwa response field names harus match legacy persis. Kalau ada mismatch, QA compat gagal sebelum code di-merge.

### Visibilitas Progress

`session/current.md` update real-time selama development:
```
COMPLETED:    5/24 tasks done
entity_A: task-3/8
entity_B: task-2/8
entity_C: task-3/8
```

---

## 9. Manfaat untuk Tech Lead

### Konsistensi Pattern

Setiap task di-implement mengikuti skill files yang sama. Tidak ada lagi:
- Developer A pakai pattern repository berbeda dari Developer B
- Lupa app_origin filter di query unified table
- JSON response tags tidak match legacy

### Two-Stage Review Otomatis

Setiap implementasi di-review dua kali sebelum dianggap selesai:

**Stage 1 — Spec Compliance:**
- Apakah semua field dari contract-matrix.md ada?
- Apakah app_origin filter ada di setiap query unified table?
- Apakah interface mengikuti ISP (≤5 methods)?

**Stage 2 — Code Quality:**
- `go build ./...` — tidak ada compile error
- `go vet ./...` — tidak ada static analysis warning
- `go test ./test/{layer}/...` — semua test pass

Kalau NEEDS_FIX → implementer fix → re-review. Loop sampai APPROVED.

### QA Backward Compatibility

`qa-compat` secara spesifik cek:
1. Setiap response field dari contract-matrix.md ada di kode
2. Tidak ada field ekstra yang tidak ada di contract (strict-mode client protection)
3. JSON tag persis sama — case-sensitive

### QA Security & Reliability

`qa-security` cek 10 kategori. Severity CRITICAL/HIGH memblokir merge:

| Kategori | Contoh temuan | Severity |
|----------|---------------|---------|
| Auth & authorization | Endpoint tanpa middleware | CRITICAL |
| Input validation & injection | Raw SQL concatenation | CRITICAL |
| Tenant isolation (app_origin) | Query unified table tanpa filter | CRITICAL |
| Sensitive data exposure | Password di response/log | CRITICAL |
| Goroutine & resource leak | `go func` tanpa exit condition | HIGH |
| Context & timeout | HTTP client tanpa timeout | HIGH |
| Panic recovery | Handler tanpa recover middleware | HIGH |
| Connection pool & unbounded query | SELECT tanpa LIMIT | MEDIUM |
| N+1 query | DB call di dalam for-loop | MEDIUM |
| Redis TTL & cache invalidation | SET tanpa expire | MEDIUM |

MEDIUM/LOW dicatat sebagai advisory di laporan final — tidak blokir, tapi tim tahu.

### Cross-Session Resume

Development bisa berhenti kapan saja (koneksi putus, mau istirahat, dll) dan dilanjutkan dengan `/continue`. State tersimpan di `session/current.md` — sistem tahu persis task mana yang sudah selesai, entity mana yang sedang dikerjakan.

### Mid-Flow Change Handling

Kalau PM minta perubahan di tengah development:

1. `/change` di-trigger
2. Sistem assess: task mana yang aman (sudah selesai), task mana yang perlu di-adjust
3. Completed tasks tidak perlu diulang — hanya task yang terpengaruh yang di-re-plan
4. Semua perubahan tercatat di `session/changelog.md` untuk audit trail

---

## Ringkasan

| Aspek | Sebelum | Sesudah |
|-------|---------|---------|
| Waktu planning per service | 8–15 jam | ~15-30 menit (user aktif) |
| Waktu implementation per service | varies | Otomatis, paralel |
| Konsistensi pattern | Bergantung developer | Enforced via skill files |
| Context arsitektur | Di kepala tech lead | `00-context.md` — explicitly documented |
| Backward compat check | Manual / sering missed | Otomatis di QA compat |
| Security & reliability check | Manual / ad-hoc | Otomatis di QA security (10 kategori) |
| Resume setelah interupsi | Mulai ulang / tebak progress | `/continue` dari checkpoint |
| Perubahan requirement | Rework besar | Impact assessment + partial re-plan |
| Input PM | Tidak ada alur formal | Taiga stories → mapped ke impl tasks |
| Audit trail | Tidak ada | session/changelog.md per perubahan |
