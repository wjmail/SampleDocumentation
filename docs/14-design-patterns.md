# 14 — Design Patterns

This document describes every architectural and design pattern applied in the VehicleLifecycle codebase, with the location of concrete implementations.

---

## 1. Domain-Driven Design (DDD)

The codebase is structured around **Bounded Contexts** — each module is an autonomous domain with its own ubiquitous language, model, and persistence.

### 1.1 Aggregate Root

All domain entities that own a consistency boundary extend `AggregateRoot<TId>` from `VehicleLifecycle.SharedKernel`.

```mermaid
classDiagram
	class AggregateRoot~TId~ {
		+TId Id
		-List~IDomainEvent~ _domainEvents
		+IReadOnlyList~IDomainEvent~ DomainEvents
		+RaiseDomainEvent(IDomainEvent)
		+ClearDomainEvents()
	}
	class Claim {
		+ClaimId Id
		+string PolicyNumber
		+ClaimStatus Status
		+RaiseClaimSubmittedEvent()
	}
	class RepairOrder {
		+RepairOrderId Id
		+string CustomerId
		+RepairOrderStatus Status
		+AddLineItem()
		+Complete()
	}
	class Part {
		+PartId Id
		+PartNumber PartNumber
		+int StockQuantity
		+AdjustStock()
	}
	AggregateRoot~TId~ <|-- Claim
	AggregateRoot~TId~ <|-- RepairOrder
	AggregateRoot~TId~ <|-- Part
```

**Location:** `src/BuildingBlocks/VehicleLifecycle.SharedKernel/Domain/AggregateRoot.cs`

### 1.2 Value Object

Immutable domain concepts with structural equality. No identity, compared by value.

| Value Object | Module | Properties |
|---|---|---|
| `ClaimId` | Claims | `Guid Value` |
| `RepairOrderId` | RepairOrders | `Guid Value` |
| `PartId` | Parts | `Guid Value` |
| `PartNumber` | Parts | `string Value` |
| `Money` | Parts | `decimal Amount`, `string Currency` |

**Base class:** `src/BuildingBlocks/VehicleLifecycle.SharedKernel/Domain/ValueObject.cs`

```csharp
// Example: structural equality via GetEqualityComponents()
public sealed class Money : ValueObject
{
	public decimal Amount { get; }
	public string Currency { get; }
	protected override IEnumerable<object?> GetEqualityComponents()
		=> [Amount, Currency];
}
```

### 1.3 Domain Events

Business facts raised inside aggregates and dispatched asynchronously via the outbox.

```mermaid
graph LR
	AGG[Aggregate Root] -->|RaiseDomainEvent| DE[Domain Event]
	DE -->|intercepted by EF SaveChanges| OBI[DomainEventsToOutboxInterceptor]
	OBI -->|serialize to JSON| OUTBOX[(OutboxMessage table)]
	OUTBOX -->|polled by| ODS[OutboxDispatcherService]
	ODS -->|publish| SB[Service Bus]
```

**Location per module:**
- `src/Modules/Claims/VehicleLifecycle.Claims.Domain/Events/`
- `src/Modules/RepairOrders/VehicleLifecycle.RepairOrders.Domain/Events/`
- `src/Modules/Parts/VehicleLifecycle.Parts.Domain/Events/`

### 1.4 Repository Pattern

Each module defines a repository interface in the Application layer and implements it in Infrastructure with EF Core. Modules depend only on the interface — no EF Core reference in Domain or Application.

```mermaid
classDiagram
	class IRepository~TAggregate,TId~ {
		<<interface>>
		+GetByIdAsync(TId) Task~TAggregate~
		+AddAsync(TAggregate) Task
		+UpdateAsync(TAggregate) Task
		+DeleteAsync(TAggregate) Task
	}
	class IClaimRepository {
		<<interface>>
	}
	class EfClaimRepository {
		-ClaimsDbContext _db
	}
	IRepository <|-- IClaimRepository
	IClaimRepository <|.. EfClaimRepository
```

---

## 2. CQRS — Command Query Responsibility Segregation

