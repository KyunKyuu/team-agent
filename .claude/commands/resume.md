---
description: |
  Add a new feature/epic to an EXISTING service — any language.
  Same planning pipeline as /init but targets a project that already has code.
  Existing entities are Reuse, new ones are Create, shared components may be Extend.
  Trigger when: user says "resume", "tambahin fitur", "epic baru di service yang sama".
---

# Resume — Add Feature to Existing Service

Adds new entities/endpoints from a new epic to a service that already exists.
**Hard constraint: existing code MUST NOT be modified unless explicitly Extend.**

---

## Step 1: Ask PROJECT_NAME

```
Question: "Nama service yang sudah ada?"
Header: "Project Name"
Free text.
```

---

## Step 2: Ask FEATURE_NAME

```
Question: "Nama fitur/entity baru yang akan ditambahkan?"
Header: "Feature Name"
Contoh: blacklist, audit-log, resend-limit, user-preference
Free text.
```

---

## Step 3: Ask LANGUAGE

```
Question: "Bahasa/stack project?"
Header: "Language"
Options: go | node | python | php | other
```

---

## Step 4: Ask SOURCE_COUNT

```
Question: "Ada berapa legacy source untuk fitur baru ini?"
Header: "Legacy Sources"
Options:
  - "0 — tidak ada legacy, desain baru"
  - "1 — rebuild dari satu legacy source"
  - "2 — merge dua legacy source"
```

---

## Step 5: Legacy Source Details (skip if SOURCE_COUNT = 0)

Sama seperti /init Step 4 — tanyakan SOURCE_A dan SOURCE_B details sesuai SOURCE_COUNT.

Step 5b: Verify source folders exist.

---

## Step 6: Ask BASE_PATH

```
Question: "Path ke project yang sudah ada?"
Header: "Base Path"
Options:
  - "apps/{LANGUAGE}/{PROJECT_NAME}/" (pre-filled)
  - "Other"
```

### Step 6b: Verify Existing Project

```bash
ls "{BASE_PATH}" 2>/dev/null && echo "OK" || echo "MISSING"
```

Jika MISSING → error, minta perbaiki path atau gunakan /init untuk project baru.

---

## Step 7: Ask PLAN_PATH

```
Question: "Path ke epic plan folder untuk fitur baru ini?"
Header: "Plan Path"
Options:
  - "docs/project/plan/E{XX}-{FEATURE_NAME}/" (pre-filled)
  - "Other"
```

---

## Step 8: Derive All Paths

```
EPIC         = extract from PLAN_PATH (segment E + angka)
               jika tidak ada → tanya "Epic ID?"

BOILERPLATE  = BASE_PATH
OUTPUT_PATH  = "{BASE_PATH}docs/{EPIC}-{FEATURE_NAME}/"
TAIGA_STORIES = "{PLAN_PATH}03-taiga-stories.md"
REPORT_PATH  = "docs/project/reports/{EPIC}-{FEATURE_NAME}/"
TEAM_NAME    = "planning-{PROJECT_NAME}-{EPIC}-{FEATURE_NAME}"

SOURCE_A_DOCS_PATH = "{PLAN_PATH}01-{SOURCE_A_NAME}/"
SOURCE_B_DOCS_PATH = "{PLAN_PATH}02-{SOURCE_B_NAME}/"  ← jika SOURCE_COUNT = 2
DB_DESIGN_PATH     = "{PLAN_PATH}00-database-design/"
```

> OUTPUT_PATH pakai subfolder per epic — tidak menimpa docs epic sebelumnya.

---

## Step 9: Confirm Parameters

