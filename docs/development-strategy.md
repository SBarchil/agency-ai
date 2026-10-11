# Development Strategy & Learning Roadmap

Companion to `analysis-and-design.md`. Phase numbers: 1 Skeleton · 2 Contracts · 3 Bookings · 4 Billing · 5 Channels · 6 AI · 7 Go-live.

## Strategy

Ten rules keep a large system manageable for a small team.

### Thin vertical slices, not layers

Build one feature end to end (database → domain → API → screen → deployed) before starting the next, instead of finishing the whole database, then the whole backend.

### Walking skeleton first

The very first slice is deliberately tiny but crosses every layer and is deployed to Kubernetes, so infrastructure problems appear in week one, not at go-live.

### Domain first inside each slice

Write the business rule and its unit tests in the Domain layer before any screen. Pricing, contract versions and invoicing are where bugs cost money.

### One module at a time, in dependency order

Catalogue → Contracts → Pricing → Bookings and itineraries → Billing → Treasury → channels and AI.

### Automate from day one

Every push runs build, tests and architecture tests; every merge to main deploys to a test environment.

### Show the customer every two weeks

A short demo on the test environment, with their real contracts and circuits, catches wrong assumptions early. This is the core habit of a Forward Deployed Engineer.

### Write decisions down

One short Architecture Decision Record (ADR) per important choice: what was decided, why, what was rejected.

### AI features after the core works

Contract intake, guest assistant and MCP rely on clean data and working use cases; build them on top, each with its test set.

### Use defaults, avoid cleverness

Standard ASP.NET Core, EF Core, React and Helm patterns; no custom frameworks, no microservices, no generic repositories.

### Learn just in time

Each concept in this guide is attached to the phase that first needs it; learn it then, practise it in the real code, and write a short note in the repository.

## Roadmap

The phases follow the modules’ dependencies; a phase ends only when its gate can be shown to the customer on the test environment.

Walking skeleton

solution, CI/CD, Kubernetes, sign-in, one screen end to end

Gate

A city typed in the browser is saved on the test cluster

Catalogue and contracts

products, partners, contract versions, pricing engine

Gate

Prices match the contracting team's worked examples

Bookings and itineraries

files, circuit templates, vouchers, transfers, lists

Gate

A real circuit produces correct vouchers and transfers

Billing and treasury

invoices, payments, cash sessions, bank, ledger

Gate

One month closed end to end on test data

Channels

public site, partner portal and partner API

Gate

A partner books through the sandbox API

AI features

contract intake, guest assistant, manager assistant (MCP)

Gate

Each AI feature reaches its test-set target

Pilot go-live

one branch first, then the second

Gate

Customer signs off after the pilot weeks

AI contract intake can start earlier, alongside phase 2, if typing contracts becomes the bottleneck; it only needs the contract model and the approval flow to exist.

## The walking skeleton

The first slice is “create a city and list cities”: trivial business logic, but it forces every technical piece into place once.

1.  Create the solution: Domain, Application, Infrastructure, BackOffice host, test projects; add architecture tests.
2.  Domain: a `City` entity with a name rule (not empty, unique).
3.  Application: `CreateCity` command and `ListCities` query, with validation.
4.  Infrastructure: EF Core `DbContext`, first schema migration, PostgreSQL in Docker.
5.  API: two endpoints, OpenAPI document, health checks, structured logs.
6.  Sign-in: OpenID Connect with a role “admin”; the endpoints require it.
7.  React back office: Vite + TypeScript app, sign-in, a cities page with a table and a form, API client generated from OpenAPI.
8.  Containers: Dockerfiles for the API and the React app; `docker compose` runs everything locally.
9.  Kubernetes: Helm chart with Deployments, Services, Ingress, Secret, probes; deploy to a local kind cluster, then to the test cluster.
10. CI/CD: GitHub Actions builds, tests, pushes images and deploys to test on every merge.

**Done when** a city typed in the browser on the test cluster appears in PostgreSQL. Every later feature reuses this path.

The second slice is the first real one: **a hotel contract with one season and one rate, and the pricing engine returning a price for a stay**. It brings in value objects (`Money`, `DateRange`), contract versions, exclusion constraints and the first domain tests that matter.

## Repository, tooling and workflow

One repository holds everything, so a feature (backend, screen, Helm values) lands in one pull request.

