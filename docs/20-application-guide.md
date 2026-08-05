# 20 — Application Guide

> This guide explains what the VehicleLifecycle platform does, how to run it, and how to use
> every feature it provides. No prior .NET or Azure knowledge is assumed.

---

## What Is This Application?

VehicleLifecycle is a backend platform for managing the **lifecycle of vehicles** in a workshop /
fleet context. It handles:

| Domain | What it manages |
|---|---|
| **Claims** | Insurance / warranty claims for vehicles |
| **Parts** | Catalog of spare parts with stock levels |
| **Repair Orders** | Workshop repair orders from opening through completion |
| **Documents** | Invoices generated from completed repair orders |
| **Notifications** | Automatic customer notifications on every status change |

There is no end-user UI — the platform exposes a **REST API** consumed by mobile apps,
web frontends, or third-party integrations.

---

## System Overview

```
┌──────────────────────────────────────────────────┐
│              Platform Host (port 5001)           │
│  ┌──────────┐  ┌────────┐  ┌──────────────────┐ │
│  │ Claims   │  │ Parts  │  │  Notifications   │ │
│  └──────────┘  └────────┘  └──────────────────┘ │
└──────────────────────────────────────────────────┘
		 ↑                         ↑
		 │   Azure Service Bus     │
		 │   topic: repair-orders  │
		 ↓                         │
┌─────────────────────────────┐    │
│  RepairOrders Service        │───┘
│  (port 5002)                │
└─────────────────────────────┘
		 ↕
   ┌─────────────────────┐
   │  SQL Server          │
   │  (3 databases)       │
   └─────────────────────┘
```

- **Platform Host** — modular monolith hosting Claims, Parts, Notifications, Identity, **Documents**.
- **RepairOrders Service** — standalone microservice, publishes events to Service Bus.
- **Service Bus** — the RepairOrders Service publishes events; Platform Host receives them
  and triggers notifications.
- **SQL Server** — three isolated databases: `ClaimsDb`, `PartsDb`, `RepairOrdersDb`.
- **Redis** — distributed idempotency cache (prevents duplicate notification delivery).

---

## Starting the Application

### Prerequisites

| Tool | Notes |
|---|---|
| .NET 10 SDK | `dotnet --version` must show `10.x` |
| Docker Desktop | Must be **running** before startup |
| Visual Studio 2026 or CLI | Either works |

### One-command start

```powershell
cd C:\Git\wjmail\NewTechnologies
dotnet run --project src/AppHost/VehicleLifecycle.AppHost.csproj
```

Aspire starts everything automatically:
- Pulls SQL Server and Service Bus Emulator Docker images (first run only, ~1.5 GB)
- Runs database migrations
- Starts both services

### Aspire Dashboard

After startup, the console prints a link like:

```
Login to the dashboard at: http://localhost:18888/login?t=<token>
```

Open it in a browser. This is your **central control panel** — all services, logs, traces,
and resource status in one place.

✅ Green = healthy. 🔴 Red = check the logs by clicking the resource name.

---

## All Available Endpoints

Authentication is **disabled in local development** — no token is required.
See [08-security-model.md](08-security-model.md) for production auth configuration.

### Interactive API Documentation (Scalar UI)

| Service | Scalar URL |
|---|---|
| Platform Host | http://localhost:5001/scalar/v1 |
| RepairOrders Service | http://localhost:5002/scalar/v1 |

Open either URL in a browser to explore and call all endpoints interactively.

---

### Repair Orders (`RepairOrders Service` — port 5002)

A repair order models the full lifecycle of a vehicle workshop job.

#### Create a repair order

```http
POST /api/v1/repair-orders
Content-Type: application/json

{
  "customerId": "C001",
  "vehicleVin": "WVWZZZ1JZXW000001",
  "serviceType": "Engine Repair"
}
```

Response: `201 Created` with `{ "id": "<guid>" }`

#### Start work

```http
PUT /api/v1/repair-orders/{id}/start
```

Transitions status from `Open` → `InProgress`. Triggers a **Work Started** notification.

#### Complete the repair order

```http
PUT /api/v1/repair-orders/{id}/complete
```

Transitions status from `InProgress` → `Completed`. Triggers a **Completed** notification.