```
┌─────────────────────────────────────────────────────────────┐
│  Resume — Planning Team Parameter Summary                   │
├─────────────────────────────────────────────────────────────┤
│  PROJECT_NAME:  {PROJECT_NAME}  (existing service)          │
│  FEATURE_NAME:  {FEATURE_NAME}  (new feature)               │
│  LANGUAGE:      {LANGUAGE}                                  │
│  EPIC:          {EPIC}  (derived dari PLAN_PATH)            │
├─────────────────────────────────────────────────────────────┤
│  SOURCE_COUNT:  {SOURCE_COUNT}                              │
│  SOURCE_A:      {SOURCE_A_NAME} @ {SOURCE_A_PATH}  ✅       │
│  SOURCE_B:      {SOURCE_B_NAME} @ {SOURCE_B_PATH}  ✅       │ ← jika ada
├─────────────────────────────────────────────────────────────┤
│  BASE_PATH:     {BASE_PATH}  ✅ existing project             │
│  OUTPUT_PATH:   {OUTPUT_PATH}  (per-epic subfolder)         │
│  TAIGA_STORIES: {TAIGA_STORIES}  (dicek saat doc-reader)    │
├─────────────────────────────────────────────────────────────┤
│  Mode: ADD TO EXISTING — existing code tidak diubah         │
└─────────────────────────────────────────────────────────────┘
```

```
Question: "Parameter sudah benar? Lanjutkan?"
Options: "Ya, lanjutkan" | "Tidak, ada yang perlu diperbaiki"
```

### Initialize Session State

Write `{OUTPUT_PATH}session/current.md`:

```markdown
# Session State — {PROJECT_NAME} / {FEATURE_NAME} ({EPIC})

## Active Project
PROJECT_NAME:  {PROJECT_NAME}
FEATURE_NAME:  {FEATURE_NAME}
LANGUAGE:      {LANGUAGE}
EPIC:          {EPIC}
MODE:          extend-existing
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
...
```

---

## Step 10a: Spawn doc-reader

Same as /init Step 9a, plus add `MODE = "extend-existing"` and instruct doc-reader to:
- Scan existing `{BASE_PATH}` first — mark all existing components as Reuse
- Focus new feature extraction on SOURCE_A / SOURCE_B for `{FEATURE_NAME}` scope only

```
Agent({
  subagent_type: "doc-reader",
  prompt: `
PROJECT_NAME   = "{PROJECT_NAME}"
FEATURE_NAME   = "{FEATURE_NAME}"
LANGUAGE       = "{LANGUAGE}"
MODE           = "extend-existing"
...same params as /init...

IMPORTANT: MODE is extend-existing.
Scan {BASE_PATH} first → all existing components = Reuse.
Focus: only new feature ({FEATURE_NAME}) endpoints and entities from legacy sources.
  `
})
```

Wait. Update session: `doc-reader: done`.

---

## Steps 10b–10e: Contract Gate → brainstormer → planner → task-breaker

Same as /init Steps 9b–9e, with these additions:

**brainstormer extra constraint:**
```
EXISTING CODE MUST NOT CHANGE — only Extend or Create. No refactoring of existing entities.
Follow the existing patterns already in {BASE_PATH} — prefer consistency over novelty.
```

**planner extra instruction:**
```
MODE = extend-existing
REUSE: skip entirely — do not include in plan
EXTEND: only the new parts (new method, new field, new route entry)
CREATE: full implementation
```

**task-breaker extra:**
- Tag Extend tasks with `⚡ EXTEND` in superpowers, `extend-existing` tag in Taiga

---

## Step 11: STOP — User Review

```
╔═══════════════════════════════════════════════════════════════╗
║  Planning Selesai — {PROJECT_NAME}/{FEATURE_NAME} ({EPIC})    ║
╠═══════════════════════════════════════════════════════════════╣
║  Cek: entities.md — Reuse/Extend/Create sudah benar?          ║
║       implementation-plan.md — tidak ada task untuk Reuse?    ║
║       Extend tasks — hanya tambahan, bukan rewrite penuh?     ║
╠═══════════════════════════════════════════════════════════════╣
║  Ketik "lanjut" untuk Git Worktree.                           ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## When User Says "lanjut"

```bash
git worktree add ".claude/worktrees/{PROJECT_NAME}-{EPIC}" -b feat/{PROJECT_NAME}-{EPIC}
```

Update session: `PHASE: development`, `WORKTREE_PATH`, `BRANCH`.
Print: `Jalankan /continue untuk mulai development loop.`
