---
description: |
  Launch the Planning Team for adding a new feature/epic to an EXISTING unified service.
  Uses the same WP + EXT legacy sources but targets a project that already has code —
  existing entities are Reuse, new entities are Create, shared components may be Extend.
  Trigger when: user wants to add a feature to an existing service, says "jr-resume",
  "tambahin fitur", "epic baru di service yang sama", "lanjut service yang sudah ada".
---

# JR Resume — Add Feature to Existing Service

Adds new entities/endpoints from a new epic to a Go service that already exists.
**Hard constraint: request/response contracts MUST NOT change — frontend cannot adapt.**
**Hard constraint: existing code MUST NOT be modified unless explicitly Extend.**

Flow: gather params → scan existing project → doc-reader → **contract conflict gate** → brainstormer → planner → task-breaker → STOP.

---

## Step 1: Ask SERVICE_NAME

```
Question: "Nama service yang sudah ada? (contoh: otp-general, notification-general)"
Header: "Service Name"
Free text input.
```

Validate: lowercase, hyphen-separated. Simpan sebagai SERVICE_NAME.

---

## Step 2: Ask FEATURE_NAME

Fitur/entity baru apa yang akan ditambahkan ke service ini?

```
Question: "Nama fitur atau entity baru yang akan ditambahkan?"
Header: "Feature Name"
Contoh: "blacklist", "resend-limit", "audit-log", "user-preference"
Free text input.
```

Simpan sebagai FEATURE_NAME. Ini dipakai untuk scope di doc-reader — bukan nama service.

---

## Step 3: Ask SUBMODULE_ROOT

Legacy code WP dan EXT tersedia sebagai folder referensi lokal (gitignored).

```
Question: "Path ke folder legacy reference? (gitignored, berisi WP + EXT source)"
Header: "Legacy Reference Root"
Options:
  - "legacy/" (default — relatif dari BASE_PATH)
  - "Other" (custom path)
MultiSelect: false
```

If "Other", ask user to type the path.

### Step 3b: Verify Legacy Folder Exists

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
  B. Ubah path SUBMODULE_ROOT → kembali ke Step 3
```

**Jangan lanjut ke Step 4 sampai folder ditemukan.**

---

## Step 4: Ask BASE_PATH

Path ke project yang sudah ada (bukan boilerplate fresh — sudah ada kode di dalamnya).

```
Question: "Path ke project yang sudah ada?"
Header: "Base Path"
Options:
  - "apps/be/rebuild-general/{SERVICE_NAME}/" (pre-filled)
  - "Other" (custom path)
MultiSelect: false
```

If "Other", ask user to type the complete path.

### Step 4b: Verify Existing Project

```bash
ls "{BASE_PATH}go.mod" 2>/dev/null && \
ls "{BASE_PATH}internal/" 2>/dev/null && \
echo "OK" || echo "MISSING"
```

**Jika MISSING**, tampilkan:
```
⚠️  Project Go tidak ditemukan di {BASE_PATH}

Tidak ada go.mod atau internal/ di path tersebut.
Opsi:
  A. Cek path lagi → kembali ke Step 4
  B. Kalau ini memang project baru (belum ada kode), pakai /jr-init
```

**Jangan lanjut sampai go.mod dan internal/ ditemukan.**

---

## Step 5: Ask PLAN_PATH

```
Question: "Path ke epic plan folder untuk fitur baru ini?"
Header: "Plan Path"
Options:
  - "docs/project/plan/E{XX}-{FEATURE_NAME}/" (pre-filled)
  - "Other" (custom path)
MultiSelect: false
```

If "Other", ask user to type the path.

---

## Step 6: Derive All Paths

EPIC di-extract otomatis dari PLAN_PATH — tidak perlu ditanya ulang.

```
EPIC = extract from PLAN_PATH — ambil segment yang diawali huruf E diikuti angka
       contoh: "docs/project/plan/E06-blacklist/" → EPIC = "E06"
       Jika tidak ditemukan, tanya: "Epic ID? (contoh: E06)"
