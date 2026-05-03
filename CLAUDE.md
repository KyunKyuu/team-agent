# Claude Agent Team — Multi-Project Template

## What This Is

A reusable AI agent pipeline for rebuilding or extending backend services.
Supports any language. Handles single-source rebuilds, dual-source merges, and feature additions to existing projects.

## Agent Teams

Enabled via `.claude/settings.local.json`:
```json
{ "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }
```

## Commands (`.claude/commands/`)

- `/init`      — new service, any language, 0–2 legacy sources
- `/resume`    — add feature/epic to existing service
- `/continue`  — run development + QA loop autonomously
- `/change`    — handle mid-flow requirement change

> Project-specific profiles (e.g. `/jr-init`, `/jr-resume`) live in the same folder
> with pre-filled defaults for that project.

## Agents (`.claude/agents/`)

**Planning (sequential):** `doc-reader` → `brainstormer` → `planner` → `task-breaker`
**Development:** Lead dispatches fresh subagent per task. Parallel across entities, sequential within.
**QA (sequential):** `qa-verify` → `qa-compat` → `qa-report`
**Review:** `reviewer` — two-stage (spec + code quality). After each implementer.

## Skills (`.claude/skills/`)

Language-specific knowledge bases. Lead session reads and injects into agent prompts.

| Folder | Language |
|--------|----------|
| `go/`  | Go — entity, dto, repository, service, handler, routes, middleware, config, testing |
| `node/`| Node.js — model, dto, repository, service, handler, routes, middleware, testing |
| `python/` | Python — model, schema, repository, service, router, middleware, testing |
| `php/` | PHP/Laravel — model, resource, repository, service, controller, routes, middleware, testing |
| `core/` | Language-agnostic — tdd, git-workflow, change-management, taiga |

## Key Constraints (apply to all projects)

- Backward compat mandatory — response field names must match legacy exactly
- TDD: write failing test → verify fail → implement → verify pass → commit
- Tests in `test/{layer}/`, not alongside source
- No secrets or credentials in committed code
