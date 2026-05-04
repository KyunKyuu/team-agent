---
description: |
  Launch the Planning Team for a unified service rebuild (WP + EXT → single Go service).
  Two legacy sources (jr-web-partner + jr-external) merged into one new service while preserving
  backward-compatible API contracts for both scopes.
  Trigger when: user wants to plan a new unified service, says "jr-init", "planning baru", "rebuild service".
---

# JR Init — Unified Service Planning Team

Merges two legacy services (WP = Node.js monolith, EXT = Go microservice) into one new Go service.
**Hard constraint: request/response contracts MUST NOT change — frontend cannot adapt.**

Flow: gather params → doc-reader (both sources) → **contract conflict gate** → brainstormer → planner → task-breaker → STOP.

---

## Step 1: Ask SERVICE_NAME

```
Question: "Nama service baru yang akan dibuat? (contoh: otp-general, notification-general)"
Header: "Service Name"
Free text input.
```

Validate: lowercase, hyphen-separated, ends with "-general" atau custom. Simpan sebagai SERVICE_NAME.

---

## Step 2: Ask SUBMODULE_ROOT

Legacy code WP dan EXT tersedia sebagai folder referensi lokal di dalam project (gitignored — tidak ikut push ke service baru).

```
Question: "Path ke folder legacy reference? (gitignored, berisi WP + EXT source)"
Header: "Legacy Reference Root"
Options:
  - "legacy/" (default — relatif dari BASE_PATH)
  - "Other" (custom path)
MultiSelect: false
```

If "Other", ask user to type the path. Path boleh relatif dari BASE_PATH atau absolut.

### Step 2b: Verify Legacy Folder Exists

Setelah user input SUBMODULE_ROOT, jalankan check:

```bash
ls "{SUBMODULE_ROOT}apps/be/jr-web-partner/" 2>/dev/null && \
ls "{SUBMODULE_ROOT}apps/be/jr-external/" 2>/dev/null && \
echo "OK" || echo "MISSING"
```

**Jika MISSING**, tampilkan:
```
⚠️  Legacy folder tidak ditemukan di {SUBMODULE_ROOT}

Pastikan folder legacy sudah ada sebelum lanjut.
Opsi:
  A. Clone/copy manual ke {SUBMODULE_ROOT} lalu ketik "retry"
  B. Ubah path SUBMODULE_ROOT → kembali ke Step 2
```

**Jangan lanjut ke Step 3 sampai folder ditemukan.**

---

## Step 3: Ask BASE_PATH

```
Question: "Path ke target project (boilerplate untuk service baru)?"
Header: "Base Path"
Options:
  - "apps/be/rebuild-general/{SERVICE_NAME}/" (pre-filled dengan SERVICE_NAME)
  - "Other" (custom path)
MultiSelect: false
```

If "Other", ask user to type the complete path.

---

## Step 4: Ask PLAN_PATH

```
Question: "Path ke epic plan folder?"
Header: "Plan Path"
Options:
  - "docs/project/plan/E05-{SERVICE_NAME}/" (pre-filled)
  - "Other" (custom path)
MultiSelect: false
```

If "Other", ask user to type the path.

---

## Step 5: Derive All Paths

EPIC di-extract otomatis dari PLAN_PATH — tidak perlu ditanya ulang.

```
EPIC = extract from PLAN_PATH — ambil segment yang diawali huruf E diikuti angka
       contoh: "docs/project/plan/E05-otp/" → EPIC = "E05"
       contoh: "docs/project/plan/E12-notification/" → EPIC = "E12"
       Jika tidak ditemukan, tanya: "Epic ID? (contoh: E05)"
```

Compute semua derived parameters:

