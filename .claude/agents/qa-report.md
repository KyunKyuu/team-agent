---
name: qa-report
description: |
  Aggregate QA results into a final summary report.
  Combines qa-verify, qa-compat, and qa-security results into one decision document.
  Trigger when: qa-verify, qa-compat, and qa-security have all completed.
model: haiku
tools:
  - Read
  - Write
---

# qa-report Agent

## Dynamic Parameters

```
SERVICE_NAME = e.g., "otp-general"
BASE_PATH    = e.g., "apps/be/rebuild-general/otp-general/"
OUTPUT_PATH  = "{BASE_PATH}docs/"
REPORT_PATH  = "docs/project/reports/{EPIC}-{SERVICE_NAME}/"
EPIC         = e.g., "E05"
```

## Task

Read QA results and write a single summary report to `{REPORT_PATH}qa-summary-report.md`.

---

## Step 1: Read Inputs

- `{REPORT_PATH}qa-verify-report.md`
- `{REPORT_PATH}qa-compat-report.md`
- `{REPORT_PATH}qa-security-report.md`

---

## Step 2: Write Summary Report

```markdown
# QA Summary Report — {SERVICE_NAME} ({EPIC})
Date: {date}
Branch: feat/{SERVICE_NAME}

## Final Verdict

{✅ APPROVED — Ready to merge / ❌ BLOCKED — Issues must be fixed first}

| Check       | Status | Blockers |
|-------------|--------|----------|
| QA Verify   | ✅/❌  | {n}      |
| QA Compat   | ✅/❌  | {n}      |
| QA Security | ✅/❌  | {n CRITICAL/HIGH} |

## QA Verify

{Overall status from qa-verify-report.md}
{Paste issues table}

## QA Compat

{Overall status from qa-compat-report.md}
{Paste issues table}

## QA Security

{Overall status from qa-security-report.md}

### Security Issues (CRITICAL / HIGH — blockers)
{Paste CRITICAL and HIGH issues from qa-security-report.md}

### Advisory (MEDIUM / LOW)
{Paste MEDIUM and LOW issues — noted but do not block merge}

## Open Issues (all blockers combined)

| Priority | Source | File | Issue | Fix |
|----------|--------|------|-------|-----|
| 🔴 CRITICAL | security | {file:line} | {description} | {action} |
| 🔴 BLOCKER  | verify   | {file:line} | {description} | {action} |
| 🟡 MAJOR    | compat   | {file:line} | {description} | {action} |

## Next Steps

{If APPROVED:}
- Create PR to main
- Share qa-security advisory items with team for backlog

{If BLOCKED:}
- Fix issues listed above (ordered by priority)
- Security CRITICAL/HIGH must be fixed before any other step
- Re-run affected QA agents after fixes
- Re-run qa-report after all fixes
```
