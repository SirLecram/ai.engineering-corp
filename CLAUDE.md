# Enterprise Engineering Organization Guidelines

## Technology Stack & Standards
- **Backend:** C# / .NET 8 or 9, ASP.NET Core Web APIs / Minimal APIs, Clean Architecture, Entity Framework Core.
- **Mobile:** C# / .NET MAUI, CommunityToolkit.Mvvm, Refit / HttpClientFactory, modern async/await patterns.
- **Contracts:** OpenAPI 3.1 YAML specifications stored in `/docs/contracts/`.
- **Git Flow:** All work is tracked via GitHub Issues and Pull Requests using the `gh` CLI. Direct commits to `main` are strictly forbidden.

## Working Directory Isolation
- Agents MUST only write files within their domain boundary.
- All tasks must be completed on dedicated feature branches: `feat/<issue-number>-<short-description>`.
- Any shared schema modification requires an explicit revision of `/docs/contracts/`.

## Reference Templates
- If a prompt names other repositories as templates or references, every agent MUST use them as the basis for structure and conventions, read them without modifying them, and never copy secrets.