```
BOILERPLATE      = BASE_PATH
OUTPUT_PATH      = "{BASE_PATH}docs/"
WP_LEGACY_PATH   = "{SUBMODULE_ROOT}apps/be/jr-web-partner/"
EXT_LEGACY_PATH  = "{SUBMODULE_ROOT}apps/be/jr-external/"
WP_DOCS_PATH     = "{PLAN_PATH}02-jr-web-partner/"
EXT_DOCS_PATH    = "{PLAN_PATH}01-jr-external/"
DB_DESIGN_PATH   = "{PLAN_PATH}00-database-design/"
CONTEXT_PATH     = "{PLAN_PATH}00-context.md"
TAIGA_STORIES    = "{PLAN_PATH}03-taiga-stories.md"
REPORT_PATH      = "docs/project/reports/{EPIC}-{SERVICE_NAME}/"
TEAM_NAME        = "planning-{SERVICE_NAME}-{EPIC}"
```

---

## Step 7: Confirm Parameters

Print summary:

```
┌──────────────────────────────────────────────────────┐
│  JR Init — Planning Team Parameter Summary           │
├──────────────────────────────────────────────────────┤
│  SERVICE_NAME:    {SERVICE_NAME}                     │
│  EPIC:            {EPIC}  (derived dari PLAN_PATH)   │
├──────────────────────────────────────────────────────┤
│  SUBMODULE_ROOT:  {SUBMODULE_ROOT}  ✅ verified      │
│  WP_LEGACY_PATH:  {WP_LEGACY_PATH}  (derived)        │
│  EXT_LEGACY_PATH: {EXT_LEGACY_PATH} (derived)        │
├──────────────────────────────────────────────────────┤
│  BASE_PATH:       {BASE_PATH}                        │
│  PLAN_PATH:       {PLAN_PATH}                        │
│  OUTPUT_PATH:     {OUTPUT_PATH}                      │
├──────────────────────────────────────────────────────┤
│  CONTEXT_PATH:    {CONTEXT_PATH}                     │
│    → (dibaca sebelum doc-reader berjalan)            │
│  TAIGA_STORIES:   {TAIGA_STORIES}                    │
│    → (akan dicek saat doc-reader berjalan)           │
├──────────────────────────────────────────────────────┤
│  ENTITIES:  (auto-extracted by doc-reader)           │
│  CONTRACTS: (auto-extracted by doc-reader)           │
└──────────────────────────────────────────────────────┘
```

Ask user to confirm:

```
Question: "Parameter sudah benar? Lanjutkan spawn doc-reader?"
Header: "Confirm"
Options:
  - "Ya, lanjutkan"
  - "Tidak, ada yang perlu diperbaiki"
MultiSelect: false
```

If "Tidak", ask which parameter to fix and go back to the relevant step.

### Initialize Session State

After user confirms, use the Write tool to create `{OUTPUT_PATH}session/current.md`:

```markdown
# Session State — {SERVICE_NAME} ({EPIC})

## Active Project
SERVICE_NAME:  {SERVICE_NAME}
EPIC:          {EPIC}
BASE_PATH:     {BASE_PATH}
OUTPUT_PATH:   {OUTPUT_PATH}
WORKTREE_PATH: (not yet created)
BRANCH:        (not yet created)

## Current Phase
PHASE:      planning
LAST_TEAM:  (none)

## Planning Team
STATUS:     in-progress
LAST_AGENT: (none)
doc-reader:      pending
brainstormer:    pending
planner:         pending
task-breaker:    pending

## Development
STATUS:       not-started
TOTAL_TASKS:  0
COMPLETED:    0
CURRENT_TASK: (none)

## QA Team
STATUS:      not-started
QA_VERIFY:   not-run
QA_COMPAT:   not-run
QA_SECURITY: not-run
QA_REPORT:   not-run

## Changes
CHANGES:     0
LAST_CHANGE: (none)
CHANGELOG:   {OUTPUT_PATH}session/changelog.md
```

---

## Step 8a: Spawn doc-reader

### Pre-spawn: Read service context

Check if `{CONTEXT_PATH}` exists and read it:

