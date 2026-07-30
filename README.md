# Waypoint - Distributed Order Management System

Event-driven order fulfillment platform built to work through the architectural patterns that actually show up in production distributed systems: service decomposition with database-per-service isolation, asynchronous messaging with idempotent consumers, resilience under partial failure, and consumer-driven contract testing across service boundaries.

Four services, Auth, Order, Inventory, and Notification, each own their data outright and communicate over a mix of synchronous REST (only where a caller needs an immediate answer) and asynchronous events (everywhere something merely happened and downstream services can catch up on their own schedule).

## Architecture

<img width="1390" height="1044" alt="Waypoint architecture diagram" src="https://github.com/user-attachments/assets/91494f91-4bbf-4cab-a3da-08b8864facb4" />

An order placed against `order-service` triggers a synchronous JWT check against `auth-service`, then publishes `order.placed`. `inventory-service` consumes that event, reserves stock against a Redis-backed lock to avoid overselling under concurrent requests, and republishes either `inventory.reserved` or `inventory.failed`. `notification-service` consumes that outcome and closes the loop with the customer. No service calls another's database directly. Everything downstream of the initial request happens over the event bus, so `inventory-service` or `notification-service` being temporarily unavailable never blocks order acceptance.

Event delivery is treated as at-least-once, not exactly-once, because Kafka doesn't guarantee otherwise and pretending it does is how you end up double-charging a customer. Every consumer records processed event IDs in its own table, in the same transaction as the side effect it performs, so replays and duplicate deliveries are a no-op rather than a duplicate charge or double notification.

## Services

| Service | Owns | Public interface |
|---|---|---|
| `auth-service` | Users, hashed credentials, signing keys | `POST /login`, `POST /token/refresh`, JWKS endpoint |
| `order-service` | Orders, order line items | `POST /orders`, `GET /orders/{id}`, publishes `order.placed` |
| `inventory-service` | Product stock levels | `GET /products/{id}`, consumes `order.placed`, publishes `inventory.reserved` / `inventory.failed` |
| `notification-service` | Notification log | consumes `inventory.reserved` / `inventory.failed` |

Each is a standalone FastAPI application with its own virtual environment, its own frozen dependency set, its own Dockerfile, and its own database. Nothing is imported across service boundaries. That's not a style preference, it's what makes it possible to redeploy `inventory-service` on its own without coordinating a release with the other three.

## Stack

| | |
|---|---|
| API | FastAPI, Uvicorn |
| Data | PostgreSQL, one instance per service, no shared schema |
| Migrations | Alembic, per service |
| Messaging | Kafka |
| Locking | Redis |
| Resilience | tenacity for retry and backoff, circuit breaker on synchronous calls |
| Auth | JWT, RS256, local signature validation via JWKS rather than per-request introspection |
| Testing | pytest, Pact for consumer-driven contracts |
| Orchestration | Docker, Docker Compose |

Kafka over RabbitMQ was a deliberate call. The deciding factor was consumer offset replay, since testing idempotency properly means being able to reprocess a topic from an earlier offset on demand, and that's a first-class Kafka operation in a way it isn't in Rabbit's model. Full reasoning is in `docs/adr/0002-kafka-over-rabbitmq.md`.

## Running it

```bash
git clone https://github.com/tyreefranklinjr/Waypoint.git
cd Waypoint
docker compose up --build
```

| Service | Port | Docs |
|---|---|---|
| auth-service | 8000 | `/docs` |
| order-service | 8001 | `/docs` |
| inventory-service | 8002 | `/docs` |
| notification-service | 8003 | `/docs` |

Each service also ships its own `.env.example`. Copy it to `.env` and fill in local values before running outside Compose.

## Design decisions

Architecturally significant calls are written up as ADRs rather than buried in commit history: context, decision, alternatives considered, consequences. See [`docs/adr/`](docs/adr/):

- [0001, Database-per-service](docs/adr/0001-database-per-service.md)
- [0002, Kafka over RabbitMQ](docs/adr/0002-kafka-over-rabbitmq.md)
- [0003, Local JWT validation over introspection](docs/adr/0003-jwt-local-validation-vs-introspection.md)
- [0004, Transactional outbox for event publishing](docs/adr/0004-outbox-pattern-for-event-publishing.md)

## Status

Layer 1, service decomposition, is complete: four services, four isolated databases, independently containerized and independently deployable, verified by tearing one down and rebuilding it without touching the other three. Layers 2 through 5, async messaging, resilience, auth, and contract testing, are in progress.

<details>
<summary>Full build log</summary>

**Layer 1, service decomposition** (done)
- Four FastAPI services, isolated virtual environments and dependency sets
- `/health` endpoint per service, verified locally
- `requirements.txt` frozen per service
- `.env.example` per service (JWT config for auth, inter-service URLs for order, DB vars throughout)
- Dockerfile per service, `.dockerignore` excluding local venvs
- PostgreSQL per service wired through `docker-compose.yml`, one network, isolated credentials
- Alembic initialized per service
- Independent redeploy verified, rebuilding one service does not restart the others
- Database boundary verified directly, no service's database contains another's tables

**Layer 2, asynchronous communication** (in progress)
Kafka event backbone, idempotent consumers via a per-service `processed_events` table, transactional outbox for reliable publishing.

**Layer 3, resilience** (planned)
tenacity retries with exponential backoff on the Auth sync call, circuit breaker around it, Redis-backed distributed lock on stock reservation, documented kill-a-service failure scenario.

**Layer 4, authentication** (planned)
Auth service issues RS256-signed JWTs. Other services validate locally against the published JWKS rather than calling back to Auth on every request.

**Layer 5, testing** (planned)
Pact consumer-driven contracts between service pairs, pytest integration suite run against `docker-compose.test.yml`.

</details>

---

[Tyree Franklin Jr.](https://github.com/tyreefranklinjr)
