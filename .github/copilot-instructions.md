# CLAUDE.md - Inventory Hold Microservice

## Project Overview
- **Service**: Inventory Hold Microservice. Places temporary holds on items during checkout with configurable expiration (default: 15 mins).
- **Tech Stack**: .NET 10 (C#), MongoDB, Redis, RabbitMQ, Docker & Docker Compose, React + TypeScript (Vite), xUnit/nUnit.
- **Architecture**: Domain-Driven Design (DDD) layered architecture.

---

## Solution Structure
.
├── docker/
│   ├── docker-compose.yml
│   ├── db.env
│   └── cache.env
├── src/
│   ├── InventoryHold.Contracts/
│   │   ├── DTOs/
│   │   ├── Enums/
│   │   └── Models/
│   ├── InventoryHold.Domain/
│   │   ├── Services/
│   │   └── Repositories/
│   ├── InventoryHold.Infrastructure/
│   │   ├── Mongo/
│   │   ├── Redis/
│   │   └── RabbitMQ/
│   ├── InventoryHold.WebApi/
│   │   ├── Controllers/
│   │   ├── Program.cs
│   │   └── Startup.cs
│   └── InventoryHold.UnitTests/
│       └── Services/
└── web/
    ├── public/
    │   └── vite.svg
    ├── src/
    │   ├── api/
    │   │   ├── apiClient.ts
    │   │   └── inventoryApi.ts
    │   ├── components/
    │   ├── hooks/
    │   ├── types/
    │   ├── App.tsx
    │   ├── main.tsx
    │   └── vite-env.d.ts
    ├── index.html
    ├── package.json
    ├── tsconfig.json
    └── vite.config.ts

---

## Docker & Containerization Rules

### 1. Docker Compose Services
- `backend`: Multi-stage .NET 10 Web API build (`src/InventoryHold.WebApi/Dockerfile`).
  - Depends on `mongodb`, `redis`, and `rabbitmq` healthchecks.
  - Receives database, cache, and broker connection strings via environment variables (no hardcoded credentials).
- `frontend`: Multi-stage React + Vite build served via Nginx or Node (`frontend/Dockerfile`).
  - Proxies `/api` requests to the `backend` service or connects via environment-configured API base URL.
  - Depends on `backend` service availability.
- `mongodb`: MongoDB container with persistent volume and startup healthcheck.
- `redis`: Redis cache container with explicit memory config and healthcheck.
- `rabbitmq`: RabbitMQ with management UI enabled (`rabbitmq:3-management`).

### 2. Networking & Volumes
- All 5 services must run on a shared user-defined bridge network (e.g., `inventory-network`).
- Persistent named volumes must be mounted for MongoDB data to prevent data loss on restarts.

---

## Core Technical Constraints & Invariants

### 1. MongoDB & Concurrency
- **Race Condition Prevention**: Stock deduction and hold placement MUST use atomic operations (`FindOneAndUpdateAsync` checking stock > 0).
- **Seeding**: Automatically seed at least 5 products with stock levels on startup.
- **Config**: Connection strings strictly loaded via environment variables / configuration providers.

### 2. API Endpoints
- `POST /api/holds`: Verifies stock, deducts atomically, creates hold (default: 15m expiry), returns details.
- `GET /api/holds/{holdId}`: Returns hold details; returns 404 if not found; handles expired state.
- `DELETE /api/holds/{holdId}`: Releases hold and restores inventory (handles non-existent, already-released, and expired holds).
- `GET /api/inventory`: Lists current inventory levels.
- **HTTP Status Codes**: Use semantic status codes (`200`, `201`, `400`, `404`, `409`, `422`) with structured error responses.

### 3. Messaging (RabbitMQ)
- Publish domain events on hold lifecycle transitions: `HoldCreated`, `HoldReleased`, `HoldExpired`.
- Ensure payloads contain complete entity context for downstream consumers.

### 4. Caching (Redis)
- Cache high-frequency read paths (`GET /api/inventory`, active holds).
- Set explicit TTLs and implement immediate cache eviction upon mutations (`POST`, `DELETE`).

### 5. Frontend (React + TypeScript)
- Components: Inventory Dashboard, Create Hold Form, Active Holds List (with countdown timer), Release Hold action.
- Instant UI state synchronization post-mutation without requiring full page refresh.
- Strict TypeScript typing matching `InventoryHold.Contracts`.

---

## Testing Guidelines
- **Framework**: xUnit or nUnit with Moq/NSubstitute and FluentAssertions.
- **Scope**: Minimum 5 functional unit tests covering validation, hold lifecycle, and race condition/edge cases.
- **Isolation**: Unit tests must mock all repository, Redis, and RabbitMQ dependencies (no live infrastructure during tests).

---

## Common Commands
- **Full Stack Start (Frontend + Backend + Infra)**: `docker-compose up --build`
- **Stop All Containers**: `docker-compose down -v`
- **Run Backend Tests**: `dotnet test src/InventoryHold.UnitTests/`
- **Frontend Dev (Local)**: `cd frontend && npm run dev`
