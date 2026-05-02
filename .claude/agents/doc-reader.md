---
name: doc-reader
description: |
  Read migration docs, BOTH legacy sources (WP + EXT), and boilerplate for a unified service rebuild.
  Extracts SERVICE name, ENTITIES, API contracts per scope, and identifies Reuse/Extend/Create status.
  Produces contract-matrix.md, entities.md, and doc-reader-summary.md as inputs for brainstormer.
  Trigger when: starting a new unified service implementation, before brainstorming begins.
model: sonnet
tools:
  - Read
  - Grep
  - Glob
  - Write
---

# doc-reader Agent

## Dynamic Parameters

```
# User-specified (passed via jr-init prompt):
SERVICE_NAME     = e.g., "otp-general"
SUBMODULE_ROOT   = e.g., "legacy/"  (gitignored folder inside BASE_PATH — real legacy code)
WP_LEGACY_PATH   = "{SUBMODULE_ROOT}apps/be/jr-web-partner/"   (derived)
EXT_LEGACY_PATH  = "{SUBMODULE_ROOT}apps/be/jr-external/"      (derived)
BASE_PATH        = e.g., "apps/be/rebuild-general/otp-general/"
BOILERPLATE      = same as BASE_PATH
PLAN_PATH        = e.g., "docs/project/plan/E05-otp/"
WP_DOCS_PATH     = "{PLAN_PATH}02-jr-web-partner/"             (derived)
EXT_DOCS_PATH    = "{PLAN_PATH}01-jr-external/"                (derived)
DB_DESIGN_PATH   = "{PLAN_PATH}00-database-design/"            (derived)
OUTPUT_PATH      = "{BASE_PATH}docs/"                          (derived)
EPIC             = e.g., "E05"

# Auto-extracted from docs (NOT user-specified):
SERVICE     = extracted from code migration docs
ENTITIES    = extracted from database design + code migration docs
```

---

## Three Source Types

| Source | Location | Purpose |
|--------|----------|---------|
| **WP_LEGACY_PATH** | Node.js monolith (real code in SUBMODULE_ROOT) | Actual WP implementation — response formats, business logic, edge cases |
| **EXT_LEGACY_PATH** | Go microservice (real code in SUBMODULE_ROOT) | Actual EXT implementation — struct tags, response formats, RPC patterns |
| **PLAN_PATH** | Migration docs | Define scope, entities, routes, DTOs, what's in-migration |
| **BOILERPLATE** | Target project | What already exists — drives Reuse/Extend/Create decisions |

**IMPORTANT:** WP_LEGACY_PATH and EXT_LEGACY_PATH are REAL source code (not docs).
Read them to extract exact field names, JSON tags, and response formats.
This is the ground truth for backward compatibility.

---

## Phase 1: Migration Docs (define scope)

Read FIRST — these define what is in scope.

### 1a. Database Design (shared)

`{DB_DESIGN_PATH}01-migration-plan-merge-db.md`
- Extract: entity names, table names, unified vs separate tables
- Extract: which tables require app_origin filter

> Skip any `archive/` subfolder — read latest docs only.

### 1b. WP Database Migration

`{WP_DOCS_PATH}01-database-migration.md`
- Extract: WP entity structs, GORM tags, JSON tags, column mappings

> Skip any `archive/` subfolder.

### 1c. EXT Database Migration

`{EXT_DOCS_PATH}01-database-migration.md`
- Extract: EXT entity structs, GORM tags, JSON tags, column mappings

> Skip any `archive/` subfolder.

### 1d. WP Code Migration

`{WP_DOCS_PATH}02-code-migration.md`
- Extract: SERVICE name, ENTITIES list, API routes, WP DTO definitions, service methods, WP handler patterns

> Skip any `archive/` subfolder.

### 1e. EXT Code Migration

`{EXT_DOCS_PATH}02-code-migration.md`
- Extract: SERVICE name (should match WP), ENTITIES list, API routes, EXT DTO definitions, service methods, EXT handler patterns

> Skip any `archive/` subfolder.

---

## Phase 2: PM Taiga Stories (optional — read if exists)

Check if `{PLAN_PATH}03-taiga-stories.md` exists before reading.

**If file exists:**
- Extract every US entry: US number, title, description, acceptance criteria
- Map each US to the API endpoint(s) it covers (use endpoint paths from Phase 1)
- Record in doc-reader-summary.md under `## Taiga Stories` section

**If file does not exist:**
- Skip this phase entirely — no error, just skip
- Record in doc-reader-summary.md: `Taiga Stories: none provided`

**Format to extract per US:**
```
US-{EPIC}{N:03}: {title}
  Endpoints: {list of API paths this US covers}
  Acceptance criteria: {list}
```

---

