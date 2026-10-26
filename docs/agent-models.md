# Agent Model Assignment

Each agent pins its model in the `model:` frontmatter of `.claude/agents/<name>.md`.

| Model | ID | Use for |
|---|---|---|
| Opus 5.5 | `claude-opus-5-5` | Architecture and orchestration only: the highest-leverage, hardest-to-reverse decisions |
| Sonnet 5.5 | `claude-sonnet-5-5` | Well-specified implementation work against an existing contract or plan |

| Agent | Model | Rationale |
|---|---|---|
| product-owner | Sonnet 5.5 | Requirements elicitation and PRD writing |
| technical-architect | Opus 5.5 | Service boundaries, schemas, and contracts are costly to change later |
| tech-lead | Opus 5.5 | Decomposes specs and coordinates parallel developers; needs strong planning |
| code-review-dev | Sonnet 5.5 | Diff review against contracts and .NET/security checklists |
| backend-dev | Sonnet 5.5 | Implements to an OpenAPI contract |
| mobile-maui-app-dev | Sonnet 5.5 | Implements to an OpenAPI contract |
| devops-dev | Sonnet 5.5 | Pipelines and Dockerfiles from established patterns |
| test-engineer-dev | Sonnet 5.5 | Test and environment setup from existing specs |

## Escalation
If a Sonnet agent hits a task that is genuinely complex (e.g. a tricky concurrency or data-migration design, or repeated failed attempts), the tech-lead may re-dispatch that task on Opus 5.5 rather than retrying.
