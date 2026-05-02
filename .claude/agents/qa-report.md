---
name: qa-report
description: |
  Aggregate QA results into a final summary report.
  Combines qa-verify and qa-compat results into one decision document.
  Trigger when: both qa-verify and qa-compat have completed.
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

---

## Step 2: Write Summary Report

```markdown
# QA Summary Report — {SERVICE_NAME} ({EPIC})
Date: {date}
Branch: feat/{SERVICE_NAME}

## Final Verdict

{✅ APPROVED — Ready to merge / ❌ BLOCKED — Issues must be fixed first}

## QA Verify

{Overall status from qa-verify-report.md}

{Paste issues table from qa-verify-report.md}

## QA Compat

{Overall status from qa-compat-report.md}

{Paste issues table from qa-compat-report.md}

## Open Issues

{List all unresolved FAIL items across both reports with priority:}

| Priority | File | Issue | Fix |
|----------|------|-------|-----|
| 🔴 BLOCKER | {file:line} | {description} | {action} |
| 🟡 MAJOR | ... | ... | ... |

## Next Steps

{If APPROVED:}
- Run /superpowers:finishing-a-development-branch
- Create PR to main
- Notify QA team

{If BLOCKED:}
- Fix issues listed above (ordered by priority)
- Re-run qa-verify and/or qa-compat as needed
- Re-run qa-report after fixes
```