``` tree
agency/
  backend/              .NET solution (src/ + tests/, Clean Architecture)
  frontend/
    apps/backoffice/      React + Vite (staff)
    apps/partner-portal/  React + Vite (B2B partners)
    apps/public-site/     Next.js (guests, search-engine indexing)
    packages/ui/          shared components
    packages/api-client/  generated from the API's OpenAPI document
  deploy/helm/          one chart, values per environment
  docs/adr/             architecture decision records
  .github/workflows/    CI/CD
  CLAUDE.md             conventions for AI coding assistants
```

### Tooling

| Need             | Tool                                                                          |
|------------------|-------------------------------------------------------------------------------|
| C# editor        | Visual Studio, Rider or VS Code with C# Dev Kit                               |
| Front end        | Node.js LTS, pnpm workspaces, VS Code                                         |
| Local services   | Docker Desktop, `docker compose` (PostgreSQL, Redis, mail catcher)            |
| Local Kubernetes | kind or k3d, `kubectl`, Helm, k9s                                             |
| Source and CI/CD | GitHub, GitHub Actions, container registry of the chosen cloud                |
| Database         | pgAdmin or DBeaver                                                            |
| API testing      | the OpenAPI UI, `.http` files in the repository                               |
| AI assistant     | Claude Code (or similar) with a `CLAUDE.md` describing the architecture rules |

### Daily workflow

1.  Pick one small task from the current slice (half a day to two days).
2.  Create a branch; write the domain test first, then the code.
3.  Open a pull request; CI runs build, unit, integration and architecture tests.
4.  Review your own diff (or the AI’s) against the definition of done; merge.
5.  CI deploys to the test environment automatically; check it there.

**Definition of done** for any task: tests pass, no warnings, OpenAPI updated, permissions checked, logs meaningful, works on the test cluster, ADR written if a decision was made.

## Concepts: C# and .NET

The coloured label shows the phase in which to learn each concept; you do not need everything at once.

| Concept                                                       | What it is                                           | Where we use it                                         | Phase         |
|---------------------------------------------------------------|------------------------------------------------------|---------------------------------------------------------|---------------|
| Records and value objects                                     | Immutable types compared by value                    | `Money`, `DateRange`, `Occupancy`                       | 2 · Contracts |
| Nullable reference types                                      | Compiler warns when something may be null            | Whole solution, turned on from the start                | 1 · Skeleton  |
| `decimal`, `DateOnly`, `TimeOnly`                             | Exact money type; dates and times without time zones | Prices, stay dates, pick-up times                       | 2 · Contracts |
| `TimeProvider`                                                | Replaceable clock                                    | Release days, option expiry, tests that control “today” | 2 · Contracts |
| async/await, `CancellationToken`                              | Non-blocking I/O, cancellable requests               | Every database, AI and HTTP call                        | 1 · Skeleton  |
| LINQ                                                          | Querying collections and EF Core in C#               | Queries, reports                                        | 1 · Skeleton  |
| Generics and interfaces                                       | Reusable, swappable code                             | `IContractExtractor`, `IFileStorage`, repositories      | 1 · Skeleton  |
| Dependency injection                                          | The framework builds objects and their dependencies  | Wiring layers in each host                              | 1 · Skeleton  |
| Configuration and Options pattern                             | Typed settings from files, environment, secrets      | Connection strings, AI keys, per-environment values     | 1 · Skeleton  |
| Minimal APIs, route groups, filters                           | Lightweight HTTP endpoints                           | Back office API, partner API                            | 1 · Skeleton  |
| Middleware                                                    | Code that runs around every request                  | Errors, logging, correlation IDs, rate limiting         | 1 · Skeleton  |
| ProblemDetails                                                | Standard error format for APIs                       | All APIs                                                | 1 · Skeleton  |
| EF Core: DbContext, migrations, owned types, value converters | ORM mapping C# to PostgreSQL                         | Persistence of all modules                              | 1 · Skeleton  |
| EF Core concurrency tokens and transactions                   | Detect conflicting updates; all-or-nothing saves     | Allotment, approvals, invoice numbering                 | 2 · Contracts |
| Validation (FluentValidation or built-in)                     | Input rules separate from business rules             | Commands from screens and APIs                          | 1 · Skeleton  |
| Result pattern vs exceptions                                  | Returning expected failures as values                | “Contract overlaps”, “credit limit exceeded”            | 2 · Contracts |
| Logging with `ILogger` (structured)                           | Logs with named fields, searchable                   | Everywhere; shipped as JSON to stdout                   | 1 · Skeleton  |
| Hosted services, job queue                                    | Work outside HTTP requests                           | AI extraction, emails, PDFs, option expiry              | 3 · Bookings  |
| `HybridCache`                                                 | Two-level cache (memory + Redis)                     | Reference data, rate tables, availability               | 3 · Bookings  |
| ASP.NET Core Identity, OpenID Connect, JWT                    | Sign-in and tokens                                   | Staff, partners, MCP                                    | 1 · Skeleton  |
| Authorization policies                                        | Named permissions checked in code                    | “ApproveContract”, “IssueInvoice”, branch limits        | 2 · Contracts |
| OpenAPI                                                       | Machine-readable API description                     | Generated React clients, partner docs                   | 1 · Skeleton  |
| Rate limiting, output caching                                 | Built-in middleware                                  | Partner API, public catalogue                           | 5 · Channels  |
| Health checks                                                 | Endpoints Kubernetes probes call                     | Every host                                              | 1 · Skeleton  |
| OpenTelemetry                                                 | Traces, metrics, logs in one standard                | Following a booking across API, worker, database        | 3 · Bookings  |
| xUnit, Testcontainers, NetArchTest                            | Unit, integration and architecture tests             | All slices                                              | 1 · Skeleton  |
| PDF and Excel libraries                                       | Generating documents                                 | Invoices, vouchers, rooming lists, reports              | 3 · Bookings  |

