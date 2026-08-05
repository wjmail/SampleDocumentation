# 06 — REST API Guidelines

## Principles

- Resource-oriented design (nouns, not verbs in paths)
- HTTP methods carry the semantics (GET, POST, PUT, PATCH, DELETE)
- Stateless — all state in resource representations or tokens
- Consistent error model across all modules
- API versioning from day one

## URL structure

```
/api/v{version}/{resource}
/api/v{version}/{resource}/{id}
/api/v{version}/{resource}/{id}/{sub-resource}
```

Examples:
```
GET    /api/v1/claims
POST   /api/v1/claims
GET    /api/v1/claims/{claimId}
PATCH  /api/v1/claims/{claimId}/decision
GET    /api/v1/claims/{claimId}/notes
POST   /api/v1/claims/{claimId}/notes

GET    /api/v1/repair-orders
POST   /api/v1/repair-orders
GET    /api/v1/repair-orders/{repairOrderId}
PUT    /api/v1/repair-orders/{repairOrderId}/status

GET    /api/v1/parts
GET    /api/v1/parts/{partId}
POST   /api/v1/parts/reservations

```

## HTTP method semantics

| Method | Use | Idempotent | Safe |
|---|---|---|---|
| GET | Read resource(s) | Yes | Yes |
| POST | Create resource or trigger action | No | No |
| PUT | Replace resource entirely | Yes | No |
| PATCH | Partial update | No | No |
| DELETE | Delete resource | Yes | No |

## Versioning

- Version in URL path: `/api/v1/...`
- Default to `v1`; increment only on breaking changes
- Maintain previous version for at least one release cycle
- Breaking change = removing a field, changing a field type, removing an endpoint

## Request / Response conventions

### Identifiers
- Use `GUID` for all resource identifiers
- Return as lowercase string: `"claimId": "3fa85f64-5717-4562-b3fc-2c963f66afa6"`

### Dates and times
- ISO 8601 UTC: `"openedAt": "2025-01-15T10:30:00Z"`

### Collections
```json
{
  "items": [...],
  "totalCount": 42,
  "pageSize": 20,
  "pageNumber": 1
}
```

Query parameters for pagination: `?page=1&pageSize=20`  
Query parameters for sorting: `?sortBy=openedAt&sortOrder=desc`  
Query parameters for filtering: `?status=approved&handlerId=...`

### Partial update (PATCH)
Use JSON Merge Patch (RFC 7396) — send only the fields to update:
```json
PATCH /api/v1/claims/{id}/decision
{
  "outcome": "approved",
  "reason": "All documents verified"
}
```

## Error model

All errors return `application/problem+json` (RFC 7807):

```json
{
  "type": "https://vehiclelifecycle.example.com/errors/claim-not-found",
  "title": "Claim not found",
  "status": 404,
  "detail": "No claim with id '3fa85f64-...' exists.",
  "instance": "/api/v1/claims/3fa85f64-...",
  "traceId": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
}
```

### HTTP status codes

| Code | When |
|---|---|
| 200 OK | Successful GET, PUT, PATCH |
| 201 Created | Successful POST creating a resource — include `Location` header |
| 204 No Content | Successful DELETE |
| 400 Bad Request | Validation failure — include `errors` array in problem body |
| 401 Unauthorized | Missing or invalid token |
| 403 Forbidden | Valid token, insufficient role/permission |
| 404 Not Found | Resource does not exist |
| 409 Conflict | State conflict (e.g., claim already approved) |
| 422 Unprocessable Entity | Business rule violation (distinct from validation) |
| 500 Internal Server Error | Unexpected error — never expose stack traces |

### Validation error body

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
	"vehicleId": ["Vehicle ID is required."],
	"description": ["Description must not exceed 2000 characters."]
  }
}
```

## Authentication

All endpoints require a valid JWT Bearer token issued by Entra ID, except:
- `GET /health` — public
- `GET /api/v1/...` with explicit `[AllowAnonymous]` attribute (none in v1)

Header: `Authorization: Bearer <token>`

## Authorization (RBAC)

| Role | Claims | RepairOrders | Parts |
|---|---|---|---|
| ClaimsHandler | Full CRUD | Read | Read |
| WorkshopCoordinator | Read | Full CRUD | Read |
| Technician | Read | Update status/tasks | Read, request |
| PartCoordinator | Read | Read | Full CRUD |
| Admin | All | All | All |

## OpenAPI / Swagger

Each module exposes an OpenAPI document at `/swagger/v1/swagger.json`.  
Scalar UI (or Swagger UI) available in Development environment at `/scalar`.  
All endpoints must have `[ProducesResponseType]` attributes for documented status codes.
