# 23 — Documents Module

> Reference guide for the **Documents** bounded context — invoices stored as MongoDB documents.

---

## Overview

The Documents module manages **invoices** generated from completed repair orders. It is a
self-contained bounded context hosted inside the Platform Host (modular monolith) that uses
**MongoDB** as its persistence store.

| Attribute | Value |
|---|---|
| **Type** | Supporting Domain |
| **Host** | `VehicleLifecycle.Platform.Host` |
| **Data store** | MongoDB — database `documents`, collection `invoices` |
| **Technology choice** | ADR-015: document-native fit for variable line items |
| **Module path** | `src/Modules/Documents/` |

---

## Architecture

```
src/Modules/Documents/
├── VehicleLifecycle.Documents.Domain/
│   └── Invoices/
│       ├── Invoice.cs               ← Aggregate root
│       ├── InvoiceId.cs             ← Strongly-typed ID
│       ├── InvoiceLineItem.cs       ← Value object (embedded)
│       ├── InvoiceStatus.cs         ← Enum
│       └── InvoiceEvents.cs         ← Domain events
├── VehicleLifecycle.Documents.Application/
│   ├── Commands/
│   │   └── CreateInvoice/
│   │       ├── CreateInvoiceCommand.cs
│   │       ├── CreateInvoiceCommandHandler.cs
│   │       └── CreateInvoiceCommandValidator.cs
│   ├── Queries/
│   │   └── InvoiceQueries.cs        ← Get by ID + by repair order
│   └── Contracts/
│       └── IInvoiceRepository.cs
├── VehicleLifecycle.Documents.Infrastructure/
│   ├── Persistence/
│   │   ├── MongoInvoiceRepository.cs
│   │   ├── DocumentsBsonConfiguration.cs
│   │   └── InvoiceIdSerializer.cs
│   └── DocumentsInfrastructureExtensions.cs
└── VehicleLifecycle.Documents.Api/
	└── DocumentsModuleExtensions.cs  ← DI + endpoint registration
```

---

## Domain Model

### Invoice (Aggregate Root)

```
Invoice
├── InvoiceId          (UUID, strongly typed)
├── RepairOrderId      (UUID reference — no FK)
├── Status             Draft | Issued | Paid | Cancelled
├── Currency           ISO 4217 string (e.g. "PLN")
├── TotalNet           computed from line items
├── TotalGross         computed from line items
├── CreatedAt
├── IssuedAt?
├── PaidAt?
├── CancelledAt?
└── LineItems[]
	├── Description
	├── Quantity
	├── UnitPrice
	├── TaxRate        (0.0 – 1.0)
	└── DiscountRate   (0.0 – 1.0)
```

### Lifecycle

```
[Draft] ──(Issue)──▶ [Issued] ──(MarkAsPaid)──▶ [Paid]       ← terminal
						   └──(Cancel)──▶ [Cancelled]         ← terminal
```

`CreateInvoiceCommand` creates **and immediately issues** the invoice in one operation.

### Domain Events

| Event | Raised when |
|---|---|
| `InvoiceCreated` | Invoice aggregate constructed |
| `InvoiceIssued` | `Invoice.Issue()` called |
| `InvoicePaid` | `Invoice.MarkAsPaid()` called |
| `InvoiceCancelled` | `Invoice.Cancel()` called |

---

## API Endpoints

All endpoints are registered in `DocumentsModuleExtensions.MapDocumentsEndpoints`.

| Method | Path | Scope | Description |
|---|---|---|---|
| `POST` | `/api/v1/invoices` | `documents.write` | Create + issue a new invoice |
| `GET` | `/api/v1/invoices/{id}` | `documents.read` | Get invoice by ID |
| `GET` | `/api/v1/invoices/by-repair-order/{repairOrderId}` | `documents.read` | List invoices for a repair order |

For full request/response examples see [20-application-guide.md](20-application-guide.md#documents-api).

---

## Infrastructure

### MongoDB Connection

In `DocumentsInfrastructureExtensions.AddDocumentsInfrastructure`:

```csharp
var connectionString = configuration.GetConnectionString("mongodb");
services.AddSingleton<IMongoClient>(_ => new MongoClient(connectionString));
services.AddSingleton(sp =>
	sp.GetRequiredService<IMongoClient>().GetDatabase("documents"));
services.AddScoped<IInvoiceRepository, MongoInvoiceRepository>();
```

The connection string name `mongodb` is the same key used by the Aspire `AddMongoDB("mongodb")`
call in `AppHost.cs`.

### BSON Serialization

`DocumentsBsonConfiguration.Register()` (called at startup) registers:

- `InvoiceIdSerializer` — serializes `InvoiceId` as UUID BSON subtype 4

### Local Development

```csharp
// AppHost.cs
var mongo = builder.AddMongoDB("mongodb").WithDataVolume();
platform.WithReference(mongo).WaitFor(mongo);
```

No `appsettings.json` changes needed — Aspire injects `ConnectionStrings:mongodb` automatically.

---

## Tests

| Project | Tests | What is covered |
|---|---|---|
| `VehicleLifecycle.Documents.Domain.Tests` | ~10 | Invoice lifecycle, totals, event emission, cancellation guard |
| `VehicleLifecycle.Documents.Application.Tests` | ~12 | `CreateInvoiceCommandHandler`, `CreateInvoiceCommandValidator` |

Run all Documents tests:

```bash
dotnet test --filter "FullyQualifiedName~Documents"
```

---

## Configuration

No module-specific configuration keys. The only external dependency is the MongoDB connection
string supplied by Aspire (local) or Azure Key Vault / App Settings (production).

| Key | Source | Example value |
|---|---|---|
| `ConnectionStrings:mongodb` | Aspire / Key Vault | `mongodb://localhost:27017` |

---

## Adding a New Invoice Field

1. Add the property to `Invoice` or `InvoiceLineItem` in the Domain project.
2. Update `CreateInvoiceCommand` and its validator in the Application project.
3. Update `InvoiceDto` in `InvoiceQueries.cs` and `InvoiceMapper.MapToDto`.
4. No migration needed — MongoDB ignores missing/extra BSON fields by default.

---

## Related Documents

- [03-bounded-contexts.md](03-bounded-contexts.md) — context map
- [05-architecture-decisions.md](05-architecture-decisions.md) — ADR-015 MongoDB decision
- [07-event-catalog.md](07-event-catalog.md) — Invoice domain events
- [08-security-model.md](08-security-model.md) — `documents.read` / `documents.write` scopes
- [16-data-architecture.md](16-data-architecture.md) — MongoDB schema reference
- [20-application-guide.md](20-application-guide.md) — Documents API usage examples
