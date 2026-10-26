# Multi-Agent Engineering Corporation

A Claude Code driven repository for an ASP.NET Core microservice backend and a .NET MAUI mobile client, developed against shared OpenAPI contracts.

## Layout
- `.claude/agents/` - agent definitions (product owner, architect, tech lead, backend, mobile, reviewer, test engineer, devops)
- `docs/prd/` - Product Requirement Documents
- `docs/architecture/` - architecture blueprints
- `docs/contracts/` - OpenAPI 3.1 contracts (source of truth)
- `src/backend/` - ASP.NET Core services
- `src/mobile/` - .NET MAUI client
- `src/tests/` - integration and system tests

## Workflow
1. `product-owner` grills requirements into a PRD.
2. `technical-architect` produces the architecture and OpenAPI contracts.
3. `tech-lead` creates GitHub issues/branches, dispatches `backend-dev` and `mobile-maui-app-dev` (and `devops-dev` when pipelines are needed), gates PRs through `code-review-dev`, then runs `test-engineer-dev`.

## Local environment
`docker compose -f docker-compose.local.yml up -d`

See `CLAUDE.md` for engineering standards. Direct commits to `main` are forbidden after the initial scaffold.
