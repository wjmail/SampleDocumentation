# 02 — Domain Analysis

## Event Storming — Big Picture

The following timeline captures the core domain events across all business areas, ordered roughly by business flow.

```
[Customer files claim]
  → ClaimSubmitted
  → ClaimAssignedToHandler
  → ClaimInvestigationStarted
  → ClaimApproved / ClaimRejected / ClaimEscalated

[Repair order created from approved claim]
  → RepairOrderCreated
  → RepairOrderAssignedToWorkshop
  → RepairOrderAssignedToTechnician
  → RepairStarted
  → PartsRequested
	→ PartReserved / PartOutOfStock / PartOrdered
  → RepairCompleted
  → RepairOrderClosed

[Notifications triggered by state changes]
  → NotificationRequested
  → NotificationSent / NotificationFailed

[Identity events]
  → UserRegistered
  → RoleAssigned / RoleRevoked
  → TokenIssued
```

## Aggregates and Entities

### Claims context

| Aggregate / Entity | Key attributes |
|---|---|
| **Claim** (aggregate root) | ClaimId, VehicleId, CustomerId, Status, HandlerId, OpenedAt, ClosedAt |
| **ClaimNote** | NoteId, ClaimId, AuthorId, Content, CreatedAt |
| **ClaimDecision** | DecisionId, ClaimId, Outcome, Reason, DecidedAt |

### RepairOrders context

| Aggregate / Entity | Key attributes |
|---|---|
| **RepairOrder** (aggregate root) | RepairOrderId, ClaimId, VehicleId, WorkshopId, Status, CreatedAt |
| **RepairTask** | TaskId, RepairOrderId, Description, TechnicianId, Status |
| **PartsRequest** | RequestId, RepairOrderId, PartId, Quantity, Status |

### Parts context

| Aggregate / Entity | Key attributes |
|---|---|
| **Part** (aggregate root) | PartId, SKU, Name, Category, UnitPrice |
| **StockEntry** | StockEntryId, PartId, WarehouseId, QuantityOnHand |
| **ProcurementRequest** | ProcurementRequestId, PartId, RequestedQty, Status |

### Notifications context

| Aggregate / Entity | Key attributes |
|---|---|
| **Notification** (aggregate root) | NotificationId, RecipientId, Channel, TemplateId, Status, ScheduledAt, SentAt |
| **NotificationTemplate** | TemplateId, Name, Subject, BodyTemplate, Channel |

### Identity context

| Aggregate / Entity | Key attributes |
|---|---|
| **User** (aggregate root) | UserId, ExternalId (Entra), Email, Roles |
| **Role** | RoleId, Name, Permissions |

## Commands

| Command | Emitted by | Handled in |
|---|---|---|
| SubmitClaim | Customer API | Claims |
| ApproveClaim | Claims handler | Claims |
| RejectClaim | Claims handler | Claims |
| CreateRepairOrder | Claims (on approval) | RepairOrders |
| AssignRepairOrder | Workshop coordinator | RepairOrders |
| StartRepair | Technician | RepairOrders |
| RequestParts | Technician | Parts |
| ReservePart | Parts system | Parts |
| CompleteRepair | Technician | RepairOrders |
| SendNotification | Any module via event | Notifications |

## Domain Events (internal)

| Event | Source aggregate | Consumed by |
|---|---|---|
| ClaimSubmitted | Claim | — |
| ClaimApproved | Claim | RepairOrders, Notifications |
| ClaimRejected | Claim | Notifications |
| RepairOrderCreated | RepairOrder | Notifications |
| RepairOrderCompleted | RepairOrder | Claims, Notifications |
| PartsRequested | RepairOrder | Parts |
| PartReserved | StockEntry | RepairOrders |
| PartOutOfStock | StockEntry | RepairOrders, Notifications |
| NotificationRequested | any | Notifications |
| NotificationSent | Notification | — |

## Ubiquitous Language (per context)

### Claims
- **Claim** — a formal request to initiate a vehicle repair under warranty or insurance
- **Handler** — an employee responsible for investigating and deciding a claim
- **Triage** — initial assessment of claim completeness and priority
- **Decision** — the formal outcome (approved / rejected / escalated)

### RepairOrders
- **Repair Order** — a work instruction authorizing a workshop to perform repairs
- **Workshop** — a physical or virtual repair facility
- **Technician** — a qualified person executing repair tasks
- **Task** — a discrete unit of repair work

### Parts
- **Part** — a physical component identified by SKU
- **Stock** — available inventory at a warehouse
- **Procurement Request** — a request to source a part from a supplier

### Notifications
- **Notification** — an outbound message to a recipient via a specific channel
- **Channel** — delivery mechanism (email, SMS, push)
- **Template** — reusable message structure with variable slots

### Identity
- **User** — a platform actor with an Entra ID-backed identity
- **Role** — a named set of permissions (e.g., ClaimsHandler, Technician, PartCoordinator)