```bash
ls "{CONTEXT_PATH}" 2>/dev/null && echo "EXISTS" || echo "MISSING"
```

- If EXISTS: use Read tool to read `{CONTEXT_PATH}` → store content as CONTEXT
- If MISSING: CONTEXT = "(context not provided)"

> **Format `00-context.md` yang diharapkan:**
> ```
> ## Situasi
> [kenapa service ini perlu dibangun/direbuild]
>
> ## Yang Dibangun
> [target state service ini setelah selesai]
>
> ## Constraints Khusus
> [rules spesifik service ini — di luar constraint generik]
>
> ## Integrasi
> [service lain yang terlibat, auth type, pola komunikasi]
>
> ## Keputusan Pre-set
> [keputusan arsitektur yang sudah fixed — brainstormer tidak perlu tanya]
>
> ## Hal yang Perlu Diperhatikan
> [edge cases, gotchas, hal tidak obvious dari legacy atau epic]
> ```
> File ini opsional, tapi sangat disarankan. Tanpa ini brainstormer akan menebak konteks.

Spawn doc-reader to read BOTH legacy sources and extract API contracts.

```
Agent({
  subagent_type: "doc-reader",
  name: "doc-reader",
  team_name: "{TEAM_NAME}",
  prompt: `
SERVICE_NAME     = "{SERVICE_NAME}"
SUBMODULE_ROOT   = "{SUBMODULE_ROOT}"
WP_LEGACY_PATH   = "{WP_LEGACY_PATH}"
EXT_LEGACY_PATH  = "{EXT_LEGACY_PATH}"
BASE_PATH        = "{BASE_PATH}"
BOILERPLATE      = "{BOILERPLATE}"
PLAN_PATH        = "{PLAN_PATH}"
WP_DOCS_PATH     = "{WP_DOCS_PATH}"
EXT_DOCS_PATH    = "{EXT_DOCS_PATH}"
DB_DESIGN_PATH   = "{DB_DESIGN_PATH}"
EPIC             = "{EPIC}"
CONTEXT          = "{CONTEXT}"

IMPORTANT: WP_LEGACY_PATH and EXT_LEGACY_PATH are inside SUBMODULE_ROOT, which is a local-only
reference folder (gitignored). Read them freely — they are real legacy code, not docs.

You are reading TWO legacy sources that will be merged into one unified service: {SERVICE_NAME}.

## Task A — Read WP Legacy (Node.js monolith)
Read actual source code at: {WP_LEGACY_PATH}
- Find and read route files, controller/handler files, middleware
- Extract all HTTP endpoints: method, path, handler function
- Extract request schema: field names, types, validation rules (from Joi/express-validator or similar)
- Extract response schema: exact field names and types — these are the contracts frontend depends on
- Extract business logic summary per endpoint
- Extract DB queries: table names, columns accessed, filters used (look for Sequelize/Knex/raw SQL)

## Task B — Read EXT Legacy (Go microservice)
Read actual source code at: {EXT_LEGACY_PATH}
- Find and read handler files, route files, DTO/model files
- Same extraction as Task A above
- Pay attention to JSON struct tags — these define the actual response field names

## Task C — Read Migration Docs
Read: {WP_DOCS_PATH} and {EXT_DOCS_PATH} and {DB_DESIGN_PATH}
- Extract scope of what is in-migration (what is included in this epic)
- Extract ENTITIES list with fields
- Identify Reuse / Extend / Create status per entity in boilerplate at {BOILERPLATE}

## Task D — Build API Contract Matrix
For each endpoint that exists in WP or EXT (or both), produce a contract table:

| Endpoint | Scope | Method | Request Fields | Response Fields | Conflict? |
|----------|-------|--------|---------------|-----------------|-----------|
| /otp/send | WP | POST | phone, type | code, expired_at | — |
| /otp/send | EXT | POST | phone_number, otp_type | otp_code, expires | ⚠️ field names differ |
| /otp/verify | WP only | POST | ... | ... | — |

