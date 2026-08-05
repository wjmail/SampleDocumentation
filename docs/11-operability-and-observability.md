# 11 — Operability and Observability

## Implementation Status (Phase 8 — commit `3f8eccd`)

| Capability | Status | Location |
|---|---|---|
| OTel traces + metrics + logs | Implemented | `src/ServiceDefaults/Extensions.cs` |
| OTLP export (Aspire Dashboard) | Implemented | conditional on `OTEL_EXPORTER_OTLP_ENDPOINT` |
| Azure Monitor / App Insights export | Implemented | conditional on `APPLICATIONINSIGHTS_CONNECTION_STRING` |
| Correlation ID middleware (`X-Correlation-Id`) | Implemented | `src/BuildingBlocks/VehicleLifecycle.Observability/` |
| `GET /health` liveness | Implemented | mapped in all environments |
| `GET /health/ready` readiness | Implemented | per-module DB checks (`claims-db`, `repairorders-db`, `parts-db`) |
| `AspNetCore` instrumentation | Implemented | `AddAspNetCoreInstrumentation()` |
| `HttpClient` instrumentation | Implemented | `AddHttpClientInstrumentation()` |
| `SqlClient` instrumentation | Implemented | `AddSqlClientInstrumentation()` |
| MediatR pipeline tracing | Planned | not yet implemented |
| Custom business metrics | Planned | not yet implemented |
| Service Bus tracing | Planned | after Phase 11 extraction |
| Alerting rules in Bicep | Planned | after Phase 11 |

---

## Principles

- Every request produces a trace, structured log entries, and contributes to metrics.
- No `Console.WriteLine` in production code — use `ILogger<T>`.
- Correlation ID propagated from APIM → API → all downstream calls.
- Health endpoints must respond in < 1 second; no DB queries in liveness check.

## OpenTelemetry

All instrumentation uses the **OpenTelemetry SDK for .NET**. No vendor-specific calls in business code.

### Signal configuration

| Signal | Local export | Azure export |
|---|---|---|
| **Traces** | Aspire Dashboard (OTLP) | Application Insights |
| **Logs** | Aspire Dashboard (OTLP) | Application Insights |
| **Metrics** | Aspire Dashboard (OTLP) | Application Insights / Azure Monitor |

### Instrumentation sources

| Source | What it captures |
|---|---|
| `AspNetCore` | HTTP request traces, status codes, durations |
| `HttpClient` | Outbound HTTP call traces |
| `SqlClient` | Database query traces (no statement text in prod) |
| `MassTransit` / custom | Service Bus message traces |
| `MediatR` pipeline | Command/Query handler traces (custom activity source) |
| Custom: `VehicleLifecycle.*` | Business-level spans (claim approved, order created) |

### Activity source naming

```
VehicleLifecycle.Claims
VehicleLifecycle.RepairOrders
VehicleLifecycle.Parts
VehicleLifecycle.Notifications
VehicleLifecycle.Outbox
```

## Structured logging

Log format: JSON (in production), human-readable in development.

### Mandatory fields on every log entry

| Field | Source |
|---|---|
| `TraceId` | OTel trace context |
| `SpanId` | OTel span context |
| `CorrelationId` | HTTP header `X-Correlation-Id` or generated |
| `UserId` | JWT `oid` claim (where authenticated) |
| `Module` | Source module name |
| `Environment` | `ASPNETCORE_ENVIRONMENT` |

### Log level guidelines

| Level | When |
|---|---|
| `Trace` | Detailed internal flow — dev only |
| `Debug` | Diagnostic values — dev only |
| `Information` | Business events (claim approved, order created) |
| `Warning` | Recoverable issues (retry, fallback) |
| `Error` | Handled exceptions, failed operations |
| `Critical` | Unrecoverable errors requiring immediate attention |

### What NOT to log

- Passwords, tokens, secrets
- Full request/response bodies (log only correlation IDs and status)
- PII (customer names, emails) — use anonymised IDs in logs

## Health checks

Registered at `/health` (liveness) and `/health/ready` (readiness):

```
GET /health        → 200 OK always (process is alive)
GET /health/ready  → 200 OK if all checks pass, 503 if degraded
```

### Readiness checks

| Check | What it tests |
|---|---|
| SQL Server | Can open connection and execute `SELECT 1` |
| Service Bus | Namespace reachable (SDK connectivity check) |
| Key Vault | Can list secrets (proves Managed Identity works) |
| Outbox worker | Background service is running |

### Container Apps health probe configuration

```bicep
livenessProbe: {
  httpGet: { path: '/health', port: 8080 }
  initialDelaySeconds: 10
  periodSeconds: 30
}
readinessProbe: {
  httpGet: { path: '/health/ready', port: 8080 }
  initialDelaySeconds: 15
  periodSeconds: 10
}
```

## Metrics

### Key business metrics (custom)

| Metric name | Type | Description |
|---|---|---|
| `claims.submitted.total` | Counter | Total claims submitted |
| `claims.approved.total` | Counter | Total claims approved |
| `claims.rejected.total` | Counter | Total claims rejected |
| `repair_orders.created.total` | Counter | Total repair orders created |
| `repair_orders.completed.total` | Counter | Total repair orders completed |
| `outbox.messages.pending` | Gauge | Unpublished outbox messages |
| `outbox.messages.failed` | Counter | Failed outbox dispatches |
| `notifications.sent.total` | Counter | Notifications delivered |
| `notifications.failed.total` | Counter | Notification delivery failures |

### Infrastructure metrics (auto-collected)

- HTTP request duration (P50, P95, P99)
- HTTP error rate (4xx, 5xx)
- DB connection pool usage
- Memory and CPU (Container Apps platform)

## Alerting (Bicep-defined)

| Alert | Condition | Severity |
|---|---|---|
| High error rate | HTTP 5xx > 1% over 5 min | 2 (Error) |
| Outbox backlog | `outbox.messages.pending` > 100 for 10 min | 2 (Error) |
| Health check failing | `/health/ready` returns 503 for 3 min | 1 (Critical) |
| Notification failures | `notifications.failed.total` increases by 10 in 5 min | 3 (Warning) |

## Deployment observability

- Each deployment creates a new Container App revision.
- Revision label matches git commit SHA.
- Application Insights annotations created on each `azd deploy`.
- `azd` deployment logs streamed to GitHub Actions output.

## Local observability (Aspire Dashboard)

Available at `https://localhost:15888` when running via AppHost:

- Structured logs with filtering
- Distributed traces with waterfall view
- Metrics graphs
- Resource health status
- No setup required — free, local, zero cost