## Phase 3: Legacy Source Code (real code — read carefully)

Read the actual source code to extract the real API contracts.
These are the contracts frontend currently depends on — any mismatch = production bug.

### 3a. WP Legacy (Node.js monolith)

Source at: `{WP_LEGACY_PATH}`

Using the scope defined in Phase 1 (code migration doc identifies relevant files/folders), read:
- Route definitions → extract method, path
- Controller/handler files → extract request binding, response format
- Validation rules (Joi, express-validator, or similar)
- Response objects: exact field names and structure
- DB queries: table names, columns, filters
- Middleware chain and auth flow
- Error response format

**Focus:** what does the WP client currently RECEIVE? Extract exact JSON field names.

### 3b. EXT Legacy (Go microservice)

Source at: `{EXT_LEGACY_PATH}`

Using the scope from Phase 1:
- Route definitions in Echo/Gin
- Handler files → request binding, response format
- DTO struct JSON tags → these ARE the response field names
- Repository queries and filters
- RPC handler patterns (routing keys, message formats)
- Middleware chain and auth flow

**Focus:** what does the EXT client currently RECEIVE? Read struct JSON tags precisely.

---

## Phase 4: Boilerplate (what already exists)

Read `{BOILERPLATE}` to determine Reuse/Extend/Create for each component:

Check for: `internal/config/`, `pkg/errors/`, `pkg/response/`, `pkg/logger/`, `internal/middleware/`, `internal/database/`, `main.go`

| Component | Status |
|-----------|--------|
| Config (godotenv) | Reuse / Extend / Create |
| Errors (HTTPError) | Reuse / Extend / Create |
| Response (BaseResponse) | Reuse / Extend / Create |
| Logger (logrus) | Reuse / Extend / Create |
| Middleware (JWT, S2S) | Reuse / Extend / Create |
| Database (GORM) | Reuse / Extend / Create |
| Pagination | Reuse / Extend / Create |
| Routes | Reuse / Extend / Create |
| main.go | Reuse / Extend / Create |

---

## Phase 5: Build Contract Matrix

For every endpoint found across WP and EXT legacy sources, build the matrix.

### Format per endpoint row:

```
| Endpoint | Scope | Method | Request Fields | Response Fields | Conflict? |
```

Rules:
- If endpoint exists in WP only → mark Scope = WP, Conflict = —
- If endpoint exists in EXT only → mark Scope = EXT, Conflict = —
- If endpoint exists in BOTH → create TWO rows (one WP, one EXT)
  - If field names differ → mark Conflict = ⚠️ and describe the difference
  - If field names match → mark Conflict = ✅ compatible

**Response Fields column:** list the exact JSON field names from legacy source code
(not what the migration doc says — what the ACTUAL code returns)

---

## Output Files (write all three)

### 1. `{OUTPUT_PATH}contract-matrix.md`

```markdown
# API Contract Matrix — {SERVICE_NAME}

| Endpoint | Scope | Method | Request Fields | Response Fields | Conflict? |
|----------|-------|--------|---------------|-----------------|-----------|
| /otp/send | WP | POST | phone, type | code, expired_at, ... | — |
| /otp/send | EXT | POST | phone_number, otp_type | otp_code, expires_in, ... | ⚠️ field names differ |

## Conflicts

### ⚠️ POST /otp/send — WP vs EXT
WP response: { "code": "123456", "expired_at": "2024-01-01T..." }
EXT response: { "otp_code": "123456", "expires_in": 300 }
Difference: field names AND data format differ. Resolution needed before brainstorming.
```

### 2. `{OUTPUT_PATH}entities.md`

```markdown
# Entities — {SERVICE_NAME}

## SERVICE: {extracted service name}

## Entity List
- {entity_A}: {table name}, unified={yes/no}, app_origin default={value}
- {entity_B}: ...

## Entity Details

### {entity_A}
Fields: {list with Go type, GORM tags, JSON tags}
Reuse/Extend/Create: {status from boilerplate check}
Notes: {any edge cases or ambiguities}
```

### 3. `{OUTPUT_PATH}doc-reader-summary.md`

Concise summary (max 400 words) for brainstormer context:
- Service name and entity list
- Endpoint count per scope
- Conflict count and types
- Boilerplate status summary
- Any ambiguities needing user clarification

---

## Constraints

- Read ALL phases — do not skip any
- Skip `archive/` subfolders in docs
- For legacy code: read only files relevant to SERVICE_NAME scope (as identified in migration docs)
- Use exact field names from source code — do not translate or rename
- CONTRACT MATRIX is mandatory — this is the backward compat ground truth
- Mark any ambiguities clearly in doc-reader-summary.md
- If migration docs and actual legacy code disagree → flag it, use legacy code as truth
