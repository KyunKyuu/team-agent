---
description: |
  Launch the Planning Team for a new service — any language, 0–2 legacy sources.
  0 sources = fresh project. 1 source = single rebuild. 2 sources = merge two into one.
  Trigger when: user says "init", "planning baru", "buat service baru", or runs /init.
---

# Init — New Service Planning Team

Supports any language and any number of legacy sources (0, 1, or 2).
**Hard constraint: response contracts MUST NOT change — clients cannot adapt.**

Flow: gather params → doc-reader → conflict gate (if 2 sources) → brainstormer → planner → task-breaker → STOP.

---

## Step 1: Ask PROJECT_NAME

```
Question: "Nama service/project yang akan dibuat?"
Header: "Project Name"
Free text. Contoh: otp-service, user-management, payment-gateway
```

---

## Step 2: Ask LANGUAGE

```
Question: "Bahasa/stack yang dipakai?"
Header: "Language"
Options:
  - "go"
  - "node"
  - "python"
  - "php"
  - "other (specify)"
MultiSelect: false
```

If "other", ask user to type. Simpan sebagai LANGUAGE. Ini menentukan skill set yang dipakai.

---

## Step 3: Ask SOURCE_COUNT

Berapa banyak legacy source yang akan digabungkan atau di-rebuild?

```
Question: "Ada berapa legacy source?"
Header: "Legacy Sources"
Options:
  - "0 — fresh project, tidak ada legacy"
  - "1 — rebuild dari satu legacy source"
  - "2 — merge dua legacy source menjadi satu"
MultiSelect: false
```

---

## Step 4: Ask Legacy Source Details (skip if SOURCE_COUNT = 0)

### Jika SOURCE_COUNT = 1:

```
Question: "Nama legacy source? (label bebas, contoh: monolith, legacy-api)"
Header: "Source Name"
Free text → simpan sebagai SOURCE_A_NAME

Question: "Path ke folder legacy source? (gitignored)"
Header: "Source A Path"
Options:
  - "legacy/" (default)
  - "Other"
→ simpan sebagai SOURCE_A_PATH

Question: "Stack/bahasa legacy source tersebut?"
Header: "Source A Language"
Options: go | node | python | other
→ simpan sebagai SOURCE_A_LANG
```

### Jika SOURCE_COUNT = 2:

```
Source A:
  Question: "Nama legacy source pertama? (contoh: web-partner, monolith-wp)"
  Header: "Source A Name"
  Free text → SOURCE_A_NAME

  Question: "Path ke folder source A? (gitignored)"
  Header: "Source A Path"
  Options: "legacy/source-a/" | "Other" → SOURCE_A_PATH

  Question: "Stack/bahasa source A?"
  Options: go | node | python | other → SOURCE_A_LANG

Source B:
  Question: "Nama legacy source kedua? (contoh: external, monolith-ext)"
  Header: "Source B Name"
  Free text → SOURCE_B_NAME

  Question: "Path ke folder source B? (gitignored)"
  Header: "Source B Path"
  Options: "legacy/source-b/" | "Other" → SOURCE_B_PATH

  Question: "Stack/bahasa source B?"
  Options: go | node | python | other → SOURCE_B_LANG

Question: "Ada partition key / field pemisah data antar source? (contoh: app_origin, tenant_id)"
Header: "Partition Key"
Options:
  - "Tidak ada"
  - "Ada → ketik nama field-nya"
→ simpan sebagai PARTITION_KEY (atau "none")
```

### Step 4b: Verify Source Folders (skip if SOURCE_COUNT = 0)

```bash
ls "{SOURCE_A_PATH}" 2>/dev/null && echo "OK" || echo "MISSING A"
# jika SOURCE_COUNT = 2:
ls "{SOURCE_B_PATH}" 2>/dev/null && echo "OK" || echo "MISSING B"
```

Jika MISSING → tampilkan error dan minta user perbaiki path.

---

## Step 5: Ask BASE_PATH

```
Question: "Path ke target project (tempat kode baru akan ditulis)?"
Header: "Base Path"
Options:
  - "apps/{LANGUAGE}/{PROJECT_NAME}/" (pre-filled)
  - "Other"
```

---

## Step 6: Ask PLAN_PATH

```
Question: "Path ke epic plan folder (berisi migration docs, taiga stories)?"
Header: "Plan Path"
Options:
  - "docs/project/plan/E01-{PROJECT_NAME}/" (pre-filled)
  - "Other"
```

---

## Step 7: Derive All Paths

