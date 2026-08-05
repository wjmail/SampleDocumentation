# 07 — Event Catalog

## Conventions

- **Domain event** — internal to a bounded context; raised by an aggregate after a state change.
- **Integration event** — published to the Outbox and dispatched to other modules (in-process) or Service Bus (cross-service). Named in past tense.
- All events carry: `eventId` (GUID), `occurredAt` (ISO 8601 UTC), `version` (int).
- Schema changes increment `version`. Consumers must handle unknown fields gracefully (forward compatibility).

---

## Claims context

### ClaimSubmitted `v1`

Raised when a new claim is accepted.

```json
{
  "eventId": "uuid",
  "eventType": "Claims.ClaimSubmitted",
  "version": 1,
  "occurredAt": "2025-01-15T10:30:00Z",
  "claimId": "uuid",
  "vehicleId": "uuid",
  "customerId": "uuid",
  "description": "string",
  "submittedAt": "2025-01-15T10:30:00Z"
}
```

Consumers: — (informational; no downstream reaction in v1)

---

### ClaimApproved `v1`

Raised when a handler approves a claim.

```json
{
  "eventId": "uuid",
  "eventType": "Claims.ClaimApproved",
  "version": 1,
  "occurredAt": "2025-01-15T14:00:00Z",
  "claimId": "uuid",
  "vehicleId": "uuid",
  "customerId": "uuid",
  "handlerId": "uuid",
  "approvedAt": "2025-01-15T14:00:00Z",
  "estimatedRepairCost": 1250.00
}
```

Consumers: **RepairOrders** (creates repair order), **Notifications** (notifies customer)

---

### ClaimRejected `v1`

```json
{
  "eventId": "uuid",
  "eventType": "Claims.ClaimRejected",
  "version": 1,
  "occurredAt": "2025-01-15T14:05:00Z",
  "claimId": "uuid",
  "customerId": "uuid",
  "handlerId": "uuid",
  "reason": "string",
  "rejectedAt": "2025-01-15T14:05:00Z"
}
```

Consumers: **Notifications**

---

### ClaimClosed `v1`

```json
{
  "eventId": "uuid",
  "eventType": "Claims.ClaimClosed",
  "version": 1,
  "occurredAt": "2025-01-20T09:00:00Z",
  "claimId": "uuid",
  "closedAt": "2025-01-20T09:00:00Z"
}
```

Consumers: — (informational)

---

## RepairOrders context

### RepairOrderCreated `v1`

```json
{
  "eventId": "uuid",
  "eventType": "RepairOrders.RepairOrderCreated",
  "version": 1,
  "occurredAt": "2025-01-15T14:02:00Z",
  "repairOrderId": "uuid",
  "claimId": "uuid",
  "vehicleId": "uuid",
  "workshopId": "uuid",
  "createdAt": "2025-01-15T14:02:00Z"
}
```

Consumers: **Notifications**

---

### RepairOrderCompleted `v1`

```json
{
  "eventId": "uuid",
  "eventType": "RepairOrders.RepairOrderCompleted",
  "version": 1,
  "occurredAt": "2025-01-20T08:55:00Z",
  "repairOrderId": "uuid",
  "claimId": "uuid",
  "vehicleId": "uuid",
  "completedAt": "2025-01-20T08:55:00Z",
  "finalCost": 1175.50
}
```

Consumers: **Claims** (triggers claim closure), **Notifications**

---

### PartsRequested `v1`

```json
{
  "eventId": "uuid",
  "eventType": "RepairOrders.PartsRequested",
  "version": 1,
  "occurredAt": "2025-01-16T10:00:00Z",
  "repairOrderId": "uuid",
  "requestId": "uuid",
  "parts": [
	{ "partId": "uuid", "sku": "string", "quantity": 2 }
  ]
}
```

Consumers: **Parts**

---

## Parts context

### PartReserved `v1`

```json
{
  "eventId": "uuid",
  "eventType": "Parts.PartReserved",
  "version": 1,
  "occurredAt": "2025-01-16T10:05:00Z",
  "requestId": "uuid",
  "repairOrderId": "uuid",
  "partId": "uuid",
  "quantity": 2
}
```

Consumers: **RepairOrders**

---

### PartOutOfStock `v1`

```json
{
  "eventId": "uuid",
  "eventType": "Parts.PartOutOfStock",
  "version": 1,
  "occurredAt": "2025-01-16T10:05:00Z",
  "requestId": "uuid",
  "repairOrderId": "uuid",
  "partId": "uuid",
  "sku": "string",
  "requestedQuantity": 2,
  "availableQuantity": 0
}
```

Consumers: **RepairOrders** (blocks task), **Notifications** (alerts coordinator)

---

## Notifications context

### NotificationSent `v1`

```json
{
  "eventId": "uuid",
  "eventType": "Notifications.NotificationSent",
  "version": 1,
  "occurredAt": "2025-01-15T14:03:00Z",
  "notificationId": "uuid",
  "recipientId": "uuid",
  "channel": "email",
  "sentAt": "2025-01-15T14:03:00Z"
}
```

Consumers: — (audit/observability only)

---

## Versioning strategy

| Scenario | Action |
|---|---|
| Add optional field | Increment minor only; `version` stays the same |
| Add required field | New `version` value; old version kept for one release |
| Remove field | Breaking — new `version`; old consumers must be updated before old version retired |
| Rename field | Treated as remove + add — breaking |

Consumers must ignore unknown fields (JSON deserialization with `JsonIgnoreCondition.WhenWritingNull` or equivalent).

## Idempotency

Every consumer stores `ProcessedEventId` on successful processing. If the same `eventId` arrives twice, the consumer acknowledges without reprocessing.

---

## Documents context

All invoice events are **domain events** raised within the modular monolith.
They are not published to Service Bus in v1 (no cross-service subscribers yet).

### InvoiceCreated `v1`

```json
{
  "eventId": "uuid",
  "eventType": "Documents.InvoiceCreated",
  "version": 1,
  "occurredAt": "2025-07-01T09:00:00Z",
  "invoiceId": "uuid",
  "repairOrderId": "uuid",
  "currency": "PLN",
  "lineItemCount": 3
}
```

Consumers: internal (audit / future Notifications)

---

### InvoiceIssued `v1`

```json
{
  "eventId": "uuid",
  "eventType": "Documents.InvoiceIssued",
  "version": 1,
  "occurredAt": "2025-07-01T09:00:05Z",
  "invoiceId": "uuid",
  "totalNet": 1200.00,
  "totalGross": 1476.00,
  "currency": "PLN"
}
```

Consumers: internal

---

### InvoicePaid `v1`

```json
{
  "eventId": "uuid",
  "eventType": "Documents.InvoicePaid",
  "version": 1,
  "occurredAt": "2025-07-01T10:30:00Z",
  "invoiceId": "uuid",
  "paidAt": "2025-07-01T10:30:00Z"
}
```

Consumers: internal

---

### InvoiceCancelled `v1`

```json
{
  "eventId": "uuid",
  "eventType": "Documents.InvoiceCancelled",
  "version": 1,
  "occurredAt": "2025-07-01T11:00:00Z",
  "invoiceId": "uuid",
  "reason": "string"
}
```

Consumers: internal