Every write operation is a **Command** and every read is a **Query**. Both are dispatched via MediatR.

```mermaid
graph LR
	EP[Minimal API Endpoint] -->|send| MED[MediatR IMediator]
	MED -->|route| CH[CommandHandler]
	MED -->|route| QH[QueryHandler]
	CH -->|write via| REPO[IRepository]
	QH -->|read via| DB[(DbContext)]
	CH -->|return| DTO[Result / Id]
	QH -->|return| DTO2[DTO]
```

**Commands (examples):**

| Command | Module | Handler |
|---|---|---|
| `SubmitClaimCommand` | Claims | `SubmitClaimCommandHandler` |
| `CreateRepairOrderCommand` | RepairOrders | `CreateRepairOrderCommandHandler` |
| `RegisterPartCommand` | Parts | `RegisterPartCommandHandler` |
| `AdjustStockCommand` | Parts | `AdjustStockCommandHandler` |

**Queries (examples):**

| Query | Module | Returns |
|---|---|---|
| `GetClaimQuery` | Claims | `ClaimDto` |
| `GetRepairOrderQuery` | RepairOrders | `RepairOrderDto` |
| `GetPartQuery` | Parts | `PartDto` |
| `ListPartsByCategoryQuery` | Parts | `IEnumerable<PartDto>` |

---

## 3. Transactional Outbox

Guarantees **at-least-once delivery** of domain events to Service Bus without distributed transactions.

```mermaid
sequenceDiagram
	participant App as Application Layer
	participant DB as SQL Database
	participant ODB as outbox_messages table
	participant ODS as OutboxDispatcherService
	participant SB as Azure Service Bus

	App->>DB: SaveChangesAsync()
	Note over DB,ODB: EF Interceptor serializes<br/>domain events → OutboxMessage rows<br/>in the SAME transaction
	DB-->>App: committed

	loop every N seconds
		ODS->>ODB: SELECT unprocessed messages
		ODS->>SB: send integration event
		SB-->>ODS: ack
		ODS->>ODB: mark processed
	end
```

**Key files:**
- `DomainEventsToOutboxInterceptor.cs` — EF SaveChanges interceptor
- `OutboxMessage.cs` — persisted record (Id, Type, Payload, ProcessedAt)
- `OutboxDispatcherService.cs` — `IHostedService` background worker

---

## 4. Mediator Pattern

MediatR decouples request senders from handlers. All cross-cutting concerns (validation, logging, correlation) are added as **pipeline behaviors**.

```mermaid
graph TB
	SND[Endpoint / Service] -->|IMediator.Send| PB1[LoggingBehavior]
	PB1 --> PB2[ValidationBehavior]
	PB2 --> H[IRequestHandler]
	H --> PB2
	PB2 --> PB1
	PB1 --> SND
```

---

## 5. Null Object Pattern — Feature Flags

AI functionality is implemented via a `NullClaimAiAssistant` that returns a placeholder response when the feature flag `Features:AiAssistant:Enabled = false`. This keeps the endpoint wired and testable without requiring a live AI service.

```mermaid
classDiagram
	class IClaimAiAssistant {
		<<interface>>
		+SummarizeClaimAsync(Guid) Task~string~
	}
	class NullClaimAiAssistant {
		+SummarizeClaimAsync(Guid) Task~string~
		note: returns placeholder,\nlogs "AI disabled"
	}
	class AzureOpenAiClaimAiAssistant {
		+SummarizeClaimAsync(Guid) Task~string~
		note: real implementation\n(not yet wired)
	}
	IClaimAiAssistant <|.. NullClaimAiAssistant
	IClaimAiAssistant <|.. AzureOpenAiClaimAiAssistant
```

Registration controlled by:
```json
"Features": { "AiAssistant": { "Enabled": false } }
```

---

## 6. Two-Level Hybrid Cache (L1 + L2)

The platform applies a **hybrid caching strategy** that assigns each use-case to the appropriate cache tier based on a single correctness question: *must all running instances agree on this value?*

