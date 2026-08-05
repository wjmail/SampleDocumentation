# 21 — Debugging Guide

> How to trace every request, command, domain event, and message through the system —
> step by step, from HTTP call to notification.

---

## Prerequisites

- Application running via AppHost (see [18-local-development.md](18-local-development.md))
- Visual Studio 2026 with the solution `src/VehicleLifecycle.Platform.slnx` open
- Docker running

---

## The Full Data and Action Flow

Every user action travels through these layers in order:

```
HTTP Request
	↓
ASP.NET Core Endpoint (Minimal API)
	↓
MediatR Command / Query
	↓
Pipeline Behaviors (validation → audit → exception)
	↓
Application Handler
	↓
Domain Aggregate (business logic + domain events)
	↓
Infrastructure: EF Core saves to database
	↓
Outbox Interceptor captures domain events
	↓
Outbox Dispatcher publishes Integration Events to Service Bus
	↓
RepairOrdersEventConsumer receives message
	↓
INotificationService sends notification
```

Understanding this flow is the foundation for every debugging session.

---

## Breakpoint Strategy — Repair Order Creation

Use this concrete example to learn the pattern and then apply it to any operation.

### 1. HTTP → Endpoint

**File:** `src/Modules/RepairOrders/VehicleLifecycle.RepairOrders.Api/RepairOrdersModuleExtensions.cs`

Find the `POST /api/v1/repair-orders` route registration and set a breakpoint on:

```csharp
.MapPost("/", async (CreateRepairOrderCommand command, ISender sender, ...) =>
	await sender.Send(command)   // ← breakpoint here
```

- `command` contains the deserialized HTTP body
- `sender.Send(command)` routes to MediatR

### 2. MediatR → Command Handler

**File:** `src/Modules/RepairOrders/VehicleLifecycle.RepairOrders.Application/Commands/CreateRepairOrderCommandHandler.cs`

Set a breakpoint at the top of `Handle(...)`:

```csharp
public async Task<Guid> Handle(CreateRepairOrderCommand request, CancellationToken ct)
// ← breakpoint here
```

**What to inspect:**
- `request` — the full command with all fields
- Step into domain aggregate construction to see the invariants being checked

### 3. Domain Aggregate (DDD)

**File:** `src/Modules/RepairOrders/VehicleLifecycle.RepairOrders.Domain/Aggregates/RepairOrder.cs`

Set a breakpoint in the factory method or constructor:

```csharp
public static RepairOrder Create(CustomerId customerId, Vin vehicleVin, ServiceType serviceType)
// ← breakpoint here
```

**What to inspect:**
- Business rules being enforced (e.g., VIN format, non-null customer)
- `_domainEvents` list — watch it grow as events are raised

Domain events are raised by the aggregate via:

```csharp
AddDomainEvent(new RepairOrderCreatedDomainEvent(Id, CustomerId, VehicleVin));
```

Set a breakpoint here to confirm the event is raised before any persistence.

### 4. EF Core Save + Outbox Interceptor

**File:** `src/BuildingBlocks/VehicleLifecycle.SharedKernel/Outbox/OutboxInterceptor.cs`

Set a breakpoint in `SavingChangesAsync`:

```csharp
public override async ValueTask<InterceptionResult<int>> SavingChangesAsync(...)
// ← breakpoint here
```

**What to inspect:**
- `eventEntities` — all aggregates with pending domain events
- After the interceptor runs, the `OutboxMessages` table in SQL will contain a new row

**Verify in SQL:**

```sql
USE RepairOrdersDb;
SELECT TOP 10 * FROM OutboxMessages ORDER BY CreatedAt DESC;
```

### 5. Outbox → Service Bus Dispatch

**File:** `src/BuildingBlocks/VehicleLifecycle.SharedKernel/Outbox/OutboxDispatcher.cs`

The dispatcher polls `OutboxMessages` every few seconds. Set a breakpoint in `ExecuteAsync`
or on the `SendMessageAsync` call:

```csharp
await sender.SendMessageAsync(new ServiceBusMessage(serialized));
// ← breakpoint here
```

**What to inspect:**
- `message.Subject` — matches the event type name (e.g., `RepairOrderCreatedIntegrationEvent`)
- `message.Body` — the serialized JSON payload

### 6. Service Bus Consumer (Platform Host)

**File:** `src/Platform/VehicleLifecycle.Platform.Host/RepairOrdersEventConsumer.cs`

Set a breakpoint in `HandleMessageAsync`:

