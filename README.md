---

```markdown
# Cart Service

The Cart Service manages shopping cart state for users of the HipsterShop platform. It is a C# (.NET 8) microservice exposing an HTTP REST API, with pluggable backend storage that supports Redis, MongoDB, PostgreSQL, Google Cloud Spanner, and Google Cloud AlloyDB.

## Table of Contents

- [Architecture](#architecture)
- [API Reference](#api-reference)
- [Configuration](#configuration)
- [Storage Backends](#storage-backends)
- [Running Locally](#running-locally)
- [Docker](#docker)
- [Observability](#observability)
- [CI/CD](#cicd)

---

## Architecture

```
┌────────────────────────────────────────┐
│             Cart Service               │
│                                        │
│  CartController (HTTP :7070)           │
│       │                                │
│  ICartStore (pluggable)                │
│  ├── RedisCartStore    (default/dev)   │
│  ├── MongoCartStore                    │
│  ├── PostgresCartStore                 │
│  ├── SpannerCartStore                  │
│  └── AlloyDBCartStore                  │
└────────────────────────────────────────┘
```

Backend selection happens at startup based on environment variables (see [Configuration](#configuration)). If no external store is configured the service falls back to an in-memory cache suitable for local development.

---

## API Reference

All endpoints accept and return `application/json`. The service listens on port **7070**.

### `POST /cart/add`

Add an item to a user's cart, or increment its quantity if it already exists.

**Request**
```json
{
  "userId": "user-abc",
  "item": {
    "productId": "prod-123",
    "quantity": 2
  }
}
```

**Response** `200 OK`
```json
{}
```

---

### `POST /cart/get`

Retrieve all items in a user's cart.

**Request**
```json
{
  "userId": "user-abc"
}
```

**Response** `200 OK`
```json
{
  "userId": "user-abc",
  "items": [
    { "productId": "prod-123", "quantity": 2 },
    { "productId": "prod-456", "quantity": 1 }
  ]
}
```

---

### `POST /cart/empty`

Remove all items from a user's cart.

**Request**
```json
{
  "userId": "user-abc"
}
```

**Response** `200 OK`
```json
{}
```

---

### `GET /_healthz`

Health check endpoint used by Kubernetes liveness/readiness probes.

**Response** `200 OK` when the service and its configured backend are reachable.

---

## Configuration

All configuration is provided via environment variables.

### Core

| Variable | Description | Default |
|---|---|---|
| `ASPNETCORE_HTTP_PORTS` | Port the service listens on | `7070` |
| `DISABLE_TRACING` | Set to any value to disable OpenTelemetry tracing | _(unset)_ |
| `DOTNET_EnableDiagnostics` | .NET diagnostics; set to `0` in production images | `0` |

### Storage Backend Selection

The service picks a backend in this priority order:

1. **MongoDB** — if `MONGO_URI` is set
2. **Redis** — if `REDIS_ADDR` is set
3. **In-memory** — fallback for local development

PostgreSQL, Spanner, and AlloyDB require additional code-level wiring (see [Storage Backends](#storage-backends) below for details).

### Redis

| Variable | Description |
|---|---|
| `REDIS_ADDR` | Redis server address, e.g. `redis:6379` |

### MongoDB

| Variable | Description | Default |
|---|---|---|
| `MONGO_URI` | MongoDB connection string | _(required)_ |
| `MONGO_DATABASE` | Database name | `cart_db` |

### Google Cloud Spanner

| Variable | Description | Default |
|---|---|---|
| `SPANNER_CONNECTION_STRING` | Full Spanner connection string | _(optional)_ |
| `SPANNER_PROJECT` | GCP project ID | _(required if not using connection string)_ |
| `SPANNER_INSTANCE` | Spanner instance name | `onlineboutique` |
| `SPANNER_DATABASE` | Spanner database name | `carts` |

### Google Cloud AlloyDB

| Variable | Description |
|---|---|
| `PROJECT_ID` | GCP project ID (used for Secret Manager) |
| `ALLOYDB_SECRET_NAME` | Secret Manager secret name for the database password |
| `ALLOYDB_PRIMARY_IP` | AlloyDB primary instance IP |
| `ALLOYDB_DATABASE_NAME` | Database name |
| `ALLOYDB_TABLE_NAME` | Table name |

---

## Storage Backends

All backends implement `ICartStore`:

```csharp
interface ICartStore {
    Task AddItemAsync(string userId, string productId, int quantity);
    Task EmptyCartAsync(string userId);
    Task<Cart> GetCartAsync(string userId);
    bool Ping();
}
```

| Backend | Store Class | Notes |
|---|---|---|
| In-memory | `RedisCartStore` (memory cache) | Development only; no persistence |
| Redis | `RedisCartStore` | Cart stored as JSON string, keyed by `userId` |
| MongoDB | `MongoCartStore` | One document per user in the `carts` collection |
| PostgreSQL | `PostgresCartStore` | Table: `carts(user_id, cart_data bytea)` |
| Spanner | `SpannerCartStore` | Table: `CartItems(userId, productId, quantity)` |
| AlloyDB | `AlloyDBCartStore` | Password fetched from Google Secret Manager at startup |

---

## Running Locally

**Prerequisites:** .NET 8.0 SDK

```bash
# Start a local Redis (optional, falls back to in-memory if omitted)
docker run -d -p 6379:6379 redis:alpine