#### Cancel

```http
PUT /api/v1/repair-orders/{id}/cancel
Content-Type: application/json

{ "reason": "Customer changed their mind" }
```

Transitions to `Cancelled`. Triggers a **Cancelled** notification.

#### Get a repair order

```http
GET /api/v1/repair-orders/{id}
```

Returns the full state including all line items and current status.

---

### Claims (`Platform Host` — port 5001)

A claim is a request for insurance or warranty coverage for a vehicle defect or damage.

#### Submit a claim

```http
POST /api/v1/claims
Content-Type: application/json

{
  "customerId": "C001",
  "vehicleId": "V001",
  "description": "Engine oil leak found after service",
  "damageType": "Mechanical"
}
```

Response: `201 Created` with `{ "id": "<guid>" }`

#### Get a claim

```http
GET /api/v1/claims/{id}
```

#### AI summary (requires `Features:AiAssistant:Enabled = true`)

```http
POST /api/v1/claims/{id}/summarize
```

Returns a natural-language summary of the claim generated by Ollama (local) or Azure OpenAI.

---

### Parts (`Platform Host` — port 5001)

Parts catalog with stock management.

#### Register a part

```http
POST /api/v1/parts
Content-Type: application/json

{
  "partNumber": "OIL-FILTER-001",
  "name": "Oil Filter",
  "category": "Engine",
  "unitPrice": 24.99,
  "initialStock": 50
}
```

#### Get a part

```http
GET /api/v1/parts/{id}
```

#### List by category

```http
GET /api/v1/parts/by-category/Engine
```

#### Adjust stock

```http
POST /api/v1/parts/{id}/stock
Content-Type: application/json

{ "delta": -5, "reason": "Used in repair order RO-001" }
```

Positive `delta` = stock in, negative = stock out. Business rule: stock cannot go below zero.

---

## Notification Flow

Notifications are triggered automatically by domain events — no manual action required.

| Event | When triggered | What is logged |
|---|---|---|
| Repair Order Created | `POST /repair-orders` | `RepairOrder {id} created for customer {customerId}` |
| Work Started | `PUT /repair-orders/{id}/start` | `RepairOrder {id} work started for customer {customerId}, vehicle {vin}` |
| Repair Order Completed | `PUT /repair-orders/{id}/complete` | `RepairOrder {id} completed for customer {customerId}, total {amount}` |
| Repair Order Cancelled | `PUT /repair-orders/{id}/cancel` | `RepairOrder {id} cancelled for customer {customerId}. Reason: {reason}` |

### Current implementation: logging stub

The current `LoggingNotificationService` **writes to the structured log** instead of sending
a real email or SMS. This is intentional — it is a ready-to-replace stub.

**To add real email delivery:**

1. Open `src/Modules/Notifications/VehicleLifecycle.Notifications.Infrastructure/LoggingNotificationService.cs`
2. Replace `logger.LogInformation(...)` calls with your chosen provider
   (e.g. Azure Communication Services Email, SendGrid)
3. Add the recipient email lookup (today `customerId` is a plain string; map it to a stored email address)

**Notification configuration file:**

```
src/Modules/Notifications/VehicleLifecycle.Notifications.Application/INotificationService.cs
src/Modules/Notifications/VehicleLifecycle.Notifications.Infrastructure/LoggingNotificationService.cs
src/Modules/Notifications/VehicleLifecycle.Notifications.Infrastructure/NotificationsInfrastructureExtensions.cs
```

---

## Typical Walkthrough — 5 Minutes End-to-End

```
1. Start the app (see "Starting the Application" above)
2. Open Aspire Dashboard — wait for all resources to turn green
3. Open http://localhost:5002/scalar/v1

4. POST /api/v1/repair-orders
   Body: { "customerId": "C001", "vehicleVin": "ABC123", "serviceType": "Oil Change" }
   → Copy the returned {id}

5. PUT /api/v1/repair-orders/{id}/start
   → In Aspire Dashboard > Logs, search for "Notification"
   → You see: "[Notification] RepairOrder ... work started for customer C001, vehicle ABC123."

6. POST /api/v1/parts
   Body: { "partNumber": "OIL-1", "name": "Engine Oil 5W40", "category": "Engine",
		   "unitPrice": 15.50, "initialStock": 100 }

7. PUT /api/v1/repair-orders/{id}/complete
   → Notification: "RepairOrder ... completed for customer C001, total 0.00."

8. GET /api/v1/repair-orders/{id}
   → Status should now be "Completed"
```

