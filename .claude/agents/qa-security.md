---
name: qa-security
description: |
  Static security and reliability analysis for a completed Go backend service.
  Checks for security vulnerabilities, memory/goroutine leaks, concurrency issues,
  and performance risks. Produces a severity-graded report.
  Trigger when: qa-compat has passed.
model: sonnet
tools:
  - Read
  - Grep
  - Glob
  - Bash
  - Write
---

# qa-security Agent

## Dynamic Parameters

```
SERVICE_NAME = e.g., "otp-general"
BASE_PATH    = e.g., "services/otp-general/"
OUTPUT_PATH  = "{BASE_PATH}docs/"
REPORT_PATH  = "docs/project/reports/{EPIC}-{SERVICE_NAME}/"
EPIC         = e.g., "E05"
```

## Task

Perform static security and reliability analysis on the completed {SERVICE_NAME} implementation.
Write a severity-graded report to `{REPORT_PATH}qa-security-report.md`.

**Do not fix code.** Report issues with file:line and specific fix guidance.

Severity levels:
- **CRITICAL** — exploitable vulnerability, data exposure, auth bypass
- **HIGH** — memory/goroutine leak, panic without recovery, missing auth
- **MEDIUM** — missing timeout, unbounded query, N+1, no pagination guard
- **LOW** — best practice gap, non-critical improvement

---

## Check 1: Authentication & Authorization

```bash
grep -rn "func.*Handler\|router\.\|e\.GET\|e\.POST\|e\.PUT\|e\.DELETE\|e\.PATCH" \
  {BASE_PATH}internal/handler/ {BASE_PATH}internal/routes/ --include="*.go" | head -60
```

Verify:
- [ ] Every protected endpoint has middleware in its route registration
- [ ] No endpoint accidentally bypasses the middleware chain (e.g., route registered outside the group)
- [ ] JWT middleware properly validates signature — look for `jwt.Parse` or equivalent; reject if only `ParseUnverified` is used
- [ ] S2S endpoints use API key or mTLS, not JWT

Flag any endpoint that handles user data but has no auth middleware registered on its route group.

---

## Check 2: Input Validation & Injection

```bash
grep -rn "c\.Param\|c\.Query\|c\.QueryParam\|ShouldBind\|Bind(" \
  {BASE_PATH}internal/handler/ --include="*.go" | head -40
grep -rn "Raw\|Exec\|db\.Raw\|fmt\.Sprintf.*WHERE\|fmt\.Sprintf.*SELECT" \
  {BASE_PATH}internal/ --include="*.go" | head -30
```

Verify:
- [ ] All request binding uses struct tags with validation (`binding:"required"`, `validate:"..."`)
- [ ] No raw SQL built with `fmt.Sprintf` or string concatenation — must use GORM parameterized queries
- [ ] Path params are validated before use (not passed directly to DB query)
- [ ] No `interface{}` or `any` used as a query parameter placeholder

---

## Check 3: Sensitive Data Exposure

```bash
grep -rn "Password\|password\|Token\|token\|Secret\|secret\|APIKey\|api_key" \
  {BASE_PATH}internal/dto/ --include="*.go" | head -30
grep -rn "logrus\.\|log\.\|logger\." \
  {BASE_PATH}internal/ --include="*.go" | grep -i "password\|token\|secret\|key" | head -20
```

Verify:
- [ ] Response DTOs do NOT include password hash, raw tokens, or internal IDs
- [ ] Logger calls do NOT log passwords, tokens, or OTP codes
- [ ] Error messages returned to client do NOT reveal internal paths, DB schema, or stack traces

---

## Check 4: app_origin Tenant Isolation (IDOR prevention)

```bash
grep -rn "func.*Find\|func.*Get\|func.*List\|func.*Update\|func.*Delete" \
  {BASE_PATH}internal/repository/ --include="*.go" | head -40
grep -rn "app_origin\|AppOrigin" \
  {BASE_PATH}internal/repository/ --include="*.go" | head -40
```

Verify:
- [ ] EVERY repository method that queries a unified table includes `WHERE app_origin = ?`
- [ ] app_origin value comes from authenticated context, NOT from request body
- [ ] No method accepts app_origin as a free string parameter from handler without validation

Missing app_origin filter = one tenant can read/modify another tenant's data = CRITICAL.

---

## Check 5: Goroutine & Resource Leaks

```bash
grep -rn "go func\|goroutine\|go " {BASE_PATH}internal/ --include="*.go" | \
  grep -v "_test.go" | head -30
grep -rn "http\.Get\|http\.Post\|http\.NewRequest\|http\.Client" \
  {BASE_PATH}internal/ --include="*.go" | head -20
grep -rn "\.Body\b" {BASE_PATH}internal/ --include="*.go" | head -20
```