Mark ⚠️ CONFLICT when: same endpoint path exists in both WP and EXT but has different field names or types.

## Output
Write the following files:

1. {OUTPUT_PATH}contract-matrix.md — API Contract Matrix (Task D)
2. {OUTPUT_PATH}entities.md — Entities, fields, Reuse/Extend/Create status
3. {OUTPUT_PATH}doc-reader-summary.md — Full extraction summary for brainstormer context

Then return a concise summary (max 300 words) of:
- ENTITIES list
- Number of endpoints per scope
- Number of CONFLICTS found
- Any ambiguities that need user clarification
  `
})
```

**Wait for doc-reader to complete.**

---

## Step 8b: Contract Conflict Gate

After doc-reader completes:

1. Use Read tool to read `{OUTPUT_PATH}contract-matrix.md`
2. Update `{OUTPUT_PATH}session/current.md`: set `doc-reader: done`

**If CONFLICTS found (⚠️ rows exist):**

Print the conflict table to user and ask for resolution per conflict:

```
⚠️  Contract Conflicts Ditemukan

Endpoint berikut ada di WP dan EXT dengan format berbeda:
[tampilkan baris ⚠️ dari contract-matrix.md]

Untuk setiap conflict, pilih salah satu:
  A. Pakai format WP → field names WP dipakai untuk semua client
  B. Pakai format EXT → field names EXT dipakai untuk semua client
  C. Pertahankan keduanya → endpoint terpisah per scope
  D. Custom → user tentukan mapping-nya
```

Collect all user resolutions, then use the Write tool to create `{OUTPUT_PATH}conflict-resolutions.md`:

```markdown
# Conflict Resolutions — {SERVICE_NAME}

## Resolution per Conflict

### {endpoint} — {scope_a} vs {scope_b}
Resolution: {A/B/C/D}
Detail: {specific field mapping decided}

[one section per conflict]
```

**If NO conflicts:**

Print: `✅ Tidak ada conflict. Semua endpoint compatible.`

Use the Write tool to create `{OUTPUT_PATH}conflict-resolutions.md` with content:

```markdown
# Conflict Resolutions — {SERVICE_NAME}

No conflicts found. All WP and EXT contracts are compatible.
```

This file must exist regardless — brainstormer reads it.

---

## Step 8c: Spawn brainstormer

### Pre-spawn: Read output files

Use the Read tool to read each file and store the content:
1. Read `{OUTPUT_PATH}doc-reader-summary.md` → DOC_READER_SUMMARY
2. Read `{OUTPUT_PATH}contract-matrix.md` → CONTRACT_MATRIX
3. Read `{OUTPUT_PATH}conflict-resolutions.md` → CONFLICT_RESOLUTIONS

(CONTEXT already read in Step 8a — reuse the same value.)

Then spawn brainstormer with the actual file contents injected:

```
Agent({
  subagent_type: "brainstormer",
  name: "brainstormer",
  team_name: "{TEAM_NAME}",
  prompt: `
SERVICE_NAME    = "{SERVICE_NAME}"
BASE_PATH       = "{BASE_PATH}"
BOILERPLATE     = "{BOILERPLATE}"
PLAN_PATH       = "{PLAN_PATH}"
OUTPUT_PATH     = "{OUTPUT_PATH}"
EPIC            = "{EPIC}"

--- 00-context.md (HIGHEST PRIORITY — constraints and pre-set decisions override all below) ---
{CONTEXT}
--- end ---

--- doc-reader-summary.md ---
{DOC_READER_SUMMARY}
--- end ---

--- contract-matrix.md ---
{CONTRACT_MATRIX}
--- end ---

--- conflict-resolutions.md ---
{CONFLICT_RESOLUTIONS}
--- end ---

## Your Task
Run Socratic brainstorming for the unified {SERVICE_NAME} service.
Propose 2-3 implementation approaches per design section.
Perform self-review before presenting to user.
Write final design decisions to {OUTPUT_PATH}design-decisions.md.

