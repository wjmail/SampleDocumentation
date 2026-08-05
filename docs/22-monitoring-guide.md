# 22 — Monitoring Guide

> Where to look and what to look for — queues, endpoints, performance, data, notifications,
> and everything in between.

---

## Central Tool: Aspire Dashboard

The Aspire Dashboard is available at **http://localhost:18888** when running locally.
It provides a single interface for:

- **Resources** — health status of all services and containers
- **Logs** — structured log output from every service, searchable
- **Traces** — distributed traces showing the full call graph
- **Metrics** — live counters and histograms for HTTP, database, and runtime

> Open this first before checking anything else. Most issues can be identified here
> without opening a database client or terminal.

---

## Monitoring Service Health

### Aspire Dashboard → Resources

| Column | Meaning |
|---|---|
| State: Running (green) | Service started and health check passed |
| State: Starting (yellow) | Container or process not yet ready |
| State: Error (red) | Check the Logs tab for this resource |

### Health Endpoints

All services expose two health endpoints:

```
GET /health        → liveness  (always 200 if process is alive)
GET /health/ready  → readiness (200 if databases and dependencies are reachable)
```

Test them directly:

```powershell
# Platform Host
Invoke-RestMethod http://localhost:5001/health
Invoke-RestMethod http://localhost:5001/health/ready

# RepairOrders Service
Invoke-RestMethod http://localhost:5002/health
Invoke-RestMethod http://localhost:5002/health/ready
```

A `503` response from `/health/ready` means at least one dependency (SQL, Service Bus, Redis)
is unreachable. Check the response body for the specific failing check name.

---

## Monitoring the Service Bus Queue

The Service Bus Emulator runs locally as a Docker container. There are two ways to inspect it.

### Option A: Aspire Dashboard → Traces

1. Open **Traces** tab
2. Filter by `platform-host` (the consumer) or `repairorders-service` (the publisher)
3. Look for spans named `ServiceBusSender.Send` (publishing) and
   `ServiceBusProcessor.ProcessMessage` (consuming)
4. Expand the trace to see message subject, success/failure, and duration

### Option B: Azure Service Bus Explorer

Azure Service Bus Explorer is a free GUI tool that connects to the local emulator.

1. Download: https://github.com/paolosalvatori/ServiceBusExplorer/releases
2. Connect using the emulator connection string (printed in Aspire Dashboard → Resources → servicebus → Connection Strings)
3. Browse topics and subscriptions:
   - `repair-orders` topic → `platform-host` subscription
   - `parts` topic

**Key views:**

| View | What to check |
|---|---|
| Active messages | Normal queue depth (should be near 0 in steady state) |
| Dead-letter queue (DLQ) | Poisoned messages that failed max delivery attempts |
| Message browser | Inspect individual message body, subject, and custom properties |

### Dead-Letter Queue (DLQ)

Path: `repair-orders/subscriptions/platform-host/$deadletterqueue`

A message ends up in the DLQ when:
- The consumer explicitly dead-letters it (unknown event type, or exceeds retry threshold)
- The max delivery count (default: 10) is exceeded

The `DeadLetterMonitorService` polls the DLQ every 30 seconds and logs each message.
Search for it in Aspire Dashboard → Logs:

```
DeadLetterMonitor
```

Each log entry includes: `Subject`, `MessageId`, `DeadLetterReason`, `DeadLetterErrorDescription`.

---

## Monitoring API Endpoints

### Aspire Dashboard → Traces

Every HTTP request generates a trace. You can:
- Filter by service name to see only Platform Host or RepairOrders traffic
- Sort by duration to find slow requests
- Click a trace to expand the full span tree (HTTP → handler → SQL → Service Bus)

### Request Logs

Aspire Dashboard → Logs → select service → search for:

```
Request received     # entry log from correlation ID middleware
Request finished     # exit log with status code and duration
```

### Rate Limiting Monitoring

If the `429 Too Many Requests` rate is high:
- Aspire Dashboard → Logs → search `RateLimiter` or `429`
- Aspire Dashboard → Metrics → look for `aspnetcore.http.server.request.duration` with status `429`

---

## Monitoring Performance

### Aspire Dashboard → Metrics

Key metrics to watch:

| Metric name | What it measures | Alert threshold (suggestion) |
|---|---|---|
| `http.server.request.duration` | HTTP response time | p99 > 500 ms |
| `db.client.operation.duration` | SQL query time | p99 > 100 ms |
| `dotnet.gc.collections` | Garbage collection frequency | Frequent Gen2 = memory pressure |
| `dotnet.process.cpu.time` | CPU consumption | Sustained > 80% |

### Slow SQL Queries

**Aspire Dashboard → Traces** — look for `db.statement` spans with high duration.

Alternatively, connect to SQL Server directly:

```sql
-- Top 10 slowest queries by average CPU time
SELECT TOP 10
	total_elapsed_time / execution_count AS avg_elapsed_ms,
	execution_count,
	SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
		((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
		  ELSE qs.statement_end_offset END - qs.statement_start_offset)/2)+1) AS statement_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY avg_elapsed_ms DESC;
```

---

## Monitoring Data

### Checking Database Content

**Connection:** use SQL Server Management Studio, Azure Data Studio, or any SQL client.

Connection details are in Aspire Dashboard → Resources → sql → Connection Strings.

#### Repair Orders

```sql
USE RepairOrdersDb;
SELECT Id, CustomerId, VehicleVin, ServiceType, Status, CreatedAt, UpdatedAt
FROM RepairOrders
ORDER BY CreatedAt DESC;
```