## Concepts: architecture and design patterns

| Concept                               | What it is                                                                               | Where we use it                                                      | Phase         |
|---------------------------------------|------------------------------------------------------------------------------------------|----------------------------------------------------------------------|---------------|
| Clean Architecture                    | Layers with dependencies pointing inward to the domain                                   | Whole solution                                                       | 1 · Skeleton  |
| Modular monolith                      | One deployable app split into modules with clear borders                                 | Catalogue, Contracts, Bookings, Billing, Treasury…                   | 1 · Skeleton  |
| Vertical slice                        | One feature built through all layers at once                                             | How work is planned                                                  | 1 · Skeleton  |
| DDD: entity, value object, aggregate  | Objects with identity; objects defined by values; a cluster saved together with one root | `Contract` + its versions; `Booking` + its itinerary                 | 2 · Contracts |
| Invariants                            | Rules an aggregate always enforces                                                       | “One active version”, “issued invoice is immutable”                  | 2 · Contracts |
| Domain events                         | “Something happened” messages inside the app                                             | `ContractVersionApproved` flags sales contracts, clears cache        | 2 · Contracts |
| CQRS-lite                             | Commands change state, queries only read                                                 | Use cases; reports and lists                                         | 2 · Contracts |
| Repository per aggregate              | Loads and saves one aggregate                                                            | `IContractRepository`, `IBookingRepository`                          | 2 · Contracts |
| Unit of work and transactions         | Several changes saved all together or not at all                                         | Booking + allotment + voucher                                        | 3 · Bookings  |
| Optimistic vs pessimistic concurrency | Detect conflicts on save vs lock rows first                                              | Editing contracts vs taking allotment and invoice numbers            | 3 · Bookings  |
| State machine                         | Allowed status changes defined in one place                                              | Contract versions, files, vouchers, cheques                          | 2 · Contracts |
| Versioning (data)                     | Keep every version, one active per date                                                  | Contracts, circuit templates, vouchers                               | 2 · Contracts |
| Outbox pattern                        | Save the event in the same transaction, send it afterwards                               | Emails, webhooks, WhatsApp, voucher sending never lost or duplicated | 3 · Bookings  |
| Idempotency                           | Repeating a request has no extra effect                                                  | Partner bookings, payment callbacks, webhooks                        | 5 · Channels  |
| API versioning                        | `/v1`, `/v2` live side by side                                                           | Partner API                                                          | 5 · Channels  |
| Audit log                             | Who did what and when                                                                    | Approvals, invoices, payments, closings                              | 2 · Contracts |
| Ledger (double entry)                 | Every movement debits one account and credits another                                    | Billing and treasury                                                 | 4 · Billing   |
| Architecture Decision Record          | Short note of a decision and its reasons                                                 | `docs/adr`                                                           | 1 · Skeleton  |