cd src
dotnet restore
dotnet run
```

The service is available at `http://localhost:7070`.

**Quick smoke test:**
```bash
# Add an item
curl -s -X POST http://localhost:7070/cart/add \
  -H "Content-Type: application/json" \
  -d '{"userId":"u1","item":{"productId":"p1","quantity":1}}'

# Retrieve the cart
curl -s -X POST http://localhost:7070/cart/get \
  -H "Content-Type: application/json" \
  -d '{"userId":"u1"}'
```

---

## Docker

### Production image

Built on `mcr.microsoft.com/dotnet/aspnet:8.0`. Runs as non-root user (UID 1000).

```bash
docker build -f src/Dockerfile -t cartservice:latest .

docker run -p 7070:7070 \
  -e REDIS_ADDR=redis:6379 \
  cartservice:latest
```

### Debug image

Built on .NET 10.0 with diagnostics enabled.

```bash
docker build -f src/Dockerfile.debug -t cartservice:debug .
```

---

## Observability

### Tracing

The service uses [OpenTelemetry](https://opentelemetry.io/) with ASP.NET Core instrumentation and an OTLP exporter. Traces are emitted to whatever OTel Collector endpoint is configured in the environment (e.g. Jaeger, Datadog, Google Cloud Trace).

Set `DISABLE_TRACING=true` to turn off tracing entirely.

### Health checks

The `/_healthz` endpoint verifies that the service and its backend are reachable. Use this for Kubernetes liveness and readiness probes:

```yaml
livenessProbe:
  httpGet:
    path: /_healthz
    port: 7070
readinessProbe:
  httpGet:
    path: /_healthz
    port: 7070
```

### Logging

Structured logs are written to stdout at `Information` level by default (configurable in `appsettings.json`). All cart operations emit a log line including the `userId`.

---

## CI/CD

The pipeline in `.github/workflows/cartservice.yaml` triggers on pushes and pull requests targeting `main` or `develop` when files under `src/` change.

| Stage | Trigger | Description |
|---|---|---|
| Sonar + Snyk scan | PR only | Static analysis and dependency vulnerability scan |
| Build & tag image | Push / PR | Multi-arch Docker build; Trivy image scan on PRs |
| Push image | Push to main/develop | Pushes tagged image to Docker Hub |
| Deploy to dev | Push to develop | Auto-updates Helm values for the dev environment |
| Production approval | Push to main | Manual approval gate before production deploy |
| Deploy to prod | Post-approval | Updates Helm values for the production environment |
| Notify | Always | Sends result email via SendGrid |

**Required secrets:** `SNYK_TOKEN`, `SONAR_TOKEN`, `SONAR_HOST_URL`, `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `HELM_CHARTS_TOKEN`, `SENDGRID_API_KEY`
```