```
EPIC         = extract from PLAN_PATH — segment diawali E diikuti angka
               contoh: "docs/project/plan/E05-otp/" → EPIC = "E05"
               jika tidak ditemukan → tanya "Epic ID?"

BOILERPLATE  = BASE_PATH
OUTPUT_PATH  = "{BASE_PATH}docs/"
TAIGA_STORIES = "{PLAN_PATH}03-taiga-stories.md"
REPORT_PATH  = "docs/project/reports/{EPIC}-{PROJECT_NAME}/"
TEAM_NAME    = "planning-{PROJECT_NAME}-{EPIC}"

# Jika SOURCE_COUNT >= 1:
SOURCE_A_DOCS_PATH = "{PLAN_PATH}01-{SOURCE_A_NAME}/"

# Jika SOURCE_COUNT = 2:
SOURCE_B_DOCS_PATH = "{PLAN_PATH}02-{SOURCE_B_NAME}/"
DB_DESIGN_PATH     = "{PLAN_PATH}00-database-design/"
```

---

## Step 8: Confirm Parameters

```
┌────────────────────────────────────────────────────────┐
│  Init — Planning Team Parameter Summary                │
├────────────────────────────────────────────────────────┤
│  PROJECT_NAME:  {PROJECT_NAME}                         │
│  LANGUAGE:      {LANGUAGE}                             │
│  EPIC:          {EPIC}  (derived dari PLAN_PATH)       │
├────────────────────────────────────────────────────────┤
│  SOURCE_COUNT:  {SOURCE_COUNT}                         │
│  SOURCE_A:      {SOURCE_A_NAME} ({SOURCE_A_LANG})      │
│                 {SOURCE_A_PATH}  ✅ verified            │
│  SOURCE_B:      {SOURCE_B_NAME} ({SOURCE_B_LANG})      │ ← jika ada
│                 {SOURCE_B_PATH}  ✅ verified            │ ← jika ada
│  PARTITION_KEY: {PARTITION_KEY}                        │ ← jika ada
├────────────────────────────────────────────────────────┤
│  BASE_PATH:     {BASE_PATH}                            │
│  PLAN_PATH:     {PLAN_PATH}                            │
│  OUTPUT_PATH:   {OUTPUT_PATH}                          │
├────────────────────────────────────────────────────────┤
│  TAIGA_STORIES: {TAIGA_STORIES}                        │
│    → (dicek saat doc-reader jalan)                     │
└────────────────────────────────────────────────────────┘
```

```
Question: "Parameter sudah benar? Lanjutkan spawn doc-reader?"
Options: "Ya, lanjutkan" | "Tidak, ada yang perlu diperbaiki"
```

### Initialize Session State

Use Write tool → `{OUTPUT_PATH}session/current.md`:

```markdown
# Session State — {PROJECT_NAME} ({EPIC})

## Active Project
PROJECT_NAME:  {PROJECT_NAME}
LANGUAGE:      {LANGUAGE}
EPIC:          {EPIC}
SOURCE_COUNT:  {SOURCE_COUNT}
SOURCE_A_NAME: {SOURCE_A_NAME}
SOURCE_B_NAME: {SOURCE_B_NAME}
PARTITION_KEY: {PARTITION_KEY}
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

```
Agent({
  subagent_type: "doc-reader",
  name: "doc-reader",
  team_name: "{TEAM_NAME}",
  prompt: `
PROJECT_NAME   = "{PROJECT_NAME}"
LANGUAGE       = "{LANGUAGE}"
EPIC           = "{EPIC}"
SOURCE_COUNT   = {SOURCE_COUNT}
SOURCE_A_NAME  = "{SOURCE_A_NAME}"
SOURCE_A_PATH  = "{SOURCE_A_PATH}"
SOURCE_A_LANG  = "{SOURCE_A_LANG}"
SOURCE_B_NAME  = "{SOURCE_B_NAME}"
SOURCE_B_PATH  = "{SOURCE_B_PATH}"
SOURCE_B_LANG  = "{SOURCE_B_LANG}"
PARTITION_KEY  = "{PARTITION_KEY}"
BASE_PATH      = "{BASE_PATH}"
BOILERPLATE    = "{BOILERPLATE}"
PLAN_PATH      = "{PLAN_PATH}"
SOURCE_A_DOCS_PATH = "{SOURCE_A_DOCS_PATH}"
SOURCE_B_DOCS_PATH = "{SOURCE_B_DOCS_PATH}"
DB_DESIGN_PATH = "{DB_DESIGN_PATH}"
OUTPUT_PATH    = "{OUTPUT_PATH}"

Read legacy sources, migration docs, and boilerplate.
Extract API contracts per source scope.
Build contract matrix. Flag conflicts if SOURCE_COUNT = 2.
Write: contract-matrix.md, entities.md, doc-reader-summary.md
  `
})
```

Wait. Update session: `doc-reader: done`.

---

## Step 9b: Contract Conflict Gate (only if SOURCE_COUNT = 2)

Read `{OUTPUT_PATH}contract-matrix.md`. If SOURCE_COUNT < 2 → skip gate, write empty conflict-resolutions.md.

**If conflicts found (⚠️ rows):**
```
⚠️  Contract Conflicts Ditemukan

Endpoint berikut ada di {SOURCE_A_NAME} dan {SOURCE_B_NAME} dengan format berbeda:
[tampilkan baris ⚠️]

