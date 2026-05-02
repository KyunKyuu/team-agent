---
name: qa-compat
description: |
  Backward compatibility verification against the API contract matrix.
  Verifies JSON response tags match legacy WP and EXT contracts exactly.
  Also checks for unexpected extra fields that could break strict-mode clients.
  Trigger when: qa-verify has passed.
model: sonnet
tools:
  - Read
  - Bash
  - Grep
  - Glob
  - Write
---

# qa-compat Agent

## Dynamic Parameters

```
SERVICE_NAME = e.g., "otp-general"
BASE_PATH    = e.g., "apps/be/rebuild-general/otp-general/"
OUTPUT_PATH  = "{BASE_PATH}docs/"
REPORT_PATH  = "docs/project/reports/{EPIC}-{SERVICE_NAME}/"
EPIC         = e.g., "E05"

# Key input:
contract-matrix.md      ← from doc-reader, at {OUTPUT_PATH}
conflict-resolutions.md ← from conflict gate, at {OUTPUT_PATH}
```

## Task

Verify the implemented service preserves backward compatibility with both WP and EXT legacy API contracts.

**Critical rule: frontend clients cannot adapt. Any mismatch is a production bug.**

---

## Step 1: Load Contract Matrix

Use the Read tool to read `{OUTPUT_PATH}contract-matrix.md` and `{OUTPUT_PATH}conflict-resolutions.md`.

Build a check list: for each endpoint, what are the EXPECTED response fields per scope?

---

## Step 2: Verify Response DTO JSON Tags (Expected Fields)

For each endpoint in the contract matrix:

### WP scope
1. Find the WP response DTO struct (in `internal/dto/`)
2. For each field in the WP contract:
   - Contract field name → check `json:"exact_name"` tag in Go struct
3. Verify types are compatible (string→string, int→int, bool→bool, nullable→pointer)

### EXT scope
Same as WP but for EXT response DTO struct.

```bash
grep -rn "json:\"" {BASE_PATH}internal/dto/ --include="*.go"
```

---

## Step 3: Verify No Extra Fields (Unexpected Fields)

Check that response DTOs do NOT have extra fields beyond what the legacy contract defines.

Extra fields can break clients that use strict JSON deserialization (reject unknown keys).

For each response DTO struct:
1. List all exported fields with `json:` tags
2. Compare against contract-matrix.md for that scope
3. Flag any field that is NOT in the legacy contract

```bash
# Get all json tags from response DTOs
grep -rn "json:\"" {BASE_PATH}internal/dto/ --include="*.go" | grep -i "response\|resp\|res"
```

For each flagged extra field, classify:
- ⚠️ WARN: field is additive and unlikely to break clients (assess case-by-case)
- ❌ FAIL: field changes the structure in a way that could break strict parsers

---

## Step 4: Verify Request Field Names

For each endpoint, verify the request DTO accepts the same field names as legacy:

```bash
grep -rn "json:\"" {BASE_PATH}internal/dto/ --include="*.go" | grep -i "request\|req\|input"
```

Field names in request DTOs must match what legacy clients send.

---

## Step 5: Verify Endpoint Paths

```bash
grep -rn "GET\|POST\|PUT\|PATCH\|DELETE" {BASE_PATH}internal/routes/ --include="*.go"
```

Compare registered routes against contract-matrix.md endpoint paths.
Any missing endpoint = regression.

---

## Step 6: Verify Error Response Format

```bash
grep -rn "HTTPError\|BaseResponse" {BASE_PATH}internal/handler/ --include="*.go" | head -30
```

WP and EXT may have different error formats — verify the handler returns the right one per scope.

---

## Step 7: Write Report

Use the Write tool to save to `{REPORT_PATH}qa-compat-report.md`:

```markdown
# QA Compat Report — {SERVICE_NAME} ({EPIC})
Date: {date}

## Summary

| Endpoint | Scope | Expected Fields | Extra Fields | Request Fields | Route | Status |
|----------|-------|----------------|-------------|----------------|-------|--------|
| POST /otp/send | WP | ✅ | ✅ none | ✅ | ✅ | ✅ PASS |
| POST /otp/send | EXT | ❌ mismatch | ⚠️ 1 extra | ✅ | ✅ | ❌ FAIL |

## Overall: ✅ PASS / ❌ FAIL — Proceed to qa-report / Fix issues first

## Issues Detail

### Missing / Mismatched Fields — {Endpoint} ({Scope})

| Contract expects | Go struct has | Status |
|-----------------|--------------|--------|
| `expired_at` | `json:"expiredAt"` | ❌ wrong tag |

**Fix:** `{file}:{line}` — change `json:"expiredAt"` to `json:"expired_at"`

### Extra Fields — {Endpoint} ({Scope})

| Extra field | json tag | Risk |
|-------------|----------|------|
| `request_id` | `json:"request_id"` | ⚠️ additive, low risk |

**Recommendation:** Remove or confirm with frontend team before shipping.
```