## Concepts: PostgreSQL

| Concept                                 | What it is                                 | Where we use it                               | Phase         |
|-----------------------------------------|--------------------------------------------|-----------------------------------------------|---------------|
| Schema design, primary and foreign keys | Tables that enforce relationships          | Every table (the old system had almost none)  | 1 · Skeleton  |
| Constraints: unique, check, not null    | The database refuses bad data              | Codes, amounts ≥ 0, statuses                  | 1 · Skeleton  |
| `numeric`, `date`, `timestamptz`        | Exact money; dates; moments with time zone | Prices, stays, audit times                    | 1 · Skeleton  |
| `daterange` + exclusion constraints     | Ranges that must not overlap               | Seasons, one active contract version per date | 2 · Contracts |
| Indexes (B-tree, GIN, GiST)             | Fast lookups; JSON and range searches      | Searches, availability, `jsonb`               | 2 · Contracts |
| Transactions and isolation levels       | Rules for concurrent changes               | Bookings, invoices, payments                  | 3 · Bookings  |
| Row locks (`SELECT … FOR UPDATE`)       | Lock a row until the transaction ends      | Allotment counters, number series             | 3 · Bookings  |
| `jsonb`                                 | Structured JSON stored and indexed         | AI extraction output, request logs            | 6 · AI        |
| Collations and `citext`                 | Case- and accent-insensitive text          | Hotel and client names in French              | 1 · Skeleton  |
| `pgvector`                              | Similarity search on embeddings            | Guest assistant knowledge                     | 6 · AI        |
| Views for reporting                     | Read-only shapes of data                   | MCP tools, reports                            | 4 · Billing   |
| Backups and point-in-time recovery      | Restore to any moment                      | Production database (managed service)         | 7 · Go-live   |
| Read-only roles                         | Users that can only read                   | Reporting and MCP access                      | 5 · Channels  |

## Concepts: React and front end

Back office and partner portal are React single-page apps built with Vite; the public site uses Next.js because guests arrive from search engines and pages must be rendered on the server.

| Concept                                               | What it is                                                           | Where we use it                        | Phase         |
|-------------------------------------------------------|----------------------------------------------------------------------|----------------------------------------|---------------|
| TypeScript                                            | JavaScript with types                                                | All front-end code                     | 1 · Skeleton  |
| Components, props, state                              | Building blocks of the UI and their data                             | Every screen                           | 1 · Skeleton  |
| Hooks (`useState`, `useEffect`, custom hooks)         | Reusable logic in components                                         | Forms, data loading, permissions       | 1 · Skeleton  |
| React Router                                          | Pages and URLs in a single-page app                                  | Back office, partner portal            | 1 · Skeleton  |
| TanStack Query                                        | Fetching, caching and refreshing server data                         | Every list and detail screen           | 1 · Skeleton  |
| Generated API client                                  | TypeScript types and calls from the OpenAPI document                 | Back office, portal, public site       | 1 · Skeleton  |
| Forms: React Hook Form + Zod                          | Form state and validation                                            | Contracts, bookings, payments          | 2 · Contracts |
| Component library and data grid                       | Ready-made inputs, tables, dialogs (e.g. MUI, Ant Design, shadcn/ui) | Back office screens with large tables  | 1 · Skeleton  |
| OIDC sign-in in the browser                           | Login and tokens for the API                                         | All apps                               | 1 · Skeleton  |
| Route guards and permission-aware UI                  | Hide or disable what a user may not do                               | Approve buttons, billing screens       | 2 · Contracts |
| Internationalisation, right-to-left                   | Several languages; Arabic layout                                     | Public site, guest documents           | 5 · Channels  |
| Next.js: server rendering, static pages, SEO          | Pages built on the server for search engines                         | Public catalogue and circuit pages     | 5 · Channels  |
| Real-time updates (SignalR or polling)                | Screens refresh when data changes                                    | Dispatch board, staff inbox            | 3 · Bookings  |
| Chat UI and streaming responses                       | Showing AI answers as they are written                               | Guest assistant widget                 | 6 · AI        |
| Front-end tests (Vitest, Testing Library, Playwright) | Component and end-to-end tests                                       | Critical flows: book, invoice, approve | 2 · Contracts |

