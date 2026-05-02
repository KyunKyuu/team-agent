---
name: change-management
description: Protocol for handling mid-flow changes — record, assess impact, continue without restarting.
---

# Change Management Protocol

## When to Invoke

A change occurs when — mid-development — any of these happen:
- Schema or table structure changes
- API contract changes (request/response fields)
- Business logic rule changes
- Scope additions or removals
- Technology or library swaps

Trigger: user says something like "ternyata schema berubah", "ada tambahan field", "PM minta perubahan", etc.

---

## Protocol Steps

### Step 1: Pause Current Work

Stop the current task immediately. Do not continue implementation with the old spec.

```
⏸️  Change detected. Pausing task: {current_task_title}
```

### Step 2: Clarify the Change

Ask the user:
```
Perubahan apa yang terjadi?
  A. Schema / database field berubah
  B. Request / response field berubah  
  C. Business logic berubah
  D. Scope ditambah / dikurangi
  E. Lainnya

Apa yang berubah secara spesifik?
Apa alasannya?
```

### Step 3: Assess Impact

Read `{OUTPUT_PATH}implementation-plan.md` and identify:

```
Completed tasks (safe):     Task 1 - Task {last_completed}
Current task (paused):      Task {current_N} — {title}
Affected tasks:             Task {N} - Task {M} — need re-evaluation
Unaffected tasks:           Task {M+1} onwards — safe to continue as-is
```

Present impact to user:
```
Perubahan ini mempengaruhi {N} task:
  🔴 PERLU DIUBAH: Task {list}
  🟢 TIDAK BERUBAH: Task {list}

Pilihan:
  A. Re-plan task yang terpengaruh (brainstormer + planner untuk bagian itu saja)
  B. Adjust manual — saya beri instruksi spesifik
  C. Batalkan perubahan — lanjut dengan spec awal
```

### Step 4: Record the Change

Write to `{OUTPUT_PATH}session/changelog.md`:

```markdown
## Change #{N} — {date} {time}

**Trigger:** {what user said}
**Type:** schema | contract | logic | scope | other
**Description:** {specific change}
**Reason:** {why it changed}

**Impact:**
- Tasks completed before change: Task 1-{last_completed} (safe, no rework needed)
- Tasks affected: Task {list} (re-planned or adjusted)
- Tasks unaffected: Task {list} (continue as-is)

**Resolution:** {how it was resolved — re-plan / manual adjust / cancelled}
**Decided by:** {user / lead session}
**Timestamp:** {ISO 8601}
```

### Step 5: Continue

After recording and resolving:
- If re-planning: spawn brainstormer + planner for the affected section only
- If manual adjust: update implementation-plan.md for affected tasks, then continue
- Update `session/current.md` with new current task

---

## Rules

- NEVER restart the entire planning process for a partial change
- NEVER continue implementing with the old spec after a change is detected
- ALWAYS record to changelog before proceeding — this is the audit trail
- Changes to completed tasks: assess whether rework is needed; if yes, flag explicitly
- If the change is small (e.g., one field name), adjust inline without re-planning
