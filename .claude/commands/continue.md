---
description: |
  Resume a session from session/current.md. Handles all phases: planning resume, development
  with parallel entity dispatch, and QA. Development runs entities in parallel — all entity
  task-N run simultaneously, then all reviewers, then task-N+1, etc.
  Trigger when: user says "lanjut", "continue", "resume", "tadi sampai mana", or runs /continue.
---

# Continue — Session Resume + Development Loop

Reads session state and resumes from last checkpoint. Development phase dispatches entities
in parallel — multiple implementer agents run simultaneously across different entities.

---

## Step 1: Read Session State

Use the Read tool to read `{OUTPUT_PATH}session/current.md`.

If file not found, ask:
```
Question: "Path ke docs/ folder project yang mau dilanjutkan?"
Header: "Output Path"
Free text — e.g., "services/otp-general/docs/"
```

Extract: SERVICE_NAME, EPIC, BASE_PATH, OUTPUT_PATH, PHASE, WORKTREE_PATH, BRANCH,
planning STATUS/LAST_AGENT, development COMPLETED/TOTAL_TASKS/CURRENT_TASK/ENTITY_STATUS,
QA status.

---

## Step 2: Show Status Dashboard

Print status lalu langsung lanjut — tidak perlu konfirmasi.

```
╔══════════════════════════════════════════════════════════╗
║  Session Resume — {SERVICE_NAME} ({EPIC})                ║
╠══════════════════════════════════════════════════════════╣
║  Phase:    {PHASE}          Branch: {BRANCH}             ║
╠══════════════════════════════════════════════════════════╣
║  Planning: {STATUS}         last agent: {LAST_AGENT}     ║
╠══════════════════════════════════════════════════════════╣
║  Dev:      {COMPLETED}/{TOTAL_TASKS} tasks complete      ║
║  Entities:                                               ║
║    {entity_A}: {status}  {entity_B}: {status}            ║
║    {entity_C}: {status}  ...                             ║
╠══════════════════════════════════════════════════════════╣
║  QA:  verify={QA_VERIFY}  compat={QA_COMPAT}            ║
╠══════════════════════════════════════════════════════════╣
║  Changes recorded: {CHANGES}                             ║
╚══════════════════════════════════════════════════════════╝

▶ Melanjutkan otomatis dari phase: {PHASE} ...
```

---

## Step 3: Route by Phase

---

### PHASE = planning

#### If LAST_AGENT = task-breaker AND status = waiting-review

Show output files for review:
```
Output planning siap:
  {OUTPUT_PATH}design-decisions.md
  {OUTPUT_PATH}implementation-plan.md
  {OUTPUT_PATH}tasks-superpowers.md
  {OUTPUT_PATH}tasks-taiga.md

Review lalu ketik "approved" untuk lanjut ke worktree setup.
```

When user says "approved":
- Use Write tool to update session/current.md: `STATUS: approved`
- Run worktree setup (see PHASE = worktree-setup below)

#### If LAST_AGENT != task-breaker

Resume planning chain — read parameters from session/current.md, then:
- doc-reader done → spawn brainstormer (pre-read output files, inject content)
- brainstormer done → spawn planner (pre-read output files, inject content)
- planner done → spawn task-breaker (pre-read output files, inject content)

Update session/current.md after each agent completes.

---

### PHASE = worktree-setup

```bash
git worktree add "{WORKTREE_PATH}" -b {BRANCH}
```

Use Write tool to update session/current.md:
- `PHASE: development`

Print: `✅ Worktree siap. Melanjutkan ke development.`

Immediately continue to PHASE = development below.

---

### PHASE = development

This is the parallel dispatch loop. Read it fully before starting.

#### Pre-loop: Skill Path Map

Read LANGUAGE from session/current.md. Use the matching block below.
Lead session reads each skill file and injects content into agent prompt — never pass skill name only.