## Concepts: Kubernetes and DevOps

| Concept                                                       | What it is                                                     | Where we use it                                    | Phase         |
|---------------------------------------------------------------|----------------------------------------------------------------|----------------------------------------------------|---------------|
| Containers and multi-stage Dockerfiles                        | App + runtime packaged as an image; small final images         | Every host and front end                           | 1 · Skeleton  |
| Container registry and image tags                             | Where images are stored; one tag per build                     | CI pushes, Kubernetes pulls                        | 1 · Skeleton  |
| Pod, Deployment, ReplicaSet                                   | Running containers and keeping N copies alive                  | Back office, partner API, public site, worker, MCP | 1 · Skeleton  |
| Service and Ingress                                           | Stable internal address; HTTP routing from outside             | One host name per front end and API                | 1 · Skeleton  |
| ConfigMap and Secret                                          | Settings and passwords given to pods                           | Connection strings, AI keys                        | 1 · Skeleton  |
| Liveness and readiness probes                                 | How Kubernetes knows a pod is alive and ready                  | Health check endpoints                             | 1 · Skeleton  |
| Resource requests and limits                                  | CPU and memory reserved and capped per pod                     | All deployments                                    | 1 · Skeleton  |
| Job and CronJob                                               | Run-once and scheduled tasks                                   | Schema updates per release; nightly checks         | 2 · Contracts |
| Namespaces                                                    | Separate environments in one cluster                           | dev, test, production                              | 1 · Skeleton  |
| Helm (charts, values)                                         | Templates for Kubernetes files, one value file per environment | `deploy/helm`                                      | 1 · Skeleton  |
| cert-manager                                                  | Automatic HTTPS certificates                                   | Every public host name                             | 1 · Skeleton  |
| Horizontal Pod Autoscaler                                     | More pods when load grows                                      | Public site, partner API                           | 5 · Channels  |
| Rolling updates and rollbacks                                 | New version replaces old without downtime; undo if broken      | Every release                                      | 1 · Skeleton  |
| CI/CD with GitHub Actions                                     | Build, test, push, deploy on every change                      | Whole repository                                   | 1 · Skeleton  |
| GitOps with Argo CD (optional)                                | The cluster follows what is in Git                             | Production deployments later                       | 7 · Go-live   |
| Observability: Prometheus, Grafana, Loki or the cloud’s tools | Metrics, dashboards, log search, alerts                        | Production monitoring                              | 3 · Bookings  |
| Managed PostgreSQL and Redis                                  | Cloud services outside the cluster                             | Data and cache                                     | 1 · Skeleton  |

## Concepts: AI, RAG and MCP

| Concept                                | What it is                                                                    | Where we use it                                            | Phase        |
|----------------------------------------|-------------------------------------------------------------------------------|------------------------------------------------------------|--------------|
| LLM API basics                         | Sending messages to a model, getting text back; tokens, cost, latency         | All AI features                                            | 6 · AI       |
| Structured output (JSON schema)        | Model answers in a fixed JSON shape you validate                              | Contract and booking-email extraction                      | 6 · AI       |
| Reading documents and images           | Model reads PDFs, scans, photos                                               | Supplier contracts, receipts, brochures                    | 6 · AI       |
| Tool use (function calling)            | Model asks your code to run a function and uses the result                    | Guest assistant: availability, leads, handoff              | 6 · AI       |
| Prompt design and system prompts       | Instructions, examples, rules for the model                                   | Every AI feature                                           | 6 · AI       |
| Embeddings and RAG                     | Turn text into vectors; find relevant passages; give them to the model        | Guest assistant knowledge in `pgvector`                    | 6 · AI       |
| Evals (test sets)                      | Fixed questions or documents with known correct answers, scored automatically | Contract intake accuracy, guest answers, manager figures   | 6 · AI       |
| Guardrails                             | Limits enforced in code, not only in the prompt                               | Read-only tools, no approval by AI, no cost data to guests | 6 · AI       |
| Human in the loop                      | AI drafts, a person approves                                                  | Contracts, booking emails, receipts                        | 6 · AI       |
| Prompt caching                         | Reusing the fixed part of a prompt to save cost and time                      | Guest assistant                                            | 6 · AI       |
| MCP: server, tools, resources, prompts | Standard way to expose your system’s actions to AI clients                    | Manager assistant                                          | 5 · Channels |
| MCP transport and auth                 | Streamable HTTP; OAuth sign-in for remote servers                             | `Agency.McpServer` on Kubernetes                           | 5 · Channels |
| Speech-to-text and text-to-speech      | Voice in and out                                                              | Manager assistant by voice                                 | 5 · Channels |

