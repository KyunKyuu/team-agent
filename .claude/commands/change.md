---
description: |
  Handle a mid-flow change request — record it, assess impact, and continue without restarting.
  Applies the change-management protocol: pause → clarify → assess → record → continue.
  Trigger when: user says "ada perubahan", "ternyata berubah", "PM minta ubah", "schema ganti", or runs /change.
---

# Change — Mid-Flow Change Handler

Handles changes that occur during planning or development without losing progress.

---

## Step 1: Pause

Immediately stop whatever is in progress.

```
⏸️  Change Management Protocol Activated
Menghentikan task yang sedang berjalan sementara...
```

Use the Read tool to read `{OUTPUT_PATH}session/current.md`.

If OUTPUT_PATH is unknown or file not found, ask:
```
Question: "Path ke docs/ folder project? (contoh: services/otp-general/docs/)"
Header: "Output Path"
Free text.
```

Extract from session state: PHASE, CURRENT_TASK, COMPLETED count, OUTPUT_PATH for use in subsequent steps.

---

## Step 2: Ask for Change Details

```
Question: "Apa yang berubah?"
Header: "Jenis Perubahan"
Options:
  - "Schema / database field berubah"
  - "Request / response field berubah"
  - "Business logic berubah"
  - "Scope ditambah / dikurangi"
  - "Lainnya"
MultiSelect: false
```

Then ask:
```
Question: "Ceritakan perubahan spesifiknya"
Header: "Detail Perubahan"
Free text.
```

Then ask:
```
Question: "Apa alasannya? (opsional — tapi penting untuk audit trail)"
Header: "Alasan"
Free text.
```

---

## Step 3: Assess Impact

Use the Read tool to read:
- `{OUTPUT_PATH}implementation-plan.md` — find which tasks are affected
- `{OUTPUT_PATH}session/current.md` — which tasks are already completed (safe)

Map the change to affected tasks:

```
✅ Completed (safe — no rework needed): Task 1 - Task {N}
⏸️  In Progress (paused):               Task {current}
🔴 Affected (need re-evaluation):       Task {list}
🟢 Unaffected (safe to continue):       Task {rest}
```

Present to user:
```
Perubahan ini mempengaruhi:
  {N} task yang perlu disesuaikan: {list titles}
  {M} task yang tidak terpengaruh: aman dilanjutkan

Pilih cara penanganan:
  A. Re-plan — spawn brainstormer+planner untuk bagian yang terpengaruh
  B. Adjust manual — jelaskan perubahan spesifik per task, saya update plannya
  C. Batalkan perubahan ini — lanjut dengan spec lama
```

---

## Step 4: Record to Changelog

Regardless of choice, ALWAYS write to `{OUTPUT_PATH}session/changelog.md`:

```markdown
## Change #{N} — {YYYY-MM-DD HH:MM}

**Type:** {schema | contract | logic | scope | other}
**Trigger:** "{exact words user said}"
**Description:** {specific change in detail}
**Reason:** {why — from user}

**Impact Assessment:**
- Completed tasks (safe): Task 1-{N} — no rework
- Affected tasks: {list with titles}
- Unaffected tasks: {list}

**Resolution:**
- Choice: {A re-plan | B manual adjust | C cancelled}
- Action taken: {what was done}
- Updated files: {list of files changed}

**Decided by:** user
**Timestamp:** {ISO 8601}
```

---

## Step 5: Execute Resolution

### If A (Re-plan affected tasks):

Spawn brainstormer with:
```
CONTEXT: "Partial re-plan — only tasks {N}-{M} are affected by this change"
CHANGE: {change description}
PREVIOUS_DECISIONS: {OUTPUT_PATH}design-decisions.md
AFFECTED_TASKS: {list}
```

After brainstormer done → spawn planner for affected tasks only → update implementation-plan.md (replace only affected sections).

### If B (Manual adjust):

For each affected task, ask user:
```
Task {N}: {title}
Perubahan apa yang diperlukan untuk task ini?
```

Update implementation-plan.md inline for each affected task using the Write tool.
Use the Write tool to update `{OUTPUT_PATH}session/current.md`: CURRENT_TASK stays the same unless the current task itself is affected.

### If C (Cancelled):

```
Perubahan dibatalkan. Melanjutkan dengan spec lama.
```

Still record to changelog with Resolution: "cancelled".
Resume from where we left off.

---

## Step 6: Resume

After resolution, ask:
```
Question: "Perubahan sudah dihandle. Lanjutkan dari task yang dipause?"
Options:
  - "Ya, lanjutkan task yang dipause"
  - "Tidak, saya mau review dulu sebelum lanjut"
  - "Tidak, stop session sekarang"
```

**If "Ya"** → resume the paused task with updated spec. Use Write tool to update `{OUTPUT_PATH}session/current.md` to reflect new state.

**If "Tidak, saya mau review dulu"** → print:
```
Ok. Review file berikut sebelum lanjut:
  {OUTPUT_PATH}session/changelog.md   ← record perubahan tadi
  {OUTPUT_PATH}implementation-plan.md ← task yang sudah di-adjust

Kalau sudah siap, jalankan /continue untuk resume development.
```

**If "Tidak, stop session"** → print:
```
Session di-pause. State tersimpan di:
  {OUTPUT_PATH}session/current.md

Jalankan /continue kapan saja untuk resume.
```