```csharp
private async Task HandleMessageAsync(ProcessMessageEventArgs args)
// ← breakpoint here
```

**What to inspect:**
- `args.Message.Subject` — the event type routing key
- `args.Message.Body.ToString()` — the raw JSON
- The `switch` statement shows which handler method will be called

Then follow into:

```csharp
private async Task NotifyRepairOrderCreatedAsync(ServiceBusReceivedMessage message)
// ← breakpoint here
```

**What to inspect:**
- `integrationEvent` — deserialized event
- `_cache.GetAsync(idempotencyKey)` — will be `null` on first delivery (not a duplicate)

### 7. Notification

**File:** `src/Modules/Notifications/VehicleLifecycle.Notifications.Infrastructure/LoggingNotificationService.cs`

Set a breakpoint on the `logger.LogInformation(...)` call inside `NotifyRepairOrderCreatedAsync`.

**What to inspect:**
- The message that would be sent to the customer
- The `customerId` — this is currently used as a display string; production code would resolve it
  to an email address

---

## Debugging State Transitions

Every `PUT .../start`, `PUT .../complete`, `PUT .../cancel` follows the same path:

```
PUT endpoint → MediatR Command → Handler → Aggregate.Start() / .Complete() / .Cancel()
	↓
Domain event raised (e.g., RepairOrderWorkStartedDomainEvent)
	↓
EF Core save → Outbox row created
	↓
Outbox dispatcher → ServiceBusMessage with Subject = "RepairOrderWorkStartedIntegrationEvent"
	↓
Consumer routes to NotifyRepairOrderWorkStartedAsync
	↓
LoggingNotificationService.NotifyRepairOrderWorkStartedAsync logs the message
```

**Key aggregate method breakpoints:**

| Command | Aggregate method | Domain event raised |
|---|---|---|
| Start work | `RepairOrder.Start()` | `RepairOrderWorkStartedDomainEvent` |
| Complete | `RepairOrder.Complete()` | `RepairOrderCompletedDomainEvent` |
| Cancel | `RepairOrder.Cancel(reason)` | `RepairOrderCancelledDomainEvent` |

---

## Debugging Claims

### Submit Claim flow

```
POST /api/v1/claims
	↓
SubmitClaimCommand → SubmitClaimCommandHandler
	↓
Claim.Submit(customerId, vehicleId, description, damageType)
	↓
ClaimSubmittedDomainEvent raised
	↓
EF Core → OutboxMessages in ClaimsDb
```

**Key files:**

| Layer | File path |
|---|---|
| Endpoint | `src/Modules/Claims/VehicleLifecycle.Claims.Api/ClaimsModuleExtensions.cs` |
| Handler | `src/Modules/Claims/VehicleLifecycle.Claims.Application/Commands/SubmitClaim/SubmitClaimCommandHandler.cs` |
| Aggregate | `src/Modules/Claims/VehicleLifecycle.Claims.Domain/Aggregates/Claim.cs` |
| Infrastructure | `src/Modules/Claims/VehicleLifecycle.Claims.Infrastructure/` |

---

## Debugging the AI Summary (Claims)

Prerequisite: `Features:AiAssistant:Enabled = true` (the default in Development).

```
POST /api/v1/claims/{id}/summarize
	↓
SummarizeClaimQuery → SummarizeClaimQueryHandler
	↓
IAiAssistantService.SummarizeAsync(claim)
	↓
OllamaAiAssistantService or AzureOpenAiAssistantService
	↓
HTTP call to Ollama at http://localhost:11434 (or Azure OpenAI)
```

**If the AI call fails:** the endpoint returns `503 Service Unavailable` with a message
explaining the provider is unreachable. Check `Features:AiAssistant:Provider` in
`appsettings.Development.json` (`Ollama` or `AzureOpenAI`).

---

## Using Visual Studio Diagnostic Tools

While stopped at a breakpoint, open the **Diagnostic Tools** window
(**Debug → Windows → Diagnostic Tools**):

| Tab | What to use it for |
|---|---|
| **Events** | See GC, exceptions, and breakpoint hits on a timeline |
| **Memory Usage** | Take a snapshot before / after a handler to check allocations |
| **CPU Usage** | Use "Record CPU Profile" to identify slow code paths |

---

## Using the Aspire Dashboard for Distributed Tracing

1. Open http://localhost:18888
2. Click **Traces** in the left sidebar
3. Find the trace for your recent request by:
   - Filtering by service name (`platform-host` or `repairorders-service`)
   - Looking at the timestamp