## Concepts: security

| Concept                              | What it is                                               | Where we use it                           | Phase         |
|--------------------------------------|----------------------------------------------------------|-------------------------------------------|---------------|
| OAuth 2 and OpenID Connect           | Standard sign-in and access tokens                       | Staff, partners (client credentials), MCP | 1 · Skeleton  |
| Roles, permissions, data scopes      | What a user may do, and on which branch’s data           | Approvals, billing, branch managers       | 2 · Contracts |
| OWASP Top 10                         | The most common web vulnerabilities                      | Code reviews, API design                  | 1 · Skeleton  |
| Secrets management                   | Passwords and keys outside the code                      | Kubernetes Secrets or a key vault         | 1 · Skeleton  |
| Least privilege                      | Every account gets only what it needs                    | Database roles, AI service accounts       | 6 · AI        |
| Input validation and output encoding | Never trust input; escape output                         | APIs, chat widget                         | 1 · Skeleton  |
| Prompt injection                     | Text inside documents or chats trying to steer the AI    | Contract files, guest messages            | 6 · AI        |
| Personal data protection             | Consent, retention, deletion of guest data               | Guests, passports, chat history           | 3 · Bookings  |
| Audit trail and tamper resistance    | Logs nobody can silently edit                            | Invoices, payments, approvals             | 4 · Billing   |
| Dependency and image scanning        | Finding known vulnerabilities in packages and containers | CI pipeline                               | 1 · Skeleton  |

## How to learn, and how to use AI assistants

For each concept, follow the same short loop when its phase starts.

Read

the official documentation page for the concept (one or two hours, not a full course).

Try it in isolation

in a small scratch project: one Dockerfile, one EF Core migration, one React hook.

Apply it

in the real slice, with tests.

Explain it

in five to ten lines in `docs/learning/<concept>.md`, in your own words, with a link to where it is used in the code. Explaining turns reading into knowledge, and these notes are useful in FDE interviews.

Review it after a month

reread your note and the code; fix what you now see differently.

### Using an AI coding assistant without losing the learning

- Put the architecture rules in `CLAUDE.md` (layers, naming, testing rules, “no generic repositories”), so generated code follows them.
- Ask it to explain a concept with an example from *your* code before asking it to write the code.
- Write the domain test yourself, then let the assistant implement; you stay in charge of the business rules.
- Read every diff before merging; ask “why this way?” when something is unfamiliar, and note the answer.
- Let it handle repetitive work (mappings, Helm boilerplate, React forms) and spend your own time on pricing, versions, billing and security.
- Never paste real customer data or secrets into prompts; use test data.

## Official documentation

[.NET documentationlearn.microsoft.com/dotnet/](https://learn.microsoft.com/dotnet/) [ASP.NET Corelearn.microsoft.com/aspnet/core/](https://learn.microsoft.com/aspnet/core/) [Entity Framework Corelearn.microsoft.com/ef/core/](https://learn.microsoft.com/ef/core/) [Npgsql EF Core providerwww.npgsql.org/efcore/](https://www.npgsql.org/efcore/) [PostgreSQL documentationwww.postgresql.org/docs/](https://www.postgresql.org/docs/) [pgvectorgithub.com/pgvector/pgvector](https://github.com/pgvector/pgvector) [Reactreact.dev/learn](https://react.dev/learn) [TanStack Querytanstack.com/query/latest](https://tanstack.com/query/latest) [Next.jsnextjs.org/docs](https://nextjs.org/docs) [Kubernetes tutorialskubernetes.io/docs/tutorials/](https://kubernetes.io/docs/tutorials/) [Helmhelm.sh/docs/](https://helm.sh/docs/) [Model Context Protocolmodelcontextprotocol.io](https://modelcontextprotocol.io) [Claude API documentationdocs.claude.com](https://docs.claude.com) [OWASP Top 10owasp.org/www-project-top-ten/](https://owasp.org/www-project-top-ten/)
