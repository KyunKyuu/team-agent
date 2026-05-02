# jrku-external-apps — Agent Teams

## Project

Jasa Raharja insurance platform rebuild. Merges two legacy services (jr-web-partner + jr-external) into unified Go microservices. Backward compat mandatory — JSON response tags must match legacy exactly.

## Agent Teams

Enabled via `.claude/settings.local.json`:
```json
{ "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }
```

## Agents (`.claude/agents/`)

**Planning (sequential):** `doc-reader` → `brainstormer` → `planner` → `task-breaker`
**Development:** Lead dispatches fresh general-purpose subagent per task. Parallel across entities, sequential within entity.
**QA (sequential):** `qa-verify` → `qa-compat` → `qa-report`
**Review:** `reviewer` — two-stage (spec + code quality). Dispatched by lead after each implementer.

## Commands (`.claude/commands/`)

- `/jr-init` — start new unified service rebuild (planning → worktree → development → QA)
- `/continue` — resume session from `session/current.md`
- `/change` — handle mid-flow requirement change

## Go Skills (`.claude/skills/go/`)

`go-entity` `go-dto` `go-domain-interface` `go-repository` `go-service`
`go-handler-http` `go-handler-rpc` `go-middleware` `go-routes` `go-config`
`go-main-wiring` `go-migration-doc` `go-project-structure` `go-testing`

## Core Skills (`.claude/skills/core/`)

`tdd` `git-workflow` `change-management` `taiga`

## Key Constraints

- `app_origin` filter mandatory on all unified table queries (partitioned by "web_partner" / "external")
- No GORM AutoMigrate in production — SQL files are source of truth
- Two channels (REST + RabbitMQ RPC) → same service layer
- TDD: write failing test → verify fail → implement → verify pass → commit
- Tests in `test/{layer}/`, not alongside source
