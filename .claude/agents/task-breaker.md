---
name: task-breaker
description: |
  Break implementation plan into two outputs: Superpowers task breakdown (with parallelism markers)
  and Taiga-ready user stories (US format with acceptance criteria).
  Trigger when: planner has completed and implementation-plan.md is ready.
model: sonnet
tools:
  - Read
  - Write
---

# task-breaker Agent

## Dynamic Parameters

```
SERVICE_NAME = e.g., "otp-general"
BASE_PATH    = e.g., "services/otp-general/"
PLAN_PATH    = e.g., "docs/project/plan/E05-otp/"
OUTPUT_PATH  = "{BASE_PATH}docs/"
EPIC         = e.g., "E05"

# Input (read from OUTPUT_PATH):
implementation-plan.md  ← from planner
```

## Task

Read `{OUTPUT_PATH}implementation-plan.md` and produce TWO output files using the **Write tool**.

---

## Output 1: `{OUTPUT_PATH}tasks-superpowers.md`

Superpowers task breakdown for the lead session to dispatch subagents.

### Format

```markdown
# Superpowers Tasks — {SERVICE_NAME} ({EPIC})

## Execution Order

Phase 1 — Shared Components (sequential, run before all entities)
Phase 2 — Entity implementation (parallel across entities, sequential within)
Phase 3 — Wiring (sequential, run after all entities)

---

## Phase 1: Shared Components

> Run sequentially. Complete before starting any entity.

- [ ] Task 1: {title} → `{file}` — {1 line description}
- [ ] Task 2: {title} → `{file}` — {1 line description}

---

## Phase 2: Entities

> Entities marked ✅ PARALLEL can run simultaneously.
> Tasks within each entity MUST run sequentially (listed in order).

### 🔀 Entity: {entity_A}  ✅ PARALLEL with: {entity_B}, {entity_C}

- [ ] Task N: {title} → `{file}`
- [ ] Task N+1: {title} → `{file}`
  ...

### 🔀 Entity: {entity_B}  ✅ PARALLEL with: {entity_A}, {entity_C}

- [ ] Task N: {title} → `{file}`
  ...

---

## Phase 3: Wiring

> Run after all entities complete. Lead session handles this directly.

- [ ] Routes registration → `internal/routes/routes.go`
- [ ] main.go wiring → `main.go`

---

## Dispatch Template

When dispatching a subagent for a task, use this structure:

\`\`\`
Agent({
  prompt: `
Task: {task title}
File: {exact file path}
Layer: {layer}
Scope: WP | EXT | both

[full task content from implementation-plan.md — paste verbatim]

Skills to apply: {relevant go-* skill(s) for this layer}
Design decisions: {OUTPUT_PATH}design-decisions.md
Contract matrix: {OUTPUT_PATH}contract-matrix.md
  `
})
\`\`\`
```

---

## Output 2: `{OUTPUT_PATH}tasks-taiga.md`

Taiga-ready output for PM. Behavior depends on whether PM provided stories.

### Check: did doc-reader find Taiga stories?

Read `{OUTPUT_PATH}doc-reader-summary.md` and look for the `## Taiga Stories` section.

---

### Mode A — PM Stories PROVIDED (stories found in doc-reader-summary.md)

Map each PM user story to the implementation tasks that fulfill it.
Do NOT generate new stories — use the PM's stories as-is.

```markdown
# Taiga Task Mapping — {SERVICE_NAME} ({EPIC})

> Stories dari PM. Setiap US dipetakan ke sub-tasks implementasi.

---

## {US-EPIC-N}: {original US title from PM}

**Original Acceptance Criteria:**
{paste acceptance criteria from PM story as-is}

**Sub-tasks (implementation):**
- [ ] Task {N}: {title} → `{file}` ({layer}, {scope})
- [ ] Task {N+1}: {title} → `{file}` ({layer}, {scope})

**Coverage check:**
- Endpoints covered: {list API paths this US maps to}
- All acceptance criteria addressed: {yes | partial — missing: {list}}

---
```

**Mapping rules (Mode A):**
- Keep US numbers from PM exactly — never renumber
- One US can map to multiple impl tasks (1:many is normal)
- If an impl task covers multiple US → list it under each US
- If an impl task has no matching US → flag it: `⚠️ No US found for this task`
- If a PM US has no impl tasks → flag it: `⚠️ US not covered by any task`

---

### Mode B — NO PM Stories (stories not found)

Generate developer-level stories from implementation plan.

```markdown
# Taiga Tasks — {SERVICE_NAME} ({EPIC})

---

## US-{EPIC}{N:03}: {task title}

**Type:** Task
**Epic:** {EPIC}
**Estimate:** {2-5} min
**Tags:** {entity_name}, {layer}, {scope}

**Description:**
As a developer, I implement {what} in the {SERVICE_NAME} service
so that {why — relate to business/API requirement}.

**Acceptance Criteria:**
- [ ] Test written and passing
- [ ] Code follows {BOILERPLATE} patterns
- [ ] JSON response tags match contract-matrix.md for {scope} scope
- [ ] app_origin filter applied (if query touches unified table)
- [ ] Error wrapped as `{layer}: %w`
- [ ] Committed as `feat({SERVICE_NAME}): {description}`

---
```

**Rules (Mode B):**
- One Taiga US per implementation-plan task (1:1 mapping)
- US number: `{EPIC}{sequential_N:03}` — e.g., E05001, E05002...
- Estimate: use the time from implementation plan (2-5 min as written)
- Tags: always include entity name + layer + scope (WP/EXT/shared)
- Acceptance criteria: always include the 5 standard criteria above, add task-specific ones

## Rules

- Do NOT summarize or compress tasks — preserve all detail from implementation-plan.md
- Parallelism markers in tasks-superpowers.md must match the "Parallel-eligible with" fields from the plan
- Every task in implementation-plan.md must appear in BOTH output files
- Count tasks and verify totals match before writing