```

Compute semua derived parameters:

```
BOILERPLATE      = BASE_PATH  (existing project — scan to determine what already exists)
OUTPUT_PATH      = "{BASE_PATH}docs/{EPIC}-{FEATURE_NAME}/"
WP_LEGACY_PATH   = "{SUBMODULE_ROOT}apps/be/jr-web-partner/"
EXT_LEGACY_PATH  = "{SUBMODULE_ROOT}apps/be/jr-external/"
WP_DOCS_PATH     = "{PLAN_PATH}02-jr-web-partner/"
EXT_DOCS_PATH    = "{PLAN_PATH}01-jr-external/"
DB_DESIGN_PATH   = "{PLAN_PATH}00-database-design/"
TAIGA_STORIES    = "{PLAN_PATH}03-taiga-stories.md"
REPORT_PATH      = "docs/project/reports/{EPIC}-{FEATURE_NAME}/"
TEAM_NAME        = "planning-{SERVICE_NAME}-{EPIC}-{FEATURE_NAME}"
```

> OUTPUT_PATH pakai subfolder `{EPIC}-{FEATURE_NAME}/` supaya tidak menimpa planning epic sebelumnya di `{BASE_PATH}docs/`.

---

## Step 8: Confirm Parameters

Print summary:

```
┌──────────────────────────────────────────────────────────┐
│  JR Resume — Planning Team Parameter Summary             │
├──────────────────────────────────────────────────────────┤
│  SERVICE_NAME:    {SERVICE_NAME}  (existing service)     │
│  FEATURE_NAME:    {FEATURE_NAME}  (new feature/entity)   │
│  EPIC:            {EPIC}  (derived dari PLAN_PATH)       │
├──────────────────────────────────────────────────────────┤
│  SUBMODULE_ROOT:  {SUBMODULE_ROOT}  ✅ verified           │
│  WP_LEGACY_PATH:  {WP_LEGACY_PATH}  (derived)            │
│  EXT_LEGACY_PATH: {EXT_LEGACY_PATH} (derived)            │
├──────────────────────────────────────────────────────────┤
│  BASE_PATH:       {BASE_PATH}  ✅ existing project        │
│  PLAN_PATH:       {PLAN_PATH}                            │
│  OUTPUT_PATH:     {OUTPUT_PATH}  (per-epic subfolder)    │
├──────────────────────────────────────────────────────────┤
│  TAIGA_STORIES:   {TAIGA_STORIES}                        │
│    → (akan dicek saat doc-reader berjalan)               │
├──────────────────────────────────────────────────────────┤
│  Mode: ADD TO EXISTING  (tidak membuat service baru)     │
│  Existing code di {BASE_PATH}: akan di-scan doc-reader   │
│  New entities dari {FEATURE_NAME}: akan di-plan          │
└──────────────────────────────────────────────────────────┘
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
# Session State — {SERVICE_NAME} / {FEATURE_NAME} ({EPIC})

## Active Project
SERVICE_NAME:  {SERVICE_NAME}
FEATURE_NAME:  {FEATURE_NAME}
EPIC:          {EPIC}
BASE_PATH:     {BASE_PATH}
OUTPUT_PATH:   {OUTPUT_PATH}
MODE:          extend-existing
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
STATUS:    not-started
QA_VERIFY: not-run
QA_COMPAT: not-run
QA_REPORT: not-run

