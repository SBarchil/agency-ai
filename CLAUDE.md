# Agency Platform — instructions for Claude Code

New back-office, B2B and guest platform for a Moroccan tour operator / DMC (branches Agadir and Marrakech).
It replaces an old VB.NET + SQL Server desktop app. **No data is migrated**: the new system starts empty.
The old system is only a source of lessons (see `docs/analysis-and-design.md`).

The developer is building this project to learn. When introducing a concept for the first time
(see the concept tables in `docs/development-strategy.md`), explain it briefly with an example from this code,
and offer to add a note in `docs/learning/<concept>.md`.

## Read these first
- `docs/analysis-and-design.md` — what the system must do: contracts, pricing, bookings and itineraries, billing, treasury, B2B, AI features, caching, Kubernetes.
- `docs/development-strategy.md` — how we build it: vertical slices, 7-phase roadmap, walking skeleton, concept map.
- `docs/adr/` — decisions already made. Do not contradict an ADR; propose a new ADR instead.

## Stack (decided)
- Backend: C# on the current .NET LTS, ASP.NET Core minimal APIs, EF Core with Npgsql, PostgreSQL.
- Front ends: React + TypeScript + Vite (back office, partner portal); Next.js (public site). pnpm workspaces.
- Infra: Docker, Helm, Kubernetes (managed cluster), managed PostgreSQL and Redis, GitHub Actions.
- AI: LLM API behind interfaces; MCP server (official MCP C# SDK) for the manager assistant.

## Architecture rules
- Clean Architecture + modular monolith. Layers: `Agency.Domain` → `Agency.Application` → `Agency.Infrastructure` → hosts
  (`Agency.BackOffice`, `Agency.PartnerApi`, `Agency.PublicSite`, `Agency.Worker`, `Agency.McpServer`, `Agency.Migrator`).
- Dependencies point inward. Domain references no framework (no EF Core, no ASP.NET). Application never references Infrastructure.
- Modules are folders inside each layer: Catalogue, Contracts, Pricing, Bookings, Partners, Billing, Treasury, GuestAssistant, Identity.
  A module never reaches into another module's internals; use its application interface or a domain event.
- Business rules live in entities and value objects (no anemic models). Examples: one active contract version per date;
  issued invoices are immutable; corrections go through credit notes.
- Repositories per aggregate only. **No generic repository.** No MediatR/AutoMapper unless an ADR says so; use plain handlers and explicit mapping.
- Expected failures return a Result; exceptions are for bugs and infrastructure errors.
- Architecture tests (NetArchTest or ArchUnitNET) enforce these rules and must pass.

## Data rules
- Money: `decimal` in C#, `numeric(18,2)` in PostgreSQL; exchange rates `numeric(18,6)`. Never `float`/`double` for money.
- Stay dates: `DateOnly` / `date`. Moments: `DateTimeOffset` / `timestamptz` in UTC. Use `TimeProvider`, never `DateTime.Now`.
- Every table has a primary key and foreign keys. Use check, unique and exclusion constraints (`daterange`) for invariants.
- Table and column names in snake_case.
- Invoice numbers, allotment and payments are changed only inside database transactions with row locks. Never trust a cached value for them.
- Contracts, circuit templates and vouchers are versioned; old versions are kept read-only.

## AI rules
- AI only creates **drafts** (contracts, booking emails, receipts). A logged-in human approves. Enforce in API permissions, not only in UI.
- The guest assistant reads only published catalogue, policies and public prices; never costs, margins, contracts or other guests' data.
- MCP tools are read-only to start; no free-form SQL tool.
- Every AI feature has a test set in `tests/evals/` and a measured accuracy.

## How to work
- Build in thin vertical slices: domain test first, then domain code, application, infrastructure, API, React screen, Helm values.
- Keep each change small enough for one pull request. Run `dotnet test` and the front-end tests before saying a task is done.
- Definition of done: tests pass, no warnings, OpenAPI updated, permissions checked, meaningful logs, works on the test cluster, ADR if a decision was made.
- Ask before adding a new dependency, a new project, or anything that changes an ADR.
- Never put real customer data or secrets in code, tests or prompts. Secrets come from configuration / Kubernetes Secrets.

## Commands (fill in as the project grows)
- Backend tests: `dotnet test backend/Agency.sln`
- Front end: `pnpm -C frontend install`, `pnpm -C frontend -r test`
- Local services: `docker compose up -d`