#### Outbox (message relay)

```sql
-- Pending messages (not yet dispatched to Service Bus)
SELECT * FROM OutboxMessages WHERE ProcessedAt IS NULL ORDER BY CreatedAt;

-- Recently dispatched
SELECT TOP 20 * FROM OutboxMessages WHERE ProcessedAt IS NOT NULL ORDER BY ProcessedAt DESC;

-- Failed dispatches
SELECT * FROM OutboxMessages WHERE Error IS NOT NULL;
```

> Pending messages older than ~30 seconds indicate the Outbox Dispatcher background service
> is not running or cannot reach the Service Bus emulator.

#### Claims

```sql
USE ClaimsDb;
SELECT Id, CustomerId, VehicleId, Status, DamageType, CreatedAt
FROM Claims
ORDER BY CreatedAt DESC;
```

#### Parts + stock

```sql
USE PartsDb;
SELECT Id, PartNumber, Name, Category, UnitPrice, StockQuantity
FROM Parts
ORDER BY Name;
```

### Redis (Idempotency Cache)

Redis is used to prevent the consumer from processing the same Service Bus message twice.

To inspect Redis, connect using the Redis CLI or Redis Insight:

```powershell
# Get the Redis port from Aspire Dashboard → Resources → redis → Connection Strings
# then connect:
redis-cli -p <port>

# List idempotency keys
KEYS "idempotency:*"

# Check a specific key (value is "1" if already processed)
GET "idempotency:<messageId>"
```

An idempotency key expires after 24 hours (controlled in `RepairOrdersEventConsumer.cs`).

---

## Monitoring Notifications

Notifications are currently implemented as log entries (stub). All notification output is
visible in the Aspire Dashboard logs.

### Finding notification log entries

Aspire Dashboard → Logs → select `platform-host` → search:

```
[Notification]
```

Each triggered notification produces a structured log entry:

| Event | Log message pattern |
|---|---|
| Repair Order Created | `[Notification] RepairOrder {id} created for customer {customerId}` |
| Work Started | `[Notification] RepairOrder {id} work started for customer {customerId}, vehicle {vin}` |
| Completed | `[Notification] RepairOrder {id} completed for customer {customerId}, total {amount}` |
| Cancelled | `[Notification] RepairOrder {id} cancelled for customer {customerId}. Reason: {reason}` |

### Confirming the notification pipeline end-to-end

1. Aspire Dashboard → Traces → find the trace for your RepairOrders operation
2. Expand to see Service Bus publish span (`servicebus.send`)
3. In a separate trace (Platform Host side), find the `ServiceBusProcessor.ProcessMessage` span
4. Expand to see the notification log span inside it
5. Correlate by `TraceId` — it is propagated through Service Bus message headers

### Notification configuration

| File | Purpose |
|---|---|
| `src/Modules/Notifications/VehicleLifecycle.Notifications.Application/INotificationService.cs` | The notification interface (contract) |
| `src/Modules/Notifications/VehicleLifecycle.Notifications.Infrastructure/LoggingNotificationService.cs` | Current stub implementation |
| `src/Modules/Notifications/VehicleLifecycle.Notifications.Infrastructure/NotificationsInfrastructureExtensions.cs` | DI registration; swap implementation here |
| `src/Platform/VehicleLifecycle.Platform.Host/RepairOrdersEventConsumer.cs` | Where notifications are dispatched from |

**To add a real email address for a customer:** the `customerId` field in the integration events
is currently treated as a display string. Add a customer lookup repository in the Notifications
infrastructure layer and resolve `customerId` to an email before calling the provider.

---

## Monitoring in Production (Azure)

When deployed to Azure Container Apps:

| Tool | What it shows |
|---|---|
| **Application Insights** | Traces, logs, metrics, failures, performance, live metrics stream |
| **Azure Monitor** | Alerts, dashboards, metric aggregations |
| **Container Apps → Log stream** | Real-time stdout/stderr for a specific replica |
| **Container Apps → Metrics** | CPU, memory, request count per replica |
| **Service Bus → Metrics** | Incoming/outgoing messages, DLQ depth, throttled requests |

### Useful Application Insights KQL queries

```kusto
-- Slow requests (p99)
requests
| where timestamp > ago(1h)
| summarize percentile(duration, 99) by name
| order by percentile_duration_99 desc

-- Notification log events
traces
| where message contains "[Notification]"
| order by timestamp desc
| take 50

-- Failed requests
requests
| where success == false
| order by timestamp desc
| take 50

-- Dead-letter events
traces
| where message contains "DeadLetterMonitor"
| order by timestamp desc
```

---

## Checklist for a Complete System Health Check

```
□ Aspire Dashboard → all resources green
□ GET /health/ready returns 200 for both services
□ Outbox: SELECT * FROM OutboxMessages WHERE ProcessedAt IS NULL → empty or near-empty
□ Dead-letter queue: empty (no stuck messages)
□ Redis: DBSIZE returns expected idempotency keys
□ Traces: no error spans in the last 5 minutes
□ Metrics: http.server.request.duration p99 < 500 ms
□ Logs: no ERROR or CRITICAL entries in the last 5 minutes
```

---

## Related Documents

- [20-application-guide.md](20-application-guide.md) — application overview and usage
- [21-debugging-guide.md](21-debugging-guide.md) — step-by-step breakpoint debugging
- [11-operability-and-observability.md](11-operability-and-observability.md) — OpenTelemetry standards and logging conventions
- [18-local-development.md](18-local-development.md) — local setup prerequisites