---

## Feature Flags

| Flag | Development | Production | Effect |
|---|---|---|---|
| `Features:DisableAuthentication` | `true` | `false` | `true` → no token needed; all requests auto-authenticated |
| `Features:AiAssistant:Enabled` | `true` | `false` | `true` → Claims `/summarize` calls AI backend |

Configuration file: `src/Platform/VehicleLifecycle.Platform.Host/appsettings.Development.json`

---

## Rate Limiting

All endpoints enforce a **fixed window** rate limit:

- 100 requests per minute per IP address
- Queue of up to 10 additional requests
- Exceeding the limit returns `429 Too Many Requests`

---


---

## Documents API

The Documents module stores **invoices** linked to repair orders. Invoices are MongoDB documents
with embedded line items and a lifecycle: `Draft → Issued → Paid / Cancelled`.

### Authorization

| Operation | Required scope |
|---|---|
| Create invoice | `documents.write` |
| Read invoice | `documents.read` |

In local development (bypass mode) all requests pass automatically.

### Create Invoice

```http
POST /api/v1/invoices
Content-Type: application/json
Authorization: Bearer <token>
```

```json
{
  "repairOrderId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "currency": "PLN",
  "lineItems": [
	{
	  "description": "Oil change",
	  "quantity": 1,
	  "unitPrice": 150.00,
	  "taxRate": 0.23,
	  "discountRate": 0.00
	},
	{
	  "description": "Brake pads (front)",
	  "quantity": 2,
	  "unitPrice": 250.00,
	  "taxRate": 0.23,
	  "discountRate": 0.05
	}
  ]
}
```

**Response `201 Created`:**

```json
{
  "invoiceId": "b3d9e4b1-1234-4c89-a012-000000000001"
}
```

The invoice is created **and immediately issued** in one operation.

---

### Get Invoice by ID

```http
GET /api/v1/invoices/{invoiceId}
Authorization: Bearer <token>
```

**Response `200 OK`:**

```json
{
  "invoiceId": "b3d9e4b1-1234-4c89-a012-000000000001",
  "repairOrderId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "status": "Issued",
  "currency": "PLN",
  "totalNet": 662.50,
  "totalGross": 814.88,
  "lineItems": [
	{
	  "description": "Oil change",
	  "quantity": 1,
	  "unitPrice": 150.00,
	  "taxRate": 0.23,
	  "discountRate": 0.00,
	  "lineTotal": 184.50
	},
	{
	  "description": "Brake pads (front)",
	  "quantity": 2,
	  "unitPrice": 250.00,
	  "taxRate": 0.23,
	  "discountRate": 0.05,
	  "lineTotal": 630.38
	}
  ]
}
```

Returns `404 Not Found` if the invoice does not exist.

---

### Get Invoices by Repair Order

```http
GET /api/v1/invoices/by-repair-order/{repairOrderId}
Authorization: Bearer <token>
```

**Response `200 OK`:** array of invoice DTOs (same shape as single-invoice response).

---

### Invoice Lifecycle

```
[Draft] ---(issued on creation)---> [Issued]
[Issued] ---> [Paid]
[Issued] ---> [Cancelled]
[Paid]    (terminal)
[Cancelled] (terminal)
```

Domain events raised:

| Transition | Event |
|---|---|
| Created | `InvoiceCreated` |
| Issued | `InvoiceIssued` |
| Paid | `InvoicePaid` |
| Cancelled | `InvoiceCancelled` |


## Related Documents

- [18-local-development.md](18-local-development.md) — full setup guide, migrations, CLI reference
- [06-rest-api-guidelines.md](06-rest-api-guidelines.md) — API design conventions
- [08-security-model.md](08-security-model.md) — authentication, scopes, Entra ID
- [21-debugging-guide.md](21-debugging-guide.md) — step-by-step debugging
- [22-monitoring-guide.md](22-monitoring-guide.md) — monitoring, queues, performance