## HARD CONSTRAINTS — Non-negotiable, never propose alternatives:
1. JSON response field names MUST match legacy exactly as per contract matrix.
   Use conflict-resolutions.md for any resolved conflicts.
   NEVER rename or restructure a response field.
2. All queries on shared tables MUST filter by app_origin ("web_partner" or "external").
3. No GORM AutoMigrate in production code. SQL files are source of truth.
4. Two channels per scope: REST API + RabbitMQ RPC → same service layer.
5. WP clients get WP-format responses. EXT clients get EXT-format responses.

## Sections to brainstorm:
- Route design (how to separate WP vs EXT traffic?)
- app_origin injection strategy
- Shared vs split DTOs
- Error response format per scope
- Database layer (shared repo with app_origin param?)
  `
})
```

**Wait for brainstormer to complete.**

After brainstormer completes, update `{OUTPUT_PATH}session/current.md`: set `brainstormer: done`.

---

## Step 8d: Spawn planner

### Pre-spawn: Read output files

Use the Read tool to read each file and store the content:
1. Read `{OUTPUT_PATH}design-decisions.md` → DESIGN_DECISIONS
2. Read `{OUTPUT_PATH}contract-matrix.md` → CONTRACT_MATRIX
3. Read `{OUTPUT_PATH}entities.md` → ENTITIES

Then spawn planner with actual file contents injected:

```
Agent({
  subagent_type: "planner",
  name: "planner",
  team_name: "{TEAM_NAME}",
  prompt: `
SERVICE_NAME    = "{SERVICE_NAME}"
BASE_PATH       = "{BASE_PATH}"
BOILERPLATE     = "{BOILERPLATE}"
PLAN_PATH       = "{PLAN_PATH}"
OUTPUT_PATH     = "{OUTPUT_PATH}"
EPIC            = "{EPIC}"

--- design-decisions.md ---
{DESIGN_DECISIONS}
--- end ---

--- contract-matrix.md ---
{CONTRACT_MATRIX}
--- end ---

--- entities.md ---
{ENTITIES}
--- end ---

## Your Task
Produce a bite-sized implementation plan for {SERVICE_NAME}.
Each task = 2-5 minutes, TDD per step, actual code, exact file paths. No placeholders.
Write to {OUTPUT_PATH}implementation-plan.md.

## Plan must include for each entity:
1. Config & constants
2. Entity struct (GORM, with TableName returning shared table name)
3. DTOs per scope (WP DTO + EXT DTO if response format differs)
4. Repository interface (ISP — split read/write if needed)
5. Repository implementation (all queries MUST include app_origin filter)
6. Service interface + implementation
7. HTTP handler per scope (thin — only binds request, calls service, formats response)
8. RPC handler per scope (RabbitMQ)
9. Routes registration

## Per-task format:
### Task N: [title]
**File:** exact/path/to/file.go
**Test first:** write this failing test → verify it fails → implement → verify it passes
**Code:**
[actual code, not pseudocode]
**Commit:** feat({SERVICE_NAME}): [description]
  `
})
```

**Wait for planner to complete.**

After planner completes, update `{OUTPUT_PATH}session/current.md`: set `planner: done`.

---

## Step 8e: Spawn task-breaker

### Pre-spawn: Read output file

Use the Read tool to read:
1. Read `{OUTPUT_PATH}implementation-plan.md` → IMPLEMENTATION_PLAN

Then spawn task-breaker with actual content injected:

```
Agent({
  subagent_type: "task-breaker",
  name: "task-breaker",
  team_name: "{TEAM_NAME}",
  prompt: `
SERVICE_NAME = "{SERVICE_NAME}"
BASE_PATH    = "{BASE_PATH}"
PLAN_PATH    = "{PLAN_PATH}"
OUTPUT_PATH  = "{OUTPUT_PATH}"
EPIC         = "{EPIC}"

