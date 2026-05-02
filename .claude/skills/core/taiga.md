---
name: taiga
description: Taiga task format — how to read tickets, map to implementation, and format output as Taiga user stories.
---

# Taiga Integration

## Reading a Ticket from Taiga

When given a Taiga ticket (user story or task), extract:

```
US ID:       e.g., US-123
Title:       the user story title
Epic:        parent epic (e.g., E05)
Description: full description + business context
Acceptance Criteria: the checklist items — these define DONE
Attachments: any linked docs, mockups, or migration docs
```

The acceptance criteria = the implementation spec. Every criterion must be testable.

---

## Mapping Ticket → Tasks

For each acceptance criterion, identify what layer it touches:

| Criterion | Layer |
|-----------|-------|
| "API endpoint returns X" | handler + dto |
| "Data is saved to DB" | repository |
| "Business rule: if X then Y" | service |
| "Validated field Z" | dto (validation tag) |
| "Auth required" | middleware + handler |

Group criteria by entity → produces the task list.

---

## Writing Taiga-Ready Tasks

### User Story Format

```markdown
**US-{EPIC}{N:03}: {title}**

Type: Task
Epic: {EPIC}
Story Points: {1-3}
Tags: {entity}, {layer}, {scope}

**Description:**
As a {role}, I need {what} so that {why / business value}.

**Acceptance Criteria:**
- [ ] {specific, testable criterion}
- [ ] {specific, testable criterion}
- [ ] Test exists and passes: `go test ./test/{layer}/... -run Test{Name}`
- [ ] JSON response matches legacy contract (field names, types)
- [ ] app_origin filter applied (if touching shared table)
- [ ] Committed as `feat({service}): {description}`
```

### Story Points Scale

| Points | Meaning |
|--------|---------|
| 1 | Simple: one file, clear spec, no edge cases (config, entity struct) |
| 2 | Medium: 2-3 files, some logic (DTO + mapper, repo impl) |
| 3 | Complex: cross-layer, business logic, multiple scenarios (service impl, handler) |

---

## Updating Taiga After Completion

When a task is done (test passes, commit made), the record should include:
```
Status:  Done
Branch:  feat/{SERVICE_NAME}
Commit:  {commit hash}
PR:      {PR URL if applicable}
```

---

## Bulk Import Format

For tasks-taiga.md, group by epic then list sequentially:

```markdown
# Taiga Import — {SERVICE_NAME} ({EPIC})
Total: {N} tasks

---
{US-001}
{US-002}
...
```

Each US separated by `---`. This format can be imported or manually entered into Taiga.
