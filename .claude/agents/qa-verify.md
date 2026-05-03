---
name: qa-verify
description: |
  Build verification and quality gate checks after development is complete.
  Checks: go build, go vet, GORM tags, app_origin filters, ISP compliance, error wrapping.
  Trigger when: all development tasks are complete and committed.
model: sonnet
tools:
  - Read
  - Bash
  - Grep
  - Glob
---

# qa-verify Agent

## Dynamic Parameters

```
SERVICE_NAME = e.g., "otp-general"
BASE_PATH    = e.g., "services/otp-general/"
OUTPUT_PATH  = "{BASE_PATH}docs/"
REPORT_PATH  = "docs/project/reports/{EPIC}-{SERVICE_NAME}/"
EPIC         = e.g., "E05"
```

## Task

Run all quality checks on the completed {SERVICE_NAME} implementation. Write a report to `{REPORT_PATH}qa-verify-report.md`.

---

## Check 1: Build

```bash
cd {BASE_PATH} && go build ./... 2>&1
```

PASS if: exit code 0, no errors.
FAIL if: any compilation error.

---

## Check 2: Vet

```bash
cd {BASE_PATH} && go vet ./... 2>&1
```

PASS if: exit code 0.
FAIL if: any vet warning.

---

## Check 3: Tests

```bash
cd {BASE_PATH} && go test ./test/... -v 2>&1 | tail -50
```

Report: total PASS / FAIL / SKIP counts.

---

## Check 4: GORM Tags

Grep all entity files for structs and verify:

```bash
grep -rn "gorm:\"" {BASE_PATH}internal/entity/ 2>&1
```

For each entity struct field:
- [ ] `column:` tag matches DB column name from design-decisions.md
- [ ] `type:` tag matches DB column type
- [ ] `not null` present where required
- [ ] `default:` present where applicable
- [ ] No `AutoMigrate` call anywhere in non-docs code

```bash
grep -rn "AutoMigrate" {BASE_PATH} --include="*.go" | grep -v "docs/" 2>&1
```

---

## Check 5: app_origin Filter

All repositories that query unified tables MUST filter by app_origin.

```bash
grep -rn "app_origin" {BASE_PATH}internal/repository/ 2>&1
```

For each repository file, verify every `db.Where` or `db.Find` on a unified table includes app_origin condition.

Flag any repository method on a unified table that does NOT include app_origin filter.

---

## Check 6: ISP Compliance

No interface should have more than 5 methods.

```bash
grep -A 20 "type.*interface" {BASE_PATH}internal/domain/repository/*.go 2>&1
grep -A 20 "type.*interface" {BASE_PATH}internal/domain/service/*.go 2>&1
```

Count methods per interface. Flag any interface with > 5 methods.

---

## Check 7: Error Wrapping

Verify error wrapping format is consistent.

```bash
grep -rn "fmt.Errorf" {BASE_PATH}internal/ --include="*.go" 2>&1
```

Each `fmt.Errorf` must follow: `fmt.Errorf("{layer}: {action}: %w", ..., err)`.
Flag any that use `errors.New` where wrapping is needed, or lose the original error.

---

## Check 8: No Business Logic in Handlers

```bash
grep -rn "gorm\." {BASE_PATH}internal/handler/ --include="*.go" 2>&1
grep -rn "db\." {BASE_PATH}internal/handler/ --include="*.go" 2>&1
```

FAIL if any direct DB access found in handler layer.

---

## Report Format

Write to `{REPORT_PATH}qa-verify-report.md`:

```markdown
# QA Verify Report — {SERVICE_NAME} ({EPIC})
Date: {date}

## Summary

| Check | Status | Issues |
|-------|--------|--------|
| Build | ✅ PASS / ❌ FAIL | {N} errors |
| Vet | ✅ PASS / ❌ FAIL | {N} warnings |
| Tests | ✅ PASS / ❌ FAIL | {pass}/{total} |
| GORM Tags | ✅ PASS / ❌ FAIL | {N} issues |
| app_origin Filter | ✅ PASS / ❌ FAIL | {N} missing |
| ISP Compliance | ✅ PASS / ❌ FAIL | {N} violations |
| Error Wrapping | ✅ PASS / ❌ FAIL | {N} issues |
| Handler Purity | ✅ PASS / ❌ FAIL | {N} violations |

## Overall: ✅ PASS / ❌ FAIL — Proceed to qa-compat / Fix issues first

## Issues Detail

### {Check Name}
- `{file}:{line}` — {description}
  Fix: {specific action}
```