```
SKILL_ROOT = .claude/skills/

──────────────────────────────────────────────────────────────
LANGUAGE = go
──────────────────────────────────────────────────────────────
config       → go/go-config.md + core/tdd.md
entity       → go/go-entity.md + core/tdd.md
dto          → go/go-dto.md + core/tdd.md
repository   → go/go-repository.md + go/go-domain-interface.md + core/tdd.md
service      → go/go-service.md + go/go-domain-interface.md + core/tdd.md
handler-http → go/go-handler-http.md + core/tdd.md
handler-rpc  → go/go-handler-rpc.md + core/tdd.md
routes       → go/go-routes.md
middleware   → go/go-middleware.md + core/tdd.md

──────────────────────────────────────────────────────────────
LANGUAGE = node
──────────────────────────────────────────────────────────────
config       → node/node-config.md + core/tdd.md
model        → node/node-model.md + core/tdd.md
dto          → node/node-dto.md + core/tdd.md
repository   → node/node-repository.md + core/tdd.md
service      → node/node-service.md + core/tdd.md
handler      → node/node-handler.md + core/tdd.md
routes       → node/node-routes.md
middleware   → node/node-middleware.md + core/tdd.md

──────────────────────────────────────────────────────────────
LANGUAGE = python
──────────────────────────────────────────────────────────────
config       → python/python-config.md + core/tdd.md
model        → python/python-model.md + core/tdd.md
schema       → python/python-schema.md + core/tdd.md
repository   → python/python-repository.md + core/tdd.md
service      → python/python-service.md + core/tdd.md
router       → python/python-router.md + core/tdd.md
middleware   → python/python-middleware.md + core/tdd.md

──────────────────────────────────────────────────────────────
LANGUAGE = php  (Laravel)
──────────────────────────────────────────────────────────────
config       → php/php-config.md + core/tdd.md
model        → php/php-model.md + core/tdd.md
resource     → php/php-resource.md + core/tdd.md
repository   → php/php-repository.md + core/tdd.md
service      → php/php-service.md + core/tdd.md
controller   → php/php-controller.md + core/tdd.md
routes       → php/php-routes.md
middleware   → php/php-middleware.md + core/tdd.md

──────────────────────────────────────────────────────────────
LANGUAGE = other
──────────────────────────────────────────────────────────────
Use core/tdd.md + core/git-workflow.md only.
No language-specific skill files — agent uses language conventions directly.
```

All paths above are relative to SKILL_ROOT (`.claude/skills/`).
If a skill file does not exist → skip it, do not error. Note the gap in session/current.md.

---

#### Pre-loop: Load Task Map

Use the Read tool to read `{OUTPUT_PATH}tasks-superpowers.md`.

Build an in-memory task map:

```
PHASE_1_TASKS: [ list of shared component tasks — sequential ]
ENTITIES: {
  entity_A: { tasks: [task-1, task-2, ...], parallel_with: [entity_B, entity_C] },
  entity_B: { tasks: [task-1, task-2, ...], parallel_with: [entity_A, entity_C] },
  entity_C: { tasks: [task-1, task-2, ...], parallel_with: [entity_A, entity_B] },
}
PHASE_3_TASKS: [ list of wiring tasks — sequential ]
```

Read `{OUTPUT_PATH}session/current.md` to get current ENTITY_STATUS per entity.

---

#### Sub-phase A: Shared Components (Phase 1 — Sequential)

If any Phase 1 tasks are not yet done:

For each task in PHASE_1_TASKS (in order):

```
1. Read full task from {OUTPUT_PATH}implementation-plan.md
2. Read skill files for this layer using the Skill Path Map above → store as SKILL_CONTENT
3. Read {OUTPUT_PATH}design-decisions.md → DESIGN_DECISIONS

4. Dispatch implementer:

   Agent({
     prompt: `
     Task: {title}
     File: {file}
     Layer: {layer}
     BASE_PATH: {BASE_PATH}
     OUTPUT_PATH: {OUTPUT_PATH}

     {full task content from implementation-plan.md}

     --- skill: {layer} ---
     {SKILL_CONTENT}
     --- end skill ---

     --- design-decisions.md ---
     {DESIGN_DECISIONS}
     --- end ---
     `
   })

3. Wait for implementer to complete.

4. Dispatch reviewer:

   Agent({
     subagent_type: "reviewer",
     prompt: `
     SERVICE_NAME: {SERVICE_NAME}
     BASE_PATH: {BASE_PATH}
     OUTPUT_PATH: {OUTPUT_PATH}
     TASK_TITLE: {title}
     TASK_FILE: {file}
     TASK_LAYER: {layer}
     TASK_SCOPE: shared
     `
   })

5. Wait for reviewer verdict.
   - APPROVED → update session/current.md, move to next task
   - NEEDS_FIX → show issues, re-dispatch implementer with fix instructions, re-review
```

---

#### Sub-phase B: Entity Implementation (Phase 2 — Parallel)

This is where parallel dispatch happens.

**Concept:** All entities advance together, round by round.
Round 1 = task-1 of every entity. Round 2 = task-2. Etc.

Determine MAX_ROUNDS = max number of tasks across all entities.

For round = 1 to MAX_ROUNDS:

```
─── Round {round} ───────────────────────────────────────

STEP B1: Identify eligible entities for this round
  eligible = entities where:
    - entity has a task at position {round}
    - entity's previous round task is APPROVED (or round = 1)

STEP B2: Dispatch all eligible implementers IN PARALLEL
  (send all Agent() calls in a single response — they run simultaneously)

  For each eligible entity:
    Read task-{round} for entity from implementation-plan.md
    Read skill files for this task's layer using Skill Path Map → SKILL_CONTENT
    Read {OUTPUT_PATH}design-decisions.md → DESIGN_DECISIONS
    Read {OUTPUT_PATH}contract-matrix.md → CONTRACT_MATRIX

    Agent({
      name: "impl-{entity}-round{round}",
      prompt: `
      Task: {title}
      Entity: {entity}
      File: {file}
      Layer: {layer}
      Scope: WP | EXT | both
      BASE_PATH: {BASE_PATH}
      OUTPUT_PATH: {OUTPUT_PATH}

      {full task content — no placeholders}

      --- skill: {layer} ---
      {SKILL_CONTENT}
      --- end skill ---

      --- design-decisions.md ---
      {DESIGN_DECISIONS}
      --- end ---

      --- contract-matrix.md ---
      {CONTRACT_MATRIX}
      --- end ---
      `
    })

  ← All agents above run simultaneously. Wait for ALL to complete.

STEP B3: Dispatch all reviewers IN PARALLEL
  (send all reviewer Agent() calls in a single response)

  For each entity that completed in B2:
    Agent({
      subagent_type: "reviewer",
      name: "review-{entity}-round{round}",
      prompt: `
      SERVICE_NAME: {SERVICE_NAME}
      BASE_PATH: {BASE_PATH}
      OUTPUT_PATH: {OUTPUT_PATH}
      TASK_TITLE: {title}
      TASK_FILE: {file}
      TASK_LAYER: {layer}
      TASK_SCOPE: {WP | EXT | both}
      `
    })

  ← All reviewers run simultaneously. Wait for ALL to complete.

STEP B4: Handle verdicts
  For each entity:
    - APPROVED → update ENTITY_STATUS in session/current.md to next round
                 increment COMPLETED counter
    - NEEDS_FIX → re-dispatch that entity's implementer (with fix instructions)
                  re-review THAT entity only
                  other entities are NOT blocked

STEP B5: Update session/current.md
  Update ENTITY_STATUS for all entities.
  Update COMPLETED count.
  Print progress:

  Round {round} complete:
    entity_A: task-{round} ✅    entity_B: task-{round} ✅
    entity_C: task-{round} ✅
  Progress: {COMPLETED}/{TOTAL_TASKS} tasks done

─── end Round {round} ────────────────────────────────────
```

Repeat until all entities have no more tasks.

---

#### Sub-phase C: Wiring (Phase 3 — Sequential, Lead Session)

After all entities done, print:

```
Semua entity selesai. Melanjutkan ke Phase 3: wiring.
  - Routes registration → internal/routes/routes.go
  - main.go wiring → DI chain, server init
```

Lead session implements routes + main.go directly using Write tool.
Commit: `feat({SERVICE_NAME}): wire routes and main entry point`

Use Write tool to update session/current.md: `PHASE: qa`

---

### PHASE = qa

Run QA team sequentially.

**If qa-verify = not-run:**
```
Agent({
  subagent_type: "qa-verify",
  prompt: `
  SERVICE_NAME: {SERVICE_NAME}
  BASE_PATH: {BASE_PATH}
  OUTPUT_PATH: {OUTPUT_PATH}
  REPORT_PATH: {REPORT_PATH}
  EPIC: {EPIC}
  `
})
```
Wait. Read result. Update session/current.md: `QA_VERIFY: pass | fail`

If FAIL → print issues → auto-dispatch implementer to fix each issue → re-run qa-verify.
If still FAIL after fix → record issues to session/current.md notes, print warning, continue to qa-compat.
If PASS → continue.

**If qa-compat = not-run AND qa-verify = pass:**
```
Agent({
  subagent_type: "qa-compat",
  prompt: `
  SERVICE_NAME: {SERVICE_NAME}
  BASE_PATH: {BASE_PATH}
  OUTPUT_PATH: {OUTPUT_PATH}
  REPORT_PATH: {REPORT_PATH}
  EPIC: {EPIC}
  `
})
```
Wait. Read result. Update session/current.md: `QA_COMPAT: pass | fail`

**If qa-report = not-run AND both pass:**
```
Agent({
  subagent_type: "qa-report",
  prompt: `
  SERVICE_NAME: {SERVICE_NAME}
  BASE_PATH: {BASE_PATH}
  OUTPUT_PATH: {OUTPUT_PATH}
  REPORT_PATH: {REPORT_PATH}
  EPIC: {EPIC}
  `
})
```

Update session/current.md: `PHASE: done`

---

### PHASE = done

```
╔══════════════════════════════════════════════════════╗
║  ✅ {SERVICE_NAME} selesai. QA passed.               ║
║                                                      ║
║  Branch: {BRANCH}                                    ║
║  Report: {REPORT_PATH}qa-summary-report.md           ║
║                                                      ║
║  Next: buat PR atau merge ke main.                   ║
╚══════════════════════════════════════════════════════╝
```

---

## Parallel Dispatch Rules

1. **Same response = parallel.** Multiple `Agent()` calls in one response run simultaneously.
2. **Different responses = sequential.** Wait for all agents in round N before starting round N+1.
3. **NEEDS_FIX is isolated.** One entity failing review does NOT block other entities.
4. **Phase 1 always before Phase 2.** All shared components done before any entity starts.
5. **Phase 3 always after Phase 2.** All entities done before wiring starts.
6. **Entity with dependency.** If entity B depends on entity A's output (e.g., shared base struct), mark B as NOT parallel with A — B starts only after A's relevant task is APPROVED.