```mermaid
flowchart TD
	Q{Cross-instance\nconsistency required?}
	Q -->|Yes| REDIS[L2 — Redis\nIDistributedCache]
	Q -->|No| MEM[L1 — IMemoryCache\nin-process]

	REDIS --> UC1[Idempotency guard\nService Bus consumer]
	REDIS --> UC2[User session cache]
	REDIS --> UC3[Notification deduplication]
	MEM --> UC4[Feature flag snapshot]
	MEM --> UC5[L1 layer in query cache]
```

### 6.1 Idempotency Guard — Redis (L2 only)

`RepairOrdersEventConsumer` deduplicates Service Bus messages by `MessageId` using **Redis only** (no L1). This guarantees exactly-once processing even when multiple Platform Host replicas receive the same message from different delivery attempts.

```mermaid
flowchart TD
	MSG[Service Bus Message] --> CHK{MessageId\nin Redis?}
	CHK -->|Yes — TTL 24h| SKIP[Complete message\n— already processed]
	CHK -->|No| PROC[Process message]
	PROC --> CACHE[SetAsync in Redis\nTTL 24h]
	PROC -->|error| DLQ[Dead-Letter\nor Abandon]
```

### 6.2 Query Cache — L1 → L2 Read-Through

Frequently-read query results (e.g. parts catalogue lists) use a **read-through** pattern: L1 is checked first (TTL ~30 s), then L2 Redis (TTL ~5 min), then the database. Both levels are populated on a miss.

```mermaid
sequenceDiagram
	participant REQ as Request
	participant L1 as IMemoryCache (L1)
	participant L2 as Redis (L2)
	participant DB as SQL Database

	REQ->>L1: TryGetValue(key)
	alt L1 hit (~30s TTL)
		L1-->>REQ: cached value
	else L1 miss
		REQ->>L2: GetAsync(key)
		alt L2 hit (~5min TTL)
			L2-->>REQ: value
			REQ->>L1: Set(key, 30s)
		else L2 miss
			REQ->>DB: query
			DB-->>REQ: result
			REQ->>L2: SetAsync(key, 5min)
			REQ->>L1: Set(key, 30s)
		end
	end
```