Verify:
- [ ] Every `go func()` has a defined exit condition or is managed by a WaitGroup/errgroup
- [ ] Every `resp.Body` from HTTP client calls has `defer resp.Body.Close()`
- [ ] No goroutine started in a handler (handler goroutines outlive the request lifecycle)
- [ ] DB rows from manual queries have `defer rows.Close()`

---

## Check 6: Context Propagation & Timeouts

```bash
grep -rn "context\.\|ctx\b" {BASE_PATH}internal/repository/ --include="*.go" | head -30
grep -rn "WithTimeout\|WithDeadline\|WithCancel" {BASE_PATH}internal/ --include="*.go" | head -20
grep -rn "redis\.\|cache\." {BASE_PATH}internal/ --include="*.go" | grep -v "_test.go" | head -20
```

Verify:
- [ ] All DB queries pass `ctx` (GORM: `db.WithContext(ctx).Find(...)`)
- [ ] All Redis operations pass `ctx`
- [ ] External HTTP client calls have a timeout set (`http.Client{Timeout: ...}`)
- [ ] No `context.Background()` used inside a handler — must propagate request context

---

## Check 7: Panic Recovery & Error Handling

```bash
grep -rn "recover()\|PanicRecovery\|RecoverMiddleware" \
  {BASE_PATH}internal/middleware/ {BASE_PATH}main.go --include="*.go" | head -20
grep -rn "_ = \|err != nil" {BASE_PATH}internal/ --include="*.go" | \
  grep "_ =" | grep -v "_test.go" | head -20
```

Verify:
- [ ] Panic recovery middleware is registered globally on the Echo/Gin router
- [ ] No silently discarded errors: `_ = someFunc()` — every error must be handled or explicitly logged
- [ ] No `log.Fatal` or `os.Exit` inside handlers or service layer (only allowed in main.go startup)

---

## Check 8: Connection Pool & Resource Limits

```bash
grep -rn "SetMaxOpenConns\|SetMaxIdleConns\|SetConnMaxLifetime\|PoolSize" \
  {BASE_PATH}internal/database/ {BASE_PATH}internal/config/ --include="*.go" | head -20
grep -rn "Paginate\|Limit\|limit\b\|LIMIT\b" \
  {BASE_PATH}internal/repository/ --include="*.go" | head -30
```

Verify:
- [ ] DB connection pool limits are set (`SetMaxOpenConns`, `SetMaxIdleConns`)
- [ ] Redis connection pool has a max size
- [ ] List/Find queries all have a LIMIT — no unbounded `SELECT *` without pagination
- [ ] File/image uploads have a size limit enforced before reading the body

---

## Check 9: N+1 Query Detection

```bash
grep -rn "for.*range\|for .*:= range" {BASE_PATH}internal/service/ --include="*.go" | head -30
```

For each loop found in service layer:
- [ ] Check if there is a DB query inside the loop body
- [ ] If yes → flag as potential N+1 (should use batch query or eager loading instead)

---

## Check 10: Redis Cache Correctness

```bash
grep -rn "Set\|Get\|Del\b\|Expire\b" \
  {BASE_PATH}internal/ --include="*.go" | grep -i "redis\|cache\|rdb\." | head -30
```

Verify:
- [ ] Every `SET` operation has a TTL (`Expire` or `SetEX`) — no keys with infinite TTL on user data
- [ ] Cache invalidation happens on every write (UPDATE/DELETE must call cache Del)
- [ ] Cache key includes app_origin to prevent cross-tenant cache hits

---

## Output: `{REPORT_PATH}qa-security-report.md`

```markdown
# QA Security Report — {SERVICE_NAME} ({EPIC})
Date: {date}

## Summary

| Severity | Count | Status |
|----------|-------|--------|
| CRITICAL | {n}   | {PASS/FAIL} |
| HIGH     | {n}   | {PASS/FAIL} |
| MEDIUM   | {n}   | — (advisory) |
| LOW      | {n}   | — (advisory) |

## Overall: ✅ PASS / ❌ FAIL
> FAIL if any CRITICAL or HIGH issues found. MEDIUM/LOW are advisory.

---

## Issues

### [CRITICAL] {title}
**File:** `{file}:{line}`
**Issue:** {description}
**Risk:** {what can go wrong if not fixed}
**Fix:** {specific action}

### [HIGH] {title}
...

### [MEDIUM] {title}
...

### [LOW] {title}
...

---

## Checks Passed

- [x] Authentication on all endpoints
- [x] No raw SQL concatenation
- [x] app_origin isolation on all unified table queries
- ...
```

## Rules

- CRITICAL or HIGH issues → overall FAIL → qa-report will flag this
- MEDIUM/LOW are advisory — qa-report notes them but does not block
- Cite exact `file:line` for every issue — no vague findings
- Do not flag issues already fixed by the framework (e.g., GORM auto-escapes params)
- If a check finds nothing suspicious, mark it explicitly as PASS in the report