--- implementation-plan.md ---
{IMPLEMENTATION_PLAN}
--- end ---

## Your Task
Generate two task breakdown files:

### 1. Superpowers Tasks → {OUTPUT_PATH}tasks-superpowers.md
Group tasks by entity. For each entity, list tasks in strict order:
config → entity → dto → repo-interface → repo-impl → service-interface → service-impl → handler-http → handler-rpc → routes

Mark parallelism:
- ✅ PARALLEL: tasks for different entities (notification vs device_token)
- 🔒 SEQUENTIAL: tasks within same entity

### 2. Taiga-ready Tasks → {OUTPUT_PATH}tasks-taiga.md
Format each task as a Taiga user story with acceptance criteria.
  `
})
```

**Wait for task-breaker to complete.**

After task-breaker completes, update `{OUTPUT_PATH}session/current.md`:
- Set `task-breaker: done`
- Set `STATUS: waiting-review`
- Set `PHASE: planning` (stays until user says "lanjut")
- Read `{OUTPUT_PATH}tasks-superpowers.md` and count total tasks → set `TOTAL_TASKS: N`

---

## Step 9: STOP — User Review

After all agents complete, print:

```
╔══════════════════════════════════════════════════════╗
║  Planning Team Selesai — {SERVICE_NAME} ({EPIC})     ║
╠══════════════════════════════════════════════════════╣
║  Output files:                                       ║
║                                                      ║
║  {OUTPUT_PATH}contract-matrix.md      (doc-reader)   ║
║  {OUTPUT_PATH}entities.md             (doc-reader)   ║
║  {OUTPUT_PATH}conflict-resolutions.md (conflict gate)║
║  {OUTPUT_PATH}design-decisions.md     (brainstormer) ║
║  {OUTPUT_PATH}implementation-plan.md  (planner)      ║
║  {OUTPUT_PATH}tasks-superpowers.md    (task-breaker) ║
║  {OUTPUT_PATH}tasks-taiga.md          (task-breaker) ║
╠══════════════════════════════════════════════════════╣
║  STOP — Review output sebelum lanjut.                ║
║                                                      ║
║  Cek terutama:                                       ║
║  1. contract-matrix.md — semua endpoint tercakup?    ║
║  2. design-decisions.md — backward compat terjaga?   ║
║  3. implementation-plan.md — ada placeholder?        ║
║                                                      ║
║  Jika ada yang perlu diperbaiki, bilang ke lead.     ║
║  Jika sudah benar, ketik "lanjut" untuk Git Worktree.║
╚══════════════════════════════════════════════════════╝
```

Do NOT proceed to Git Worktree or Development until user explicitly says "lanjut".

---

## When User Says "lanjut": Git Worktree Setup

### Create worktree

```bash
git worktree add ".claude/worktrees/{SERVICE_NAME}" -b feat/{SERVICE_NAME}
```

If worktree already exists:
```bash
git worktree list
```
Confirm branch name and path, then continue.

### Update session state

Use the Write tool to update `{OUTPUT_PATH}session/current.md`:
- Set `PHASE: development`
- Set `WORKTREE_PATH: .claude/worktrees/{SERVICE_NAME}`
- Set `BRANCH: feat/{SERVICE_NAME}`

### Hand off to development

Print:
```
╔══════════════════════════════════════════════════════╗
║  Worktree siap.                                      ║
║                                                      ║
║  Branch:   feat/{SERVICE_NAME}                       ║
║  Path:     .claude/worktrees/{SERVICE_NAME}          ║
║                                                      ║
║  Jalankan /continue untuk mulai development loop.    ║
║  /continue akan baca tasks-superpowers.md dan        ║
║  dispatch implementer agent per task.                ║
╚══════════════════════════════════════════════════════╝
```

**STOP.** Development loop dijalankan via `/continue`, bukan otomatis dari jr-init.