**See also:** [ADR-009](05-architecture-decisions.md#adr-009) · [docs/19-caching-architecture.md](19-caching-architecture.md)

---

## 7. Correlation ID Propagation

Every HTTP request receives a `X-Correlation-Id` header (generated if absent). The ID is stored in `ICorrelationIdAccessor`, injected into log scopes, and forwarded downstream via OpenTelemetry baggage.

```mermaid
sequenceDiagram
	participant Client
	participant MW as CorrelationIdMiddleware
	participant Handler
	participant OTel as OpenTelemetry

	Client->>MW: HTTP request [X-Correlation-Id: abc]
	MW->>MW: extract or generate correlation ID
	MW->>OTel: add to Activity baggage
	MW->>Handler: forward request
	Handler-->>Client: HTTP response [X-Correlation-Id: abc]
```

---

## 8. Options Pattern (Configuration)

Strongly-typed configuration sections bound via `IOptions<T>`. No raw `IConfiguration` access in domain or application layers.

| Options Class | Section | Used by |
|---|---|---|
| `AiAssistantOptions` | `Features:AiAssistant` | Claims Infrastructure |
| `ServiceBusOptions` | `ConnectionStrings:servicebus` | Injected by Aspire |
| `SqlOptions` | `ConnectionStrings:*-db` | Injected by Aspire |

---

## 9. Assembly Marker Pattern

Each module exposes an empty marker type (`*AssemblyMarker`) used by MediatR registration to locate handlers without hardcoding assembly names.

```csharp
// Usage in module extension:
services.AddMediatR(cfg =>
	cfg.RegisterServicesFromAssemblyContaining<ClaimsApplicationAssemblyMarker>());
```

---

## 10. Vertical Slice / Modular Monolith

Each bounded context is a self-contained vertical slice with its own:
- Domain model (no cross-module references)
- Application layer (CQRS handlers)
- Infrastructure (separate `DbContext`, own connection string)
- API surface (endpoint registration extension)

Cross-module communication is only allowed via **integration events on Service Bus** — never direct in-process calls between module internals.

---

## 11. MediatR Pipeline Behaviors

Cross-cutting concerns are applied to every MediatR request via `IPipelineBehavior<TRequest, TResponse>`. All behaviors live in `VehicleLifecycle.SharedKernel.Behaviors` and are registered in the shared `AddSharedKernel(...)` extension.

```mermaid
sequenceDiagram
	participant Client
	participant LOG as LoggingBehavior
	participant PERF as PerformanceBehavior
	participant VAL as ValidationBehavior
	participant HAND as Handler

	Client->>LOG: Send(request)
	LOG->>PERF: next()
	PERF->>VAL: next()
	VAL->>VAL: Run FluentValidation
	alt validation errors
		VAL-->>Client: ValidationException (→ 400)
	else valid
		VAL->>HAND: next()
		HAND-->>VAL: response
		VAL-->>PERF: response
		PERF->>PERF: log if elapsed > 500 ms
		PERF-->>LOG: response
		LOG->>LOG: log exit + elapsed
		LOG-->>Client: response
	end
```

| Behavior | Responsibility |
|---|---|
| `LoggingBehavior` | Structured log on entry and exit with elapsed time |
| `PerformanceBehavior` | Warning log when handler exceeds 500 ms |
| `ValidationBehavior` | Runs all `IValidator<TRequest>` implementations; throws `ValidationException` on failure |

**Location:** `src/BuildingBlocks/VehicleLifecycle.SharedKernel/Behaviors/`

---

## 12. Lightweight Saga (RepairOrder lifecycle)

A **manually-driven saga** tracks the RepairOrder lifecycle across multiple commands without a workflow engine.

```mermaid
stateDiagram-v2
	[*] --> Created : CreateRepairOrderCommand
	Created --> WorkStarted : StartRepairOrderCommand
	WorkStarted --> Completed : CompleteRepairOrderCommand
	Created --> Completed : CompleteRepairOrderCommand\n(compensating – skips WorkStarted)
	Created --> Cancelled : CancelRepairOrderCommand
	WorkStarted --> Cancelled : CancelRepairOrderCommand
	Completed --> [*]
	Cancelled --> [*]
```

`RepairSagaState` is an EF Core entity stored in `repairorders-db`. Each command handler loads the saga by `RepairOrderId`, calls the appropriate transition method, and saves.

**Location:** `src/Modules/RepairOrders/VehicleLifecycle.RepairOrders.Application/Saga/`

**See also:** [ADR-011](05-architecture-decisions.md#adr-011)

---

## 13. Resilience Pipeline (Service Bus)

All Service Bus send/receive operations are wrapped in a **Polly `ResiliencePipeline`** registered as a singleton in DI.

```mermaid
flowchart LR
	OP[Service Bus operation] --> CB{Circuit\nBreaker}
	CB -->|closed| RETRY[Retry up to 3×\nexponential back-off]
	RETRY -->|success| OK[✓ Result]
	RETRY -->|all attempts failed| EX[throw]
	CB -->|open 15 s| EX2[BrokenCircuitException]
```

| Strategy | Configuration |
|---|---|
| Retry | 3 attempts, base 2 s, exponential, ±20 % jitter |
| Circuit breaker | 5 failures / 30 s window → open 15 s |

**Location:** `src/BuildingBlocks/VehicleLifecycle.SharedKernel/Resilience/`

**See also:** [ADR-012](05-architecture-decisions.md#adr-012)

---

## 14. Rate Limiting

A fixed-window rate limiting policy `api` is applied to all public endpoints in Platform.Host and RepairOrders.Service.

| Setting | Value |
|---|---|
| Window | 60 seconds |
| Permit limit | 100 requests |
| Partition key | Client IP address |
| Response on exceed | `429 Too Many Requests` |

Endpoints are decorated with `RequireRateLimiting("api")`. The policy is registered via `builder.Services.AddRateLimiter(...)` in each host's `Program.cs`.