## Changes
CHANGES:     0
LAST_CHANGE: (none)
CHANGELOG:   {OUTPUT_PATH}session/changelog.md
```

---

## Step 9a: Spawn doc-reader

Spawn doc-reader dengan mode `extend-existing` — fokus pada new feature dalam project yang sudah ada.

```
Agent({
  subagent_type: "doc-reader",
  name: "doc-reader",
  team_name: "{TEAM_NAME}",
  prompt: `
SERVICE_NAME     = "{SERVICE_NAME}"
FEATURE_NAME     = "{FEATURE_NAME}"
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
MODE             = "extend-existing"

IMPORTANT: MODE is "extend-existing". This is NOT a fresh service — the project at {BASE_PATH}
already has code. Your job is to scope only the NEW feature ({FEATURE_NAME}) being added.

## Task A — Scan Existing Project (what already exists)
Read actual files at: {BASE_PATH}internal/
- List existing entities: internal/entity/
- List existing DTOs: internal/dto/
- List existing repositories: internal/repository/
- List existing services: internal/service/
- List existing handlers: internal/handler/
- List existing config, middleware, pkg/ — mark each as Reuse (no changes needed)
These are ALL Reuse — document them so planner knows NOT to re-implement.

## Task B — Read WP Legacy (scope: {FEATURE_NAME} only)
Read: {WP_LEGACY_PATH}
Focus only on the new feature: {FEATURE_NAME}
- Find routes, handlers, DTOs related to {FEATURE_NAME}
- Extract exact response field names — these are the contracts frontend depends on
- Extract request field names and validation rules
- Extract DB queries: table names, columns, app_origin usage

## Task C — Read EXT Legacy (scope: {FEATURE_NAME} only)
Read: {EXT_LEGACY_PATH}
Focus only on the new feature: {FEATURE_NAME}
- Same extraction as Task B
- Read struct JSON tags precisely — these ARE the response field names

## Task D — Read Migration Docs (new epic only)
Read: {WP_DOCS_PATH}, {EXT_DOCS_PATH}, {DB_DESIGN_PATH}
Extract: entities, routes, DTOs for {FEATURE_NAME} in {EPIC}
Mark Reuse/Extend/Create:
  - Reuse: already exists in {BASE_PATH}internal/, no changes needed
  - Extend: exists but needs new methods or fields (e.g., config with new env vars)
  - Create: new entity/DTO/repo/service/handler — does not exist yet

## Task E — Build API Contract Matrix (new endpoints only)
Same format as standard contract matrix, but ONLY for {FEATURE_NAME} endpoints.
Skip endpoints already implemented in the existing project.

## Output
Write:
1. {OUTPUT_PATH}contract-matrix.md  ← new-feature contracts only
2. {OUTPUT_PATH}entities.md         ← Reuse/Extend/Create per component
3. {OUTPUT_PATH}doc-reader-summary.md ← summary including existing project state
  `
})
```

**Wait for doc-reader to complete.**

---

## Step 9b: Contract Conflict Gate

Same as jr-init — read `{OUTPUT_PATH}contract-matrix.md`, resolve any WP vs EXT conflicts.

After doc-reader completes:
1. Use Read tool to read `{OUTPUT_PATH}contract-matrix.md`
2. Update `{OUTPUT_PATH}session/current.md`: set `doc-reader: done`

**If CONFLICTS found (⚠️ rows exist):**

```
⚠️  Contract Conflicts Ditemukan

Endpoint berikut ada di WP dan EXT dengan format berbeda:
[tampilkan baris ⚠️ dari contract-matrix.md]

Untuk setiap conflict, pilih:
  A. Pakai format WP
  B. Pakai format EXT
  C. Pertahankan keduanya (endpoint terpisah per scope)
  D. Custom mapping
```

Use the Write tool to create `{OUTPUT_PATH}conflict-resolutions.md` with user choices.

**If NO conflicts:**

Use the Write tool to create `{OUTPUT_PATH}conflict-resolutions.md`:
```markdown
# Conflict Resolutions — {SERVICE_NAME} / {FEATURE_NAME}

