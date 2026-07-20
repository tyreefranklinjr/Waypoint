# Waypoint

**An event-driven order fulfillment platform demonstrating service decomposition, asynchronous messaging, and resilience patterns at small scale.**

Waypoint models the lifecycle of an order — from authentication, through placement, inventory reservation, and customer notification — as four independently deployable services communicating over a shared event backbone. It's a hands-on implementation of patterns commonly found in distributed, event-driven backend systems: database-per-service isolation, idempotent event consumption, circuit breakers, distributed locking, and consumer-driven contract testing.

> **Status:** Actively under construction. This README documents the target architecture and tracks progress against it layer by layer — see [Build status](#build-status) below for what's implemented today versus planned.

---

## Table of contents

- [Architecture](#architecture)
- [Service catalog](#service-catalog)
- [Tech stack](#tech-stack)
- [Repository structure](#repository-structure)
- [Quick start](#quick-start)
- [Design decisions](#design-decisions)
- [Build status](#build-status)
- [Roadmap](#roadmap)

---

## Architecture

Waypoint is decomposed into four services, each owning its own PostgreSQL database. Services communicate synchronously (REST) only where a request needs an immediate answer — e.g. token validation — and asynchronously (event bus) for everything that represents "something happened," so that no service's availability depends on another service being up at the moment of the call.

```
                 ┌───────────────┐
   sync (JWT)    │  Auth service  │
   ┌────────────▶│                │
   │              └───────────────┘
┌──────────────┐
│ Order service │
└──────┬────────┘
       │ publishes: order.placed
       ▼
┌──────────────────┐
│   Event bus       │
└──────┬────────────┘
       │ consumed by
       ▼
┌────────────────────┐
│ Inventory service   │
└──────┬───────────────┘
       │ publishes: inventory.reserved / inventory.failed
       ▼
┌──────────────────┐
│   Event bus        │
└──────┬────────────┘
       │ consumed by
       ▼
┌──────────────────────┐
│ Notification service  │
└───────────────────────┘
```

A rendered architecture diagram with event-flow annotations lives at `docs/architecture/event-flow.png` (added once Layer 2 is implemented).

## Service catalog

| Service | Responsibility | Owns | Exposes |
|---|---|---|---|
| **auth-service** | Issues and validates JWTs; manages user credentials | User accounts, hashed credentials | `POST /login`, `POST /token/refresh`, JWKS endpoint for local token validation |
| **order-service** | Accepts and records customer orders | Orders, order line items | `POST /orders`, `GET /orders/{id}` &nbsp;·&nbsp; publishes `order.placed` |
| **inventory-service** | Tracks and reserves stock | Product stock levels | `GET /products/{id}` &nbsp;·&nbsp; consumes `order.placed` → publishes `inventory.reserved` / `inventory.failed` |
| **notification-service** | Notifies customers of order status | Notification log | consumes `inventory.reserved` / `inventory.failed` |

Each service is a self-contained FastAPI application with its own dependencies, its own database, and its own Dockerfile. No service imports code from another, and no service queries another's database directly — all cross-service communication happens over documented APIs or events.

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| API framework | FastAPI + Uvicorn | Async-native, automatic OpenAPI docs, strong typing via Pydantic |
| Database | PostgreSQL (one instance per service) | Enforces the database-per-service boundary at the infrastructure level, not just convention |
| ORM / migrations | SQLAlchemy 2.x + Alembic | Version-controlled schema evolution from day one |
| Event backbone | Kafka | Chosen over RabbitMQ for consumer offset/replay semantics — see [ADR-0002](docs/adr/0002-kafka-over-rabbitmq.md) |
| Caching / locking | Redis | Distributed locks for race-condition-prone operations (e.g. stock decrement) |
| Resilience | tenacity | Retry with backoff on synchronous inter-service calls, paired with a circuit breaker |
| Containerization | Docker + Docker Compose | Independent build and deploy per service; one-command local environment |
| Testing | pytest, Pact | Unit/integration tests per service; consumer-driven contract tests across service boundaries |

## Repository structure

```
waypoint/
├── auth-service/
│   ├── main.py
│   ├── requirements.txt
│   ├── Dockerfile
│   ├── .env.example
│   └── alembic/
├── order-service/          (same layout)
├── inventory-service/      (same layout)
├── notification-service/   (same layout)
├── docs/
│   ├── architecture/
│   ├── adr/
│   └── failure-scenarios/
├── docker-compose.yml
├── docker-compose.test.yml
└── README.md
```

Each `*-service/` directory is a fully independent Python project — its own virtual environment, its own dependency list, its own container image. Nothing is shared between them by design; that independence is what makes "database-per-service" and "independently deployable" true statements about this codebase rather than aspirational labels.

## Quick start

```bash
git clone https://github.com/tyreefranklinjr/Waypoint.git
cd Waypoint
docker compose up --build
```

Once running, each service exposes a liveness check and interactive API docs:

| Service | Health check | API docs |
|---|---|---|
| auth-service | `localhost:8000/health` | `localhost:8000/docs` |
| order-service | `localhost:8001/health` | `localhost:8001/docs` |
| inventory-service | `localhost:8002/health` | `localhost:8002/docs` |
| notification-service | `localhost:8003/health` | `localhost:8003/docs` |

## Design decisions

Significant architectural decisions are recorded as Architecture Decision Records in [`docs/adr/`](docs/adr/), following the standard context → decision → alternatives → consequences format. Notable ones:

- [ADR-0001: Database-per-service](docs/adr/0001-database-per-service.md)
- [ADR-0002: Kafka over RabbitMQ](docs/adr/0002-kafka-over-rabbitmq.md)
- [ADR-0003: Local JWT validation over introspection](docs/adr/0003-jwt-local-validation-vs-introspection.md)
- [ADR-0004: Transactional outbox for event publishing](docs/adr/0004-outbox-pattern-for-event-publishing.md)

## Build status

Waypoint is being built in layers, each one a complete, independently verifiable milestone rather than a partial slice of the whole system.

- [x] **Layer 1 — Service decomposition:** four independently deployable FastAPI services scaffolded, each with its own virtual environment and dependency set
- [ ] Database-per-service wired up via Docker Compose (one Postgres instance per service)
- [ ] Alembic migration scaffolding per service
- [ ] Dockerfiles per service
- [ ] **Layer 2 — Asynchronous communication:** Kafka event backbone, idempotent consumers
- [ ] **Layer 3 — Resilience:** circuit breakers, retries, Redis distributed locking, documented failure scenario
- [ ] **Layer 4 — Authentication:** JWT issuance and local validation
- [ ] **Layer 5 — Testing:** Pact contract tests, dockerized integration test suite

## Roadmap

Planned beyond the core five layers: CI via GitHub Actions running the full test suite on every push, a lightweight read-only dashboard for observing event flow through the system, and a documented "what I'd do differently at production scale" write-up covering Kafka cluster sizing, secrets management, and service mesh trade-offs.

---

**Author:** [Tyree Franklin Jr.](https://github.com/tyreefranklinjr)