4. Click on a trace to expand the full span tree:
   - `POST /api/v1/repair-orders` (HTTP span)
   - `SubmitRepairOrderCommandHandler.Handle` (if custom activity is present)
   - `repairordersdb` (SQL span — shows the exact database call)
   - `service-bus.send` (outgoing Service Bus publish span)
5. Cross-trace: the `TraceId` is propagated through Service Bus headers so you can
   follow the same logical operation from the RepairOrders Service into the Platform Host consumer

---

## Inspecting the Outbox Database

If you suspect the outbox is stuck (messages not being dispatched):

```sql
-- Pending messages (not yet sent)
SELECT * FROM OutboxMessages WHERE ProcessedAt IS NULL ORDER BY CreatedAt;

-- Recently processed
SELECT TOP 20 * FROM OutboxMessages WHERE ProcessedAt IS NOT NULL ORDER BY ProcessedAt DESC;

-- Error messages
SELECT * FROM OutboxMessages WHERE Error IS NOT NULL;
```

**Connection strings (local):**

| Database | Connection |
|---|---|
| `ClaimsDb` | `Server=localhost,1433;Database=ClaimsDb;User Id=sa;Password=...` |
| `RepairOrdersDb` | `Server=localhost,1433;Database=RepairOrdersDb;User Id=sa;Password=...` |
| `PartsDb` | `Server=localhost,1433;Database=PartsDb;User Id=sa;Password=...` |

The actual `sa` password is printed by Aspire in the console at startup and available in
the Aspire Dashboard under **Resources → sql → Connection Strings**.

---

## Debugging MediatR Pipeline Behaviors

Three behaviors run on every command/query:

| Order | Behavior | What it does |
|---|---|---|
| 1 | `ValidationBehavior<,>` | Runs FluentValidation; throws `ValidationException` on failure |
| 2 | `AuditBehavior<,>` | Logs the command name and duration |
| 3 | `ExceptionBehavior<,>` | Catches domain exceptions and converts to problem details |

**To debug validation failures:**

Set a breakpoint in `ValidationBehavior.Handle`:

```csharp
src/BuildingBlocks/VehicleLifecycle.SharedKernel/Behaviors/ValidationBehavior.cs
```

Inspect `failures` to see exactly which fields failed and why.

---

## Common Debugging Scenarios

### "My request returned 400 but I don't know why"

1. Open Aspire Dashboard → **Logs** → filter by `platform-host` or `repairorders-service`
2. Search for `Validation` or the correlation ID from the response header `X-Correlation-Id`
3. The log entry will contain the field names and validation messages

### "The notification was not received"

1. Check if the outbox message was created:
   ```sql
   SELECT * FROM OutboxMessages WHERE ProcessedAt IS NULL;
   ```
2. Check if the Service Bus message was sent:
   Aspire Dashboard → **Traces** → look for `service-bus.send` span
3. Check the dead-letter queue:
   Aspire Dashboard → **Logs** → search for `DeadLetterMonitor`
4. Check the consumer received and processed the message:
   Aspire Dashboard → **Logs** → search for `RepairOrdersEventConsumer`

### "My domain aggregate threw an exception"

Domain exceptions are caught by `ExceptionBehavior` and converted to:
- `400 Bad Request` — business rule violation (e.g., cannot complete an already-completed order)
- `404 Not Found` — aggregate not found

The exception message is returned in the `ProblemDetails.detail` field.
Set a breakpoint in `ExceptionBehavior.Handle` to see the raw exception before it is mapped.

### "I want to re-process a dead-lettered message"

Currently `DeadLetterMonitorService` only logs DLQ messages — it does not re-enqueue them.
To manually re-process:

1. Use Azure Service Bus Explorer (see [22-monitoring-guide.md](22-monitoring-guide.md))
2. Find the message in `repair-orders/subscriptions/platform-host/$deadletterqueue`
3. Inspect `DeadLetterReason` and `DeadLetterErrorDescription` properties
4. Fix the underlying cause, then re-send the message body as a new `ServiceBusMessage`

---

## Related Documents

- [20-application-guide.md](20-application-guide.md) — what the app does and how to use it
- [22-monitoring-guide.md](22-monitoring-guide.md) — monitoring, queues, metrics
- [11-operability-and-observability.md](11-operability-and-observability.md) — OpenTelemetry, logging standards
- [18-local-development.md](18-local-development.md) — local setup and prerequisites
- [04-domain-model.md](04-domain-model.md) — domain model and bounded contexts