No conflicts found. All WP and EXT contracts are compatible.
```

---

## Step 9c: Spawn brainstormer

### Pre-spawn: Read output files

1. Read `{OUTPUT_PATH}doc-reader-summary.md` → DOC_READER_SUMMARY
2. Read `{OUTPUT_PATH}contract-matrix.md` → CONTRACT_MATRIX
3. Read `{OUTPUT_PATH}conflict-resolutions.md` → CONFLICT_RESOLUTIONS

```
Agent({
  subagent_type: "brainstormer",
  name: "brainstormer",
  team_name: "{TEAM_NAME}",
  prompt: `
SERVICE_NAME    = "{SERVICE_NAME}"
FEATURE_NAME    = "{FEATURE_NAME}"
BASE_PATH       = "{BASE_PATH}"
BOILERPLATE     = "{BOILERPLATE}"
PLAN_PATH       = "{PLAN_PATH}"
OUTPUT_PATH     = "{OUTPUT_PATH}"
EPIC            = "{EPIC}"
MODE            = "extend-existing"

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
Run Socratic brainstorming for adding {FEATURE_NAME} to the existing {SERVICE_NAME} service.

## HARD CONSTRAINTS — Non-negotiable:
1. JSON response field names MUST match legacy exactly per contract matrix.
2. All queries on shared tables MUST filter by app_origin.
3. No GORM AutoMigrate in production code. SQL files are source of truth.
4. Two channels: REST API + RabbitMQ RPC → same service layer.
5. WP clients get WP-format responses. EXT clients get EXT-format responses.
6. EXISTING CODE MUST NOT CHANGE — only Extend or Create. No refactoring of already-implemented entities.

## Sections to brainstorm (scope: {FEATURE_NAME} only):
- Do new entities follow the same route/app_origin/DTO pattern already established?
- Are there new shared components needed (e.g., new middleware, new pkg utility)?
- Are there shared tables between new and existing entities? (app_origin filter impact)
- Any design decisions that deviate from the existing service pattern? (flag and justify)

## Key difference from fresh planning:
The existing {BASE_PATH} already has a working pattern. Prefer consistency over novelty.
Only propose alternatives if the existing pattern cannot handle the new feature.
  `
})
```

**Wait for brainstormer to complete.**

After brainstormer completes, update `{OUTPUT_PATH}session/current.md`: set `brainstormer: done`.

---

## Step 9d: Spawn planner

### Pre-spawn: Read output files

1. Read `{OUTPUT_PATH}design-decisions.md` → DESIGN_DECISIONS
2. Read `{OUTPUT_PATH}contract-matrix.md` → CONTRACT_MATRIX
3. Read `{OUTPUT_PATH}entities.md` → ENTITIES

```
Agent({
  subagent_type: "planner",
  name: "planner",
  team_name: "{TEAM_NAME}",
  prompt: `
SERVICE_NAME    = "{SERVICE_NAME}"
FEATURE_NAME    = "{FEATURE_NAME}"
BASE_PATH       = "{BASE_PATH}"
BOILERPLATE     = "{BOILERPLATE}"
PLAN_PATH       = "{PLAN_PATH}"
OUTPUT_PATH     = "{OUTPUT_PATH}"
EPIC            = "{EPIC}"
MODE            = "extend-existing"

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
Produce a bite-sized implementation plan for the {FEATURE_NAME} feature in {SERVICE_NAME}.
Each task = 2-5 minutes, TDD per step, actual code, exact file paths. No placeholders.
Write to {OUTPUT_PATH}implementation-plan.md.

## CRITICAL — Mode is extend-existing:
- entities.md marks each component as Reuse / Extend / Create
- REUSE: DO NOT include these in the plan at all. They are already done.
- EXTEND: Include ONLY the new parts (new method, new field, new route entry). Never rewrite the whole file.
- CREATE: Full implementation — entity → DTO → repo interface → repo impl → service interface → service impl → handler → routes

## Extend examples:
- Config: only add the new env vars, not the whole config struct
- Routes: only add new route registrations to the existing routes.go
- main.go: only add new DI wiring lines, not a full rewrite

## Per-task format:
### Task N: [title]
**Entity/Component:** {name}
**Status:** Extend | Create  (never Reuse — those are skipped)
**File:** exact/path/to/file.go
**Test first:** write this failing test → verify it fails → implement → verify it passes
**Code:** [actual code, not pseudocode]
**Commit:** feat({SERVICE_NAME}): [description]
  `
})
```

**Wait for planner to complete.**

After planner completes, update `{OUTPUT_PATH}session/current.md`: set `planner: done`.

---

## Step 9e: Spawn task-breaker

### Pre-spawn: Read output file

1. Read `{OUTPUT_PATH}implementation-plan.md` → IMPLEMENTATION_PLAN

```
Agent({
  subagent_type: "task-breaker",
  name: "task-breaker",
  team_name: "{TEAM_NAME}",
  prompt: `
SERVICE_NAME = "{SERVICE_NAME}"
FEATURE_NAME = "{FEATURE_NAME}"
BASE_PATH    = "{BASE_PATH}"
PLAN_PATH    = "{PLAN_PATH}"
OUTPUT_PATH  = "{OUTPUT_PATH}"
EPIC         = "{EPIC}"

--- implementation-plan.md ---
{IMPLEMENTATION_PLAN}
--- end ---

## Your Task
Generate two task breakdown files for the {FEATURE_NAME} feature addition.

### 1. Superpowers Tasks → {OUTPUT_PATH}tasks-superpowers.md
Group tasks by entity/component. Mark:
  - ✅ PARALLEL: tasks for different entities
  - 🔒 SEQUENTIAL: tasks within same entity
  - ⚡ EXTEND: partial file changes (add to existing file, not full rewrite)

### 2. Taiga-ready Tasks → {OUTPUT_PATH}tasks-taiga.md
Format each task as a Taiga user story with acceptance criteria.
Add tag "extend-existing" to all Extend tasks so PM knows these are additive.
  `
})
```

**Wait for task-breaker to complete.**

After task-breaker completes, update `{OUTPUT_PATH}session/current.md`:
- Set `task-breaker: done`
- Set `STATUS: waiting-review`
- Read `{OUTPUT_PATH}tasks-superpowers.md` and count total tasks → set `TOTAL_TASKS: N`

---

## Step 10: STOP — User Review

```
╔════════════════════════════════════════════════════════════╗
║  Planning Selesai — {SERVICE_NAME} / {FEATURE_NAME} ({EPIC})║
╠════════════════════════════════════════════════════════════╣
║  Output files:                                             ║
║                                                            ║
║  {OUTPUT_PATH}contract-matrix.md      (doc-reader)         ║
║  {OUTPUT_PATH}entities.md             (doc-reader)         ║
║  {OUTPUT_PATH}conflict-resolutions.md (conflict gate)      ║
║  {OUTPUT_PATH}design-decisions.md     (brainstormer)       ║
║  {OUTPUT_PATH}implementation-plan.md  (planner)            ║
║  {OUTPUT_PATH}tasks-superpowers.md    (task-breaker)        ║
║  {OUTPUT_PATH}tasks-taiga.md          (task-breaker)        ║
╠════════════════════════════════════════════════════════════╣
║  STOP — Review output sebelum lanjut.                      ║
║                                                            ║
║  Cek terutama:                                             ║
║  1. entities.md — Reuse/Extend/Create sudah benar?         ║
║  2. implementation-plan.md — tidak ada task untuk Reuse?   ║
║  3. Extend tasks — hanya tambahan, bukan rewrite penuh?    ║
║  4. contract-matrix.md — backward compat terjaga?          ║
║                                                            ║
║  Jika sudah benar, ketik "lanjut" untuk Git Worktree.      ║
╚════════════════════════════════════════════════════════════╝
```

Do NOT proceed until user explicitly says "lanjut".

---

## When User Says "lanjut": Git Worktree Setup

### Create worktree

```bash
git worktree add ".claude/worktrees/{SERVICE_NAME}-{EPIC}" -b feat/{SERVICE_NAME}-{EPIC}
```

If worktree already exists:
```bash
git worktree list
```
Confirm branch name and path, then continue.

### Update session state

Use the Write tool to update `{OUTPUT_PATH}session/current.md`:
- Set `PHASE: development`
- Set `WORKTREE_PATH: .claude/worktrees/{SERVICE_NAME}-{EPIC}`
- Set `BRANCH: feat/{SERVICE_NAME}-{EPIC}`

### Hand off to development

```
╔══════════════════════════════════════════════════════════════╗
║  Worktree siap.                                              ║
║                                                              ║
║  Branch:   feat/{SERVICE_NAME}-{EPIC}                        ║
║  Path:     .claude/worktrees/{SERVICE_NAME}-{EPIC}           ║
║                                                              ║
║  Jalankan /continue untuk mulai development loop.            ║
║  /continue akan baca tasks-superpowers.md dan dispatch       ║
║  implementer per task. Extend tasks hanya ubah bagian baru.  ║
╚══════════════════════════════════════════════════════════════╝
```

**STOP.** Development loop dijalankan via `/continue`.
