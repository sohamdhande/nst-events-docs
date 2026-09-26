# Database Connections, Pooling, & Role Management

This document defines the database connection architecture, client pooling configurations, and role privilege boundaries across the NST-Events platform.

---

## 1. Connection Topology & Application Clients

```mermaid
graph TD
    subgraph "Kubernetes Cluster (nst-events namespace)"
        API1[apps/api Pod 1]
        API2[apps/api Pod 2]
        Worker[apps/worker Pod]
        Migrate[DB Migration Job]
    end

    subgraph "PostgreSQL 16 (StatefulSet)"
        PG[(nst-postgres:5432)]
    end

    API1 -->|Prisma Pool: 15 conn as nst_app| PG
    API1 -->|pg.Client: 1 persistent conn for LISTEN| PG
    API2 -->|Prisma Pool: 15 conn as nst_app| PG
    API2 -->|pg.Client: 1 persistent conn for LISTEN| PG
    Worker -->|Prisma Pool: 10 conn as nst_worker| PG
    Migrate -->|Direct: 1 conn as postgres admin| PG
```

### Connection Matrix
| Client | Protocol / Driver | PostgreSQL Role | Connection String Secret | Pool Size / Lifetime | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **API Queries & RPCs** | Prisma Client (`@prisma/client`) | `nst_app` | `DATABASE_URL` | Pool: 10–15 per pod | All REST API user operations, auth checks, and stored procedure calls. Bound by RLS. |
| **API SSE Realtime** | Native `pg.Client` (`node-postgres`) | `nst_app` | `DATABASE_URL` | Exactly **1 persistent connection** per pod | Listens to PostgreSQL `LISTEN` channels for real-time SSE event updates. |
| **Worker Queue** | Prisma Client (`@prisma/client`) | `nst_worker` | `WORKER_DATABASE_URL` | Pool: 5–10 per pod | Polling `notification_jobs` via `SKIP LOCKED`, updating delivery statuses. |
| **Migrations & CI** | Prisma CLI | `postgres` | `ADMIN_DATABASE_URL` | Ephemeral (1–2 connections) | Running DDL migrations, extension creation, and database maintenance. |

---

## 2. PostgreSQL Roles & Minimum Privilege Model

PostgreSQL enforces strict role separation. The application code NEVER executes as `postgres` or with superuser privileges:

```sql
-- Role Creation DDL (from migration 20260805112600 & 20260827151000)
CREATE ROLE nst_app WITH LOGIN PASSWORD '***' NOSUPERUSER NOBYPASSRLS NOCREATEDB NOCREATEROLE NOREPLICATION;
CREATE ROLE nst_worker WITH LOGIN PASSWORD '***' NOSUPERUSER NOBYPASSRLS NOCREATEDB NOCREATEROLE NOREPLICATION;
```

### Privilege Comparison Matrix
| Resource / Operation | `postgres` (Admin) | `nst_app` (API) | `nst_worker` (Worker) |
| :--- | :---: | :---: | :---: |
| **Bypass RLS?** | **YES** | **NO (`NOBYPASSRLS`)** | **NO (`NOBYPASSRLS`)** |
| **DDL (`CREATE TABLE`, etc.)** | **FULL** | **NONE** | **NONE** |
| **`users` Table** | **FULL** | SELECT, INSERT, UPDATE (via RLS) | SELECT ONLY (for notification lookups) |
| **`events`, `teams`, `clubs`** | **FULL** | SELECT, INSERT, UPDATE, DELETE (via RLS) | **NO ACCESS** |
| **`attendance_records`** | **FULL** | SELECT, INSERT (via RLS / RPC) | **NO ACCESS** |
| **`notification_jobs`** | **FULL** | **INSERT ONLY** | **SELECT, UPDATE, DELETE** |
| **`notifications`** | **FULL** | SELECT, INSERT, UPDATE (via RLS) | **UPDATE ONLY (delivery timestamp fields)** |
| **`push_tokens`** | **FULL** | SELECT, INSERT, UPDATE (via RLS) | **SELECT ONLY** |
| **`notification_preferences`** | **FULL** | SELECT, INSERT, UPDATE (via RLS) | **SELECT ONLY** |

---

## 3. Connection Sizing & Pool Limits

PostgreSQL runs with a configured ceiling of `max_connections = 100`:

$$\text{Active Connections} = (N_{\text{API pods}} \times 15) + (N_{\text{API pods}} \times 1) + (N_{\text{Worker pods}} \times 10) + N_{\text{Admin/Maintenance}}$$

### Sizing for 2 API Replicas and 1 Worker Replica:
- `apps/api` Prisma Pools: $2 \times 15 = 30$
- `apps/api` SSE Listeners: $2 \times 1 = 2$
- `apps/worker` Prisma Pool: $1 \times 10 = 10$
- Scheduled Jobs & Migrations: $5$
- **Total Peak Active Connections:** $\sim 47$ connections
- **Headroom Remaining:** $53$ connections (allows scaling API up to 4 pods under peak traffic spikes).

### Connection String Configuration
In production secrets, connection limits and timeouts must be explicitly tuned in the URI:
```
# API Connection URI:
postgresql://nst_app:SECRET@nst-postgres-service:5432/nst_events?schema=public&connection_limit=15&pool_timeout=10

# Worker Connection URI:
postgresql://nst_worker:SECRET@nst-postgres-service:5432/nst_events?schema=public&connection_limit=10&pool_timeout=10
```

---

## 4. The Long-Lived SSE Connection (`pg-listener.ts`)

`apps/api/src/modules/sse/pg-listener.ts` manages a single dedicated `pg.Client` connection per API pod:
- **Protocol:** Uses native PostgreSQL `LISTEN` commands (e.g. `LISTEN "event_<eventId>_live"`).
- **Life Cycle:**
  - Connects on application startup.
  - Automatically re-subscribes to all active channels upon unexpected connection drop with exponential reconnection backoff.
  - Emits incoming notifications directly to the in-memory `sseEventBus`.
  - Cleanly executes `UNLISTEN *` and terminates connection on `SIGTERM` / `SIGINT`.
