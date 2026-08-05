# VehicleLifecycle Platform

[![.NET](https://img.shields.io/badge/.NET-10-512BD4)](https://dotnet.microsoft.com/)
[![Azure](https://img.shields.io/badge/Azure-Container%20Apps-0089D6)](https://azure.microsoft.com/)

A learning and portfolio solution for a Senior .NET Engineer preparing for a **Senior Software Architect** role focused on Microsoft Azure, microservices, REST, DDD, CQRS, event-driven systems, security, and architecture artifacts.

## What this is

An end-to-end platform in the **vehicle lifecycle domain**, built as a modular monolith in .NET 10 / ASP.NET Core, evolving toward selected microservices after domain boundaries, contracts, data ownership, and event flows are clear.

## Delivery Status

All phases up to Phase 19 are complete.

| Phase | Description | Status |
|-------|-------------|--------|
| 0-1   | Docs, Aspire skeleton | ✅ Done |
| 2-3   | Claims DDD/CQRS + EF Core + Outbox | ✅ Done |
| 4-5   | Entra ID auth + Azure IaC + CI/CD | ✅ Done |
| 6-7   | RepairOrders and Parts modules | ✅ Done |
| 8     | Observability — OTel, Azure Monitor, health checks | ✅ Done |
| 9     | Unit tests — Claims domain + application | ✅ Done |
| 10    | Documentation update | ✅ Done |
| 11    | Extract RepairOrders to microservice via Service Bus | ✅ Done |
| 12    | AI Case Assistant (stub, feature-flagged) | ✅ Done |
| 13    | Redis hybrid cache (L1 + L2 idempotency) | ✅ Done |
| 14    | Validation, MediatR behaviors, rate limiting | ✅ Done |
| 15    | Service Bus resilience, DLQ handling, RepairOrder Saga | ✅ Done |
| 16    | Unit tests — RepairOrders domain + application | ✅ Done |
| 17    | Unit tests — Parts domain + application + Notifications | ✅ Done |
| 18    | AI Case Assistant — real provider (Ollama / Azure OpenAI) | ✅ Done |
| 19    | Documents module — MongoDB-backed invoices | ✅ Done |

## Business domains

| Module | Bounded Context |
|--------|-----------------|
| Claims | Claim intake, triage, lifecycle management |
| Repair Orders | Workshop assignment, repair tracking |
| Parts | Parts catalog, procurement integration |
| Documents | Invoice generation and lifecycle (MongoDB) |
| Notifications | Event-driven outbound communication |
| Identity | Authentication, authorization, RBAC |

## Technology stack

| Concern | Technology |
|---------|------------|
| Runtime | .NET 10 / ASP.NET Core |
| Local orchestration | .NET Aspire |
| Database | Azure SQL (production), SQL Server container (local) |
| Document store | MongoDB / Azure Cosmos DB for MongoDB (Documents module) |
| Messaging | Azure Service Bus (production), emulator (local) |
| Identity | Microsoft Entra ID, OAuth2 / OIDC |
| Secrets | Azure Key Vault, Managed Identity |
| Observability | OpenTelemetry, Azure Monitor, Application Insights |
| Infrastructure | Bicep (IaC) |
| CI/CD | GitHub Actions + Azure Developer CLI (azd) |
| Patterns | DDD, CQRS, Outbox, Saga, Modular Monolith, Circuit Breaker |
| Testing | xUnit, FluentAssertions, NSubstitute (200 tests) |

## Local development

Prerequisites: Docker Desktop, .NET 10 SDK, azd CLI.

```bash
dotnet run --project src/AppHost
```

Aspire Dashboard opens at https://localhost:15888.
API available with Scalar UI at https://localhost:PORT/scalar/v1.
Health: GET /health (liveness), GET /health/ready (readiness).

## Azure deploy / teardown

```bash
azd up      # provision + deploy (Container Apps, SQL, Service Bus, ACR, Key Vault)
azd down    # full teardown - zero cost when not running
```

Or use the scripts directly:

```powershell
./scripts/deploy.ps1
./scripts/teardown.ps1
```

GitHub Actions **ci.yml** runs build and tests on every push to main.
Azure deploy (**deploy.yml**) is manual-only - never triggers automatically.

## Running tests

```bash
dotnet test src/VehicleLifecycle.Platform.slnx
```

| Project | Tests |
|---------|-------|
| VehicleLifecycle.Claims.Domain.Tests | 32 |
| VehicleLifecycle.Claims.Application.Tests | 5 |
| VehicleLifecycle.RepairOrders.Domain.Tests | 26 |
| VehicleLifecycle.RepairOrders.Application.Tests | 31 |
| VehicleLifecycle.Parts.Domain.Tests | 37 |
| VehicleLifecycle.Parts.Application.Tests | 31 |
| VehicleLifecycle.Notifications.Tests | 11 |
| VehicleLifecycle.Observability.Tests | 5 |
| VehicleLifecycle.Documents.Domain.Tests | ~10 |
| VehicleLifecycle.Documents.Application.Tests | ~12 |
| **Total** | **~200 passing** |

## Repository layout

```
NewTechnologies/
  src/
    AppHost/                           # .NET Aspire orchestration
    ServiceDefaults/                   # Shared OTel, health, service discovery
    BuildingBlocks/
      VehicleLifecycle.SharedKernel/
      VehicleLifecycle.Contracts/
      VehicleLifecycle.Observability/  # Correlation ID middleware
    Platform/
      VehicleLifecycle.Platform.Host/  # Composition root
    Modules/
      Claims/          # Reference module (Domain/Application/Infrastructure/Api)
      RepairOrders/
      Parts/
      Notifications/
      Identity/
      Documents/       # MongoDB-backed invoice module
    VehicleLifecycle.Platform.slnx
  tests/
    VehicleLifecycle.Claims.Domain.Tests/
    VehicleLifecycle.Claims.Application.Tests/
    VehicleLifecycle.RepairOrders.Domain.Tests/
    VehicleLifecycle.RepairOrders.Application.Tests/
    VehicleLifecycle.Parts.Domain.Tests/
    VehicleLifecycle.Parts.Application.Tests/
    VehicleLifecycle.Notifications.Tests/
    VehicleLifecycle.Observability.Tests/
    VehicleLifecycle.Documents.Domain.Tests/
    VehicleLifecycle.Documents.Application.Tests/
  infra/              # Bicep IaC modules
  scripts/            # deploy.ps1, teardown.ps1
  .github/workflows/  # ci.yml, deploy.yml
  docs/               # Architecture docs 01-23
  azure.yaml          # azd manifest
  Dockerfile          # Multi-stage .NET 10
```

## Documentation

| File | Content |
|------|---------|
| [01 Context and Goals](docs/01-context-and-goals.md) | Vision, stakeholders, quality attributes |
| [02 Domain Analysis](docs/02-domain-analysis.md) | Event storming, domain events, aggregates |
| [03 Bounded Contexts](docs/03-bounded-contexts.md) | Context definitions, ownership, language |
| [04 Context Map](docs/04-context-map.md) | Inter-context relationships |
| [05 Architecture Decisions](docs/05-architecture-decisions.md) | ADR log (ADR-001 – ADR-015) |
| [06 REST API Guidelines](docs/06-rest-api-guidelines.md) | Resource design, versioning, error model |
| [07 Event Catalog](docs/07-event-catalog.md) | Domain events, integration events, schema |
| [08 Security Model](docs/08-security-model.md) | Identity, AuthN/AuthZ, secrets, threat model |
| [09 Azure Topology](docs/09-azure-topology.md) | Resource map, networking, deployment topology |
| [10 Delivery Roadmap](docs/10-delivery-roadmap.md) | Phased delivery plan with status |
| [11 Operability and Observability](docs/11-operability-and-observability.md) | Telemetry, health, alerting |
| [12 AI Option](docs/12-ai-option-later.md) | Optional AI Case Assistant |
| [13 System Architecture](docs/13-system-architecture.md) | Architecture diagrams |
| [16 Data Architecture](docs/16-data-architecture.md) | Database schemas, MongoDB model |
| [18 Local Development](docs/18-local-development.md) | Full setup guide, migrations, CLI |
| [20 Application Guide](docs/20-application-guide.md) | How to use every feature / API |
| [21 Debugging Guide](docs/21-debugging-guide.md) | Step-by-step debugging |
| [22 Monitoring Guide](docs/22-monitoring-guide.md) | Monitoring, queues, performance |
| [23 Documents Module](docs/23-documents-module.md) | MongoDB invoice module reference |
