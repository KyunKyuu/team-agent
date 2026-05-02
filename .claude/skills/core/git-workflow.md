---
name: git-workflow
description: Git workflow for agent teams — worktree isolation, branch conventions, commit format.
---

# Git Workflow for Agent Teams

## Worktree Isolation

Development happens in a git worktree — not on the main branch.

```bash
# Create worktree for a service
git worktree add ".claude/worktrees/{SERVICE_NAME}-{SCOPE_SHORT}" -b feat/{SERVICE_NAME}-{SCOPE_SHORT}

# Work inside the worktree
cd ".claude/worktrees/{SERVICE_NAME}-{SCOPE_SHORT}"

# Remove worktree after merge
git worktree remove ".claude/worktrees/{SERVICE_NAME}-{SCOPE_SHORT}"
```

Why worktrees:
- Main branch stays clean while development runs
- Multiple entities can have separate worktrees (true parallel)
- Lead session stays in main, subagents work in worktrees

---

## Branch Naming

```
feat/{SERVICE_NAME}                     ← single scope
feat/{SERVICE_NAME}-{SCOPE_SHORT}       ← scope-specific (wp or ext)
fix/{SERVICE_NAME}-{short-description}  ← bug fix
chore/{description}                     ← non-feature work
```

Examples:
```
feat/otp-general
feat/otp-general-wp
fix/otp-general-token-expiry
```

---

## Commit Format

```
{type}({scope}): {description}

Types: feat, fix, refactor, test, chore, docs
Scope: service name (e.g., otp-general)

Examples:
feat(otp-general): add OTP entity struct with GORM tags
test(otp-general): add failing test for OTP send service
fix(otp-general): correct app_origin filter in FindByPhone
```

**One commit per task** — commit after test passes, not before.

Commit order per task:
1. `test({scope}): failing test for {task}` → verify FAIL
2. `feat({scope}): {task description}` → verify PASS

Or combine into one commit if the task is atomic.

---

## Subagent Commit Rules

Each subagent that implements a task MUST:
1. Verify tests pass before committing
2. Use exact commit format above
3. Never commit failing tests
4. Never force-push

---

## PR Checklist (after all tasks complete)

```
- [ ] All tests pass: go test ./test/... -v
- [ ] Build clean: go build ./...
- [ ] Vet clean: go vet ./...
- [ ] QA verify: PASS
- [ ] QA compat: PASS
- [ ] No TODO or placeholder in code
- [ ] Branch is up to date with main
```

---

## Merge Strategy

```bash
# From main branch
git merge --no-ff feat/{SERVICE_NAME} -m "feat({SERVICE_NAME}): implement {SERVICE_NAME} service"

# Or create PR via gh
gh pr create --title "feat({SERVICE_NAME}): implement {SERVICE_NAME}" --base main --head feat/{SERVICE_NAME}
```
