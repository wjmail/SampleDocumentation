# 13 — System Architecture

## Overview

VehicleLifecycle is a **.NET 10 cloud-native application** built as a **modular monolith with an extracted microservice**, running on **Azure Container Apps** and orchestrated locally with **.NET Aspire**.

The system manages the full lifecycle of vehicles in a service/repair shop context: insurance claims, repair orders, parts catalogue, invoice documents, notifications, and identity.

---

## High-Level Architecture

```mermaid
graph TB
	subgraph Clients
		UI[Web / Mobile Client]
		APICLIENT[API Client / Scalar UI]
	end

	subgraph Azure["Azure (Production)"]
		subgraph ACA["Azure Container Apps"]
			PH["Platform Host\n(Modular Monolith)\nport 8080"]
			ROS["RepairOrders Service\n(Standalone Microservice)\nport 8081"]
		end

		subgraph Databases["Azure SQL (per module)"]
			DB_CLAIMS[(claims-db)]
			DB_PARTS[(parts-db)]
			DB_REPAIRORDERS[(repairorders-db)]
		end

		DB_DOCS[("Cosmos DB for MongoDB\ndocuments-db")]

		subgraph Messaging["Azure Service Bus (Standard)"]
			T_RO[Topic: repair-orders]
			T_PARTS[Topic: parts]
			SUB_PH[Subscription: platform-host]
		end

		ACR[Azure Container Registry]
		KV[Azure Key Vault]
		AI[Azure Application Insights]
		ENTRA[Microsoft Entra ID]
	end

	UI -->|HTTPS + JWT| PH
	APICLIENT -->|HTTPS + JWT| ROS

	PH --> DB_CLAIMS
	PH --> DB_PARTS
	PH --> DB_DOCS
	PH --> T_PARTS

	ROS --> DB_REPAIRORDERS
	ROS -->|publish| T_RO

	T_RO --> SUB_PH
	SUB_PH -->|consume| PH

	PH -.->|secrets| KV
	ROS -.->|secrets| KV
	PH -.->|telemetry| AI
	ROS -.->|telemetry| AI
	ENTRA -.->|JWT validation| PH
	ENTRA -.->|JWT validation| ROS
```

---

## Local Development Architecture (.NET Aspire)

```mermaid
graph TB
	subgraph Aspire["Aspire AppHost (orchestrator)"]
		AH[AppHost.cs]
	end

	subgraph Services["Hosted Services"]
		PH["platform-host\n:5001"]
		ROS["repairorders-service\n:5002"]
	end

	subgraph LocalInfra["Local Infrastructure (emulators / containers)"]
		SQL1[(SQL Server\nclaims-db)]
		SQL2[(SQL Server\nparts-db)]
		SQL3[(SQL Server\nrepairorders-db)]
		SBE[Service Bus Emulator\n:5672 AMQP]
		MONGO[(MongoDB\ndocuments-db)]
	end

	DASH[Aspire Dashboard\n:18888]

	AH --> PH
	AH --> ROS
	AH --> SQL1
	AH --> SQL2
	AH --> SQL3
	AH --> SBE
	AH --> MONGO

	PH --> SQL1
	PH --> SQL2
	PH --> MONGO
	PH --> SBE
	ROS --> SQL3
	ROS --> SBE

	PH -.->|metrics/traces| DASH
	ROS -.->|metrics/traces| DASH
```

---

## Component Decomposition

### Platform Host — Modular Monolith

The Platform Host is a single ASP.NET Core process that hosts **five bounded context modules** in one deployment unit. Modules are isolated by project boundaries and communicate only via in-process MediatR or Service Bus integration events.

```mermaid
graph LR
	subgraph PlatformHost["VehicleLifecycle.Platform.Host"]
		API[ASP.NET Core\nMinimal API]
		MW[Middleware\nCorrelation ID\nAuth/Authz]

		subgraph Modules
			CLAIMS[Claims Module]
			PARTS[Parts Module]
			NOTIFY[Notifications Module]
			IDENTITY[Identity Module]
			DOCS[Documents Module]
		end

		CONSUMER[RepairOrdersEventConsumer\nIHostedService]
	end

	API --> MW
	MW --> CLAIMS
	MW --> PARTS
	MW --> NOTIFY
	MW --> IDENTITY
	MW --> DOCS
	CONSUMER --> NOTIFY
```

### RepairOrders Service — Standalone Microservice

Extracted from the monolith in Phase 11. Owns its own database, publishes integration events to Service Bus, and exposes its own API surface.

```mermaid
graph LR
	subgraph RepairOrdersService["VehicleLifecycle.RepairOrders.Service"]
		API2[ASP.NET Core\nMinimal API]
		MODULE[RepairOrders Module\nDomain + Application\n+ Infrastructure]
		OUTBOX[Outbox Dispatcher\nIHostedService]
		DB[(repairorders-db)]
		SB_OUT[Service Bus\nrepair-orders topic]
	end

	API2 --> MODULE
	MODULE --> DB
	MODULE --> OUTBOX
	OUTBOX --> SB_OUT
```

---

## Module Structure (per bounded context)

Each module follows a **four-layer vertical slice**:

```
VehicleLifecycle.<Module>.Domain          — Entities, Value Objects, Domain Events
VehicleLifecycle.<Module>.Application     — Commands, Queries, Handlers (CQRS), Contracts
VehicleLifecycle.<Module>.Infrastructure  — EF Core DbContext, Repository, Outbox
VehicleLifecycle.<Module>.Api             — Minimal API endpoint registration, DTOs
```

---

## Deployment Pipeline

```mermaid
sequenceDiagram
	participant Dev as Developer
	participant GH as GitHub Actions
	participant ACR as Azure Container Registry
	participant ACA as Azure Container Apps

	Dev->>GH: git push main
	GH->>GH: dotnet build + test (ci.yml)
	GH->>ACR: docker build & push (platform-host)
	GH->>ACR: docker build & push (repairorders-service)
	GH->>ACA: azd deploy (deploy.yml)
	ACA-->>GH: print service URLs
```

---

## Security Boundary

```mermaid
graph LR
	ENTRA[Microsoft Entra ID\nExternal IdP]
	CLIENT[Client]
	GW[HTTPS Endpoint\nAzure Container Apps Ingress]
	PH[Platform Host]
	ROS[RepairOrders Service]

	CLIENT -->|1. Obtain JWT| ENTRA
	CLIENT -->|2. Bearer token| GW
	GW -->|3. Forward| PH
	GW -->|3. Forward| ROS
	PH -->|4. Validate JWT\ncheck roles + scopes| ENTRA
	ROS -->|4. Validate JWT\ncheck roles + scopes| ENTRA
```

All inter-service communication (Service Bus) uses **Managed Identity** — no passwords or connection strings stored in application code.