Untuk setiap conflict:
  A. Pakai format {SOURCE_A_NAME}
  B. Pakai format {SOURCE_B_NAME}
  C. Pertahankan keduanya (endpoint terpisah per scope)
  D. Custom mapping
```

Write `{OUTPUT_PATH}conflict-resolutions.md` dengan resolusi user.

**If no conflicts / SOURCE_COUNT < 2:**
Write `{OUTPUT_PATH}conflict-resolutions.md`: `No conflicts.`

---

## Step 9c: Spawn brainstormer

Pre-read: doc-reader-summary.md → DOC_READER_SUMMARY, contract-matrix.md → CONTRACT_MATRIX, conflict-resolutions.md → CONFLICT_RESOLUTIONS

```
Agent({
  subagent_type: "brainstormer",
  prompt: `
PROJECT_NAME   = "{PROJECT_NAME}"
LANGUAGE       = "{LANGUAGE}"
SOURCE_COUNT   = {SOURCE_COUNT}
SOURCE_A_NAME  = "{SOURCE_A_NAME}"
SOURCE_B_NAME  = "{SOURCE_B_NAME}"
PARTITION_KEY  = "{PARTITION_KEY}"
BASE_PATH      = "{BASE_PATH}"
OUTPUT_PATH    = "{OUTPUT_PATH}"
EPIC           = "{EPIC}"

--- doc-reader-summary.md ---
{DOC_READER_SUMMARY}
--- end ---

--- contract-matrix.md ---
{CONTRACT_MATRIX}
--- end ---

--- conflict-resolutions.md ---
{CONFLICT_RESOLUTIONS}
--- end ---

Run brainstorming for {PROJECT_NAME}.
Design: architecture, layer separation, DTO strategy, error handling.
If SOURCE_COUNT = 2: design scope separation and {PARTITION_KEY} injection strategy.
Write: {OUTPUT_PATH}design-decisions.md
  `
})
```

Wait. Update session: `brainstormer: done`.

---

## Step 9d: Spawn planner

Pre-read: design-decisions.md → DESIGN_DECISIONS, contract-matrix.md → CONTRACT_MATRIX, entities.md → ENTITIES

```
Agent({
  subagent_type: "planner",
  prompt: `
PROJECT_NAME = "{PROJECT_NAME}"
LANGUAGE     = "{LANGUAGE}"
BASE_PATH    = "{BASE_PATH}"
OUTPUT_PATH  = "{OUTPUT_PATH}"
EPIC         = "{EPIC}"
PARTITION_KEY = "{PARTITION_KEY}"

--- design-decisions.md ---
{DESIGN_DECISIONS}
--- end ---

--- contract-matrix.md ---
{CONTRACT_MATRIX}
--- end ---

--- entities.md ---
{ENTITIES}
--- end ---

Write implementation plan to {OUTPUT_PATH}implementation-plan.md.
Tasks: 2-5 min each, TDD per step, actual code, no placeholders.
Language: {LANGUAGE} — use idiomatic patterns for this language.
  `
})
```

Wait. Update session: `planner: done`.

---

## Step 9e: Spawn task-breaker

Pre-read: implementation-plan.md → IMPLEMENTATION_PLAN

```
Agent({
  subagent_type: "task-breaker",
  prompt: `
PROJECT_NAME = "{PROJECT_NAME}"
LANGUAGE     = "{LANGUAGE}"
BASE_PATH    = "{BASE_PATH}"
OUTPUT_PATH  = "{OUTPUT_PATH}"
EPIC         = "{EPIC}"

--- implementation-plan.md ---
{IMPLEMENTATION_PLAN}
--- end ---

Write tasks-superpowers.md and tasks-taiga.md.
  `
})
```

Wait. Update session: `task-breaker: done`, `STATUS: waiting-review`, count TOTAL_TASKS.

---

## Step 10: STOP — User Review

```
╔═══════════════════════════════════════════════════════════╗
║  Planning Selesai — {PROJECT_NAME} ({EPIC})               ║
╠═══════════════════════════════════════════════════════════╣
║  {OUTPUT_PATH}contract-matrix.md                          ║
║  {OUTPUT_PATH}entities.md                                 ║
║  {OUTPUT_PATH}conflict-resolutions.md                     ║
║  {OUTPUT_PATH}design-decisions.md                         ║
║  {OUTPUT_PATH}implementation-plan.md                      ║
║  {OUTPUT_PATH}tasks-superpowers.md                        ║
║  {OUTPUT_PATH}tasks-taiga.md                              ║
╠═══════════════════════════════════════════════════════════╣
║  Review lalu ketik "lanjut" untuk Git Worktree.           ║
╚═══════════════════════════════════════════════════════════╝
```

---

## When User Says "lanjut"

```bash
git worktree add ".claude/worktrees/{PROJECT_NAME}" -b feat/{PROJECT_NAME}
```

Update session: `PHASE: development`, `WORKTREE_PATH`, `BRANCH`.

Print: `Jalankan /continue untuk mulai development loop.`
