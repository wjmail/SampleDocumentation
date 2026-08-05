# 05 — Architecture Decisions (ADR Log)

Format: [MADR](https://adr.github.io/madr/) — lightweight, Markdown-based.

---

## ADR-001 — Start as modular monolith, not microservices

**Status:** Accepted  
**Date:** 2025-01

### Context
The domain is being modelled from scratch. Service boundaries, data ownership, and event contracts are not yet proven. Premature decomposition leads to distributed monolith anti-patterns.

### Decision
Start with a single deployable ASP.NET Core application with clearly separated modules (one folder per bounded context, no cross-module direct class references). Extract to independent services only after boundaries are stable and operational justification exists.

### Consequences
- Simpler local dev, single deploy unit, no network latency between modules.
- Harder to scale individual modules independently (acceptable at this stage).
- Refactoring to microservices is structurally enabled by clean module boundaries.

---

## ADR-002 — Use .NET Aspire for local orchestration

**Status:** Accepted  
**Date:** 2025-01

### Context
The platform needs SQL Server, Service Bus emulator, and multiple future services running locally. Docker Compose is hard to maintain for .NET projects; Aspire provides first-class .NET integration.

### Decision
Use .NET Aspire `AppHost` as the local orchestration layer. All resources (SQL, Service Bus emulator, modules) are registered in AppHost. `azd` uses the AppHost manifest for Azure provisioning.

### Consequences
- Aspire Dashboard provides built-in traces, logs, and metrics locally at zero cost.
- Aspire is still evolving; accept minor API churn.
- When a module becomes an independent service, add it to AppHost as a new project reference.

---

## ADR-003 — Azure SQL as primary relational store

**Status:** Accepted  
**Date:** 2025-01

### Context
The domain is transaction-heavy (claims, repair orders, parts reservations). Strong consistency within a bounded context is required.

### Decision
Use Azure SQL (SQL Server) with EF Core. One schema per module in the same database server (dev), with a migration path to separate databases per service when extracted.

### Consequences
- Familiar tooling, strong ACID guarantees.
- Separate schemas enforce logical isolation today.
- Moving to separate databases later requires data migration scripts — acceptable cost.

---

## ADR-004 — Outbox pattern for event publishing

**Status:** Accepted  
**Date:** 2025-01

### Context
Domain events must be reliably published even if the message broker is temporarily unavailable. Publishing in the same transaction as the business operation ensures consistency.

### Decision
Use the Transactional Outbox pattern: persist `OutboxMessage` rows in the same database transaction as the aggregate change. A background worker polls the outbox and dispatches events to MediatR (in-process) or Service Bus (cross-service).

### Consequences
- At-least-once delivery guaranteed.
- Consumers must be idempotent (handled via `ProcessedEventId` deduplication table).
- Adds background polling overhead — acceptable.

---

## ADR-005 — CQRS within modules, not globally

**Status:** Accepted  
**Date:** 2025-01

### Context
Not all modules benefit equally from CQRS complexity. Claims and RepairOrders have complex business logic and query requirements. Parts and Notifications are simpler.

### Decision
Apply CQRS (Commands / Queries via MediatR handlers) within each module where it adds value. Do not force a global read-model infrastructure. Introduce separate read models only when query complexity or performance justifies it.

### Consequences
- Claims and RepairOrders use explicit Command/Query handlers.
- Parts and Notifications may use simpler service classes initially.
- No premature read-model infrastructure.

---

## ADR-006 — Microsoft Entra ID as identity provider

**Status:** Accepted  
**Date:** 2025-01

### Context
The platform needs AuthN and AuthZ. Building a custom IdP is out of scope. Entra ID is the Azure-native choice and integrates with Managed Identity.

### Decision
Use Microsoft Entra ID with OAuth2 / OIDC. APIs validate JWT Bearer tokens. App Roles define RBAC. Managed Identity is used for service-to-service calls (no passwords/keys in code).

### Consequences
- Entra ID Free tier is sufficient; zero cost for the identity layer.
- App Registrations persist between `azd up/down` cycles (no cost, no teardown needed).
- ACL in Identity module translates Entra concepts to internal domain model.

---

## ADR-007 — Azure Developer CLI (`azd`) for infra and deploy

**Status:** Accepted  
**Date:** 2025-01

### Context
Manual Azure deployments are error-prone and hard to reproduce. A consistent IaC + deploy CLI is needed.

### Decision
Use `azd` with Bicep for all Azure resource provisioning and application deployment. `azd up` = full stack provision + deploy. `azd down` = full teardown. GitHub Actions runs `azd` in CI/CD.

### Consequences
- `azd down` at end of day keeps Azure costs near zero.
- Aspire AppHost manifest is the single source of truth for resource wiring.
- `azd pipeline config` sets up GitHub Actions secrets automatically.

---

## ADR-008 — OpenTelemetry for observability

**Status:** Accepted  
**Date:** 2025-01

### Context
Distributed traces, structured logs, and metrics are required from day one.

### Decision
Instrument all modules with OpenTelemetry SDK. Export to Aspire Dashboard locally, and to Azure Monitor / Application Insights on Azure. No vendor-specific SDK calls in business code.

### Consequences
- Vendor-neutral instrumentation.
- Aspire provides a free local dashboard.
- App Insights as the Azure backend is cost-effective and well-integrated.

---

## ADR-009 — Two-level (L1 + L2) caching strategy: IMemoryCache + Redis

**Status:** Accepted  
**Date:** 2026-08

### Context

The platform uses `IMemoryCache` in `RepairOrdersEventConsumer` as an idempotency guard for Service Bus messages. This works correctly when running as a single process, but fails silently when scaled to two or more container instances on Azure Container Apps: each instance holds its own in-process cache, so duplicate messages delivered to different instances are not detected.

At the same time, replacing all `IMemoryCache` uses with Redis would introduce unnecessary network latency and operational overhead for use-cases that are purely local (feature flag reads, per-request short-lived data).

### Decision

Adopt a **two-level hybrid caching strategy**:

- **L1 — `IMemoryCache`** (in-process, zero-latency): used for data where cross-instance consistency is not required — query result caching, feature flag snapshots, per-request temporary data.
- **L2 — Redis (`IDistributedCache` via `StackExchange.Redis`)** (shared, cross-instance): used wherever correctness depends on all instances seeing the same state — idempotency guards, distributed rate limiting, user session data, notification deduplication.

The idempotency guard in `RepairOrdersEventConsumer` is migrated from `IMemoryCache` to Redis as the first concrete application of this rule.

For query caches a **read-through L1→L2** pattern is used: check L1 first (TTL ~30 s), on miss check L2 (TTL ~5 min), on miss query the database and populate both levels.

**Infrastructure:**
- Local: Redis emulator via Aspire (`AddRedis("redis").RunAsContainer()`)
- Azure: Azure Cache for Redis (Basic C0 for dev/test, Standard C1+ for production), accessed via Managed Identity

### Alternatives considered

| Option | Rejected because |
|---|---|
| Keep `IMemoryCache` everywhere | Correctness bug at ≥2 instances |
| Replace everything with Redis | Unnecessary latency for local-only caches; every cache read becomes a network call |
| Use session affinity (sticky sessions) on ACA | Fragile, ties scaling to routing; single instance failure drops all in-flight cache state |

### Consequences

**Positive:**
- Idempotency is correct regardless of how many Platform Host replicas run.
- Query caches remain fast (L1 hit = no network hop).
- Single Redis connection shared across all consumers in a process.
- Pattern is explicit and documented — easy to apply consistently to future features.

**Negative:**
- Redis is a new infrastructure dependency (local emulator + Azure resource + Bicep module).
- L1→L2 read-through logic adds complexity vs. a single cache abstraction.
- Redis TTL and L1 TTL must be coordinated — stale data window = L1 TTL.

### Implementation notes

- Redis connection registered once in `Program.cs` via `builder.AddRedis("redis")` (Aspire) or `builder.Services.AddStackExchangeRedisCache(...)`.
- `RepairOrdersEventConsumer` injects `IDistributedCache` instead of `IMemoryCache`.
- A `IHybridCache` wrapper (or `HybridCache` from `Microsoft.Extensions.Caching.Hybrid` in .NET 9+) may be introduced to unify L1+L2 access behind one abstraction in future.
- See [docs/19-caching-architecture.md](19-caching-architecture.md) for full diagrams.

---

## ADR-010 — FluentValidation as the single validation strategy

**Status:** Accepted
**Date:** 2026-08

### Context

Two approaches to request validation exist in ASP.NET Core: Data Annotations (declarative attributes on DTOs) and FluentValidation (explicit validator classes). The project needs a consistent strategy that is testable in isolation and integrates naturally with MediatR.

### Decision

Use **FluentValidation 12** as the exclusive validation library. Validators live in the Application layer next to their commands/queries and are executed automatically by `ValidationBehavior<TRequest, TResponse>` in the MediatR pipeline before any handler is invoked.

Data Annotations are only used for OpenAPI schema hints (e.g. `[Required]` on request DTOs for Scalar UI), never for validation logic.

### Consequences

- Validation errors are returned as `400 Bad Request` with a structured problem-details body before any handler executes.
- Validators are easily unit-tested in isolation with `TestValidate(...)`.
- No validator = no validation; the pipeline is opt-in per command/query.

---

## ADR-011 — Lightweight saga over workflow engine for RepairOrder lifecycle

**Status:** Accepted
**Date:** 2026-08

### Context

RepairOrder has a defined lifecycle: `Open → InProgress → Completed | Cancelled`. Tracking this lifecycle across multiple commands (Create, Start, Complete, Cancel) requires some form of long-running process coordination. Options include a full workflow engine (Temporal, Durable Functions, MassTransit Sagas) or a lightweight EF-backed state object.

### Decision

Implement a **lightweight manual saga** (`RepairSagaState`) persisted via EF Core in the RepairOrders database. The saga records state transitions triggered by domain commands:

| Command | Saga transition |
|---|---|
| `CreateRepairOrderCommand` | Creates `RepairSagaState` in `Created` |
| `StartRepairOrderCommand` | `Created → WorkStarted` |
| `CompleteRepairOrderCommand` | `WorkStarted → Completed` (compensates to `WorkStarted` first if saga is still `Created`) |
| `CancelRepairOrderCommand` | `* → Cancelled` (if not already completed) |

### Alternatives considered

| Option | Rejected because |
|---|---|
| MassTransit Sagas | Requires adopting MassTransit across the whole messaging layer — too large a dependency for the scope |
| Temporal / Durable Functions | External orchestration dependency; over-engineered for a three-state machine |
| Event sourcing | Adds significant infrastructure complexity before value is demonstrated |

### Consequences

- Saga state is queryable via EF Core — easy to inspect and test.
- No external dependency beyond EF Core + SQL Server.
- Compensating transition in `CompleteRepairOrderCommand` keeps the saga consistent even if `StartRepairOrderCommand` was never called (e.g. direct complete by administrator).
- Future: saga can be extended with timeout/reminder support via a background worker.

---

## ADR-012 — Service Bus resilience via Polly + explicit DLQ routing

**Status:** Accepted
**Date:** 2026-08

### Context

The outbox dispatcher and `RepairOrdersEventConsumer` communicate with Azure Service Bus over the network. Transient failures (network blips, Service Bus throttling) must not cause message loss or cascading failures. Additionally, non-retryable failures (deserialization errors, unknown event types) must be routed deterministically rather than silently dropped or retried indefinitely.

### Decision

**Resilience pipeline** (Polly `ResiliencePipeline`) applied to all Service Bus send/receive operations:
- **Retry:** 3 attempts, exponential back-off (base 2 s), random jitter ±20 %
- **Circuit breaker:** opens after 5 failures in a 30-second window; stays open for 15 s

**Explicit DLQ routing** in `RepairOrdersEventConsumer`:

| Scenario | Action |
|---|---|
| Transient exception | `AbandonMessageAsync` — Service Bus retries up to `MaxDeliveryCount` |
| Deserialization / null payload | `DeadLetterMessageAsync` with reason string |
| Unknown event type | `CompleteMessageAsync` + warning log (intentional discard) |
| `MaxDeliveryCount` exceeded | Service Bus automatically moves to DLQ |

**DeadLetterMonitorService** (`IHostedService`) polls the DLQ every 5 minutes and emits structured warning logs so Application Insights alerts can be configured.

### Consequences

- Transient faults are absorbed invisibly; throughput recovers automatically.
- The circuit breaker prevents a failing Service Bus namespace from exhausting thread-pool threads.
- Dead-lettered messages are observable without manual portal inspection.
- Polly `ResiliencePipeline` is registered as a singleton in DI and reused across all consumers/dispatchers in a process.

---

## ADR-013 — Feature-flagged authentication bypass for local development

**Status:** Accepted  
**Date:** 2025-06

### Context

The platform uses Microsoft Entra ID (JWT Bearer via `Microsoft.Identity.Web`) for authentication.
Running the application locally requires a valid Entra ID tenant, app registration, and token
acquisition flow, which creates friction for new contributors and CI environments that do not need
identity validation.

Additionally, the RepairOrders standalone microservice uses `RequireAuthorization(...)` on its
endpoints but originally had no authentication scheme registered, making it non-functional in any
environment.

### Decision

Introduce a `Features:DisableAuthentication` boolean flag in `appsettings.json`:

- When `false` (production default): `Microsoft.Identity.Web` JWT Bearer is registered and Entra ID
  validation is enforced.
- When `true` (local development default): a `LocalDevAuthHandler` is registered as the sole
  authentication scheme. It automatically authenticates every request as a synthetic principal that
  holds all App Roles and API scopes; no Entra ID credentials are required.

The flag is set in:

| File | Value |
|---|---|
| `appsettings.json` (base / production) | `false` |
| `appsettings.Development.json` (local) | `true` |
| Azure Container Apps `aca.bicep` env var | `false` (explicit, overrides any accidental config) |

The same `IdentityModuleExtensions` class is consumed by both the Platform Host and the RepairOrders
standalone service, ensuring consistent behavior.

### Consequences

- Local development requires no Entra ID setup; contributors can run and test all endpoints immediately.
- A loud `WARNING` log entry is emitted at startup when bypass is active, preventing accidental use in production.
- Azure deployments are explicitly protected by the `Features__DisableAuthentication=false` env var in `aca.bicep`, regardless of `appsettings.json` content.
- The bypass code is isolated in `LocalDevAuthHandler` and conditionally registered — it adds no overhead in production.
- Switching to full Entra ID locally (Option B) requires only User Secrets configuration and no code changes.
- The RepairOrders Service now correctly registers an authentication scheme before calling `UseAuthentication()` and `UseAuthorization()`.

---

## ADR-014 — Dual-provider AI assistant for Claims

**Status:** Accepted  
**Date:** 2025-06

### Context

Phase 18 adds an AI Case Assistant that generates a plain-text summary of an insurance claim on
demand (`POST /api/v1/claims/{id}/summarize`). Two operational contexts need to be supported:

- **Local development**: no Azure subscription required; use Ollama running on `localhost:11434`.
- **Production (Azure)**: use Azure OpenAI with `DefaultAzureCredential` (Managed Identity); no API keys.

The assistant must degrade gracefully when AI is unavailable or not configured.

### Decision

Register one of three `IClaimAiAssistant` implementations at startup based on configuration:

| `Features:AiAssistant:Enabled` | `AiAssistant:Provider` | Implementation registered |
|---|---|---|
| `false` (default) | any | `NullClaimAiAssistant` — placeholder, no AI call |
| `true` | `"Ollama"` | `OllamaClaimAiAssistant` — HTTP POST to Ollama |
| `true` | `"AzureOpenAI"` | `AzureOpenAiClaimAiAssistant` — Azure OpenAI SDK |

Provider selection happens once at startup in `ClaimsInfrastructureExtensions`. No runtime switching.

### Consequences

- Local development works without any Azure subscription or OpenAI account.
- Azure production uses Managed Identity — no API keys to rotate or leak.
- The `NullClaimAiAssistant` fallback ensures the claim workflow is never blocked by AI unavailability.
- Adding a new provider requires only a new `IClaimAiAssistant` implementation and a new `Provider` string — no changes to handlers or API layer.
- See [12-ai-option-later.md](12-ai-option-later.md) for the full implementation reference.

---

## ADR-015 -- MongoDB for the Documents bounded context

**Status:** Accepted
**Date:** 2025-07

### Context

Phase 19 introduces the Documents bounded context to manage **invoices** generated from completed
repair orders. Invoices contain a variable number of line items, each with independent tax rates
and discount rules that differ per repair type and customer contract.

Three persistence options were considered:

| Option | Pros | Cons |
|---|---|---|
| Azure SQL (EF Core) | Already in use; familiar tooling | Schema migrations every time line-item structure changes; EF join or JSON column needed |
| Azure SQL JSON column | No additional infrastructure | Ad-hoc querying via JSON_VALUE; poor type safety |
| **MongoDB** | Document-native fit; embedded line items; schema-less evolution | New infrastructure component; different query model |

### Decision

Use **MongoDB** (via `Aspire.MongoDB.Driver`) as the sole persistence store for the Documents
bounded context.

- The invoice aggregate is stored as a single BSON document in the `invoices` collection of the `documents` database.
- Line items are **embedded documents** -- no join needed for retrieval.
- Schema changes (new line-item fields) require no migration; the repository reads only fields it knows about.
- Locally, `builder.AddMongoDB("mongodb").WithDataVolume()` in `AppHost.cs` starts a MongoDB container managed by Aspire.
- In Azure, a MongoDB-compatible store (Azure Cosmos DB for MongoDB) is provisioned by Bicep; the connection string is injected as a named connection `mongodb` via Key Vault / App Settings.

The `IMongoClient` is registered as `AddSingleton` built directly from the named connection
string, keeping the registration independent of the Aspire SDK path for production deployments.

### Consequences

- Invoices survive schema evolution without EF migrations.
- A second database technology is now in the platform (SQL + MongoDB); operators must be aware of both when planning backups and monitoring.
- The Documents module is self-contained -- it never reads from or writes to any SQL schema.
- Adding MongoDB to `AppHost` is the only infrastructure change needed for local development.
- Tests mock `IInvoiceRepository`; no MongoDB process is required for the unit/application test suite.
