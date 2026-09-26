# 03 — Compute Workloads, Pods, Scaling & Multi-Replica State

This document details the containerized application microservices running inside the `nst-events` Kubernetes namespace. It describes how each workload operates, scales, performs health checks, and coordinates state across multiple replicas.

---

## 1. Application Workload Architecture

The NST Events platform separates user-facing traffic, frontend rendering, and background computation into three discrete container workloads:

```text
 ┌──────────────────────────────────────────────────────────────────────────────────────────┐
 │                                   KUBERNETES NAMESPACE: nst-events                       │
 │                                                                                          │
 │  ┌──────────────────────────────────────────────┐        ┌────────────────────────────┐  │
 │  │ nst-dashboard (Next.js Standalone)           │        │ nst-api (Express.js REST)  │  │
 │  │ • Replicas: 2 (High Availability)            │        │ • Replicas: 2 (HA)         │  │
 │  │ • CPU: 50m req / 250m lim                    │        │ • CPU: 100m req / 500m lim │  │
 │  │ • RAM: 128Mi req / 256Mi lim                 │        │ • RAM: 128Mi req / 512Mi   │  │
 │  │ • Port: 3000 (HTTP)                          │        │ • Port: 3001 (HTTP)        │  │
 │  │ • Probes: /login                             │        │ • Probes: /ready, /health  │  │
 │  │ • PodDisruptionBudget: minAvailable 1        │        │ • PDB: minAvailable 1      │  │
 │  └──────────────────────┬───────────────────────┘        └─────────────┬──────────────┘  │
 │                         │                                              │                 │
 │                         │ Internal HTTP Proxy (ClusterIP)              │                 │
 │                         └─────────────────────────────────────────────►│                 │
 │                                                                        │                 │
 │                                             ┌──────────────────────────┴──────────────┐  │
 │                                             │ Writes new notification jobs to DB      │  │
 │                                             ▼                                         │  │
 │  ┌───────────────────────────────────────────────────────┐                            │  │
 │  │ nst-postgres (PostgreSQL 16 + PostGIS 3.4)            │                            │  │
 │  │ • Replicas: 1 (StatefulSet)                           │                            │  │
 │  │ • Port: 5432 (TCP)                                    │                            │  │
 │  │ • Persistent Volume: 10Gi (Longhorn 3-Way Replicated) │                            │  │
 │  │ • Table: notification_jobs                            │                            │  │
 │  │ • Pub/Sub: PostgreSQL LISTEN / NOTIFY                 │                            │  │
 │  └──────────────────────────┬────────────────────────────┘                            │  │
 │                             │                                                         │  │
 │                             │ Atomic Queue Poll: SELECT ... FOR UPDATE SKIP LOCKED    │  │
 │                             ▼                                                         │  │
 │  ┌───────────────────────────────────────────────────────┐                            │  │
 │  │ nst-worker (Node 20 Background Engine)                │                            │  │
 │  │ • Replicas: Strictly 1                                │                            │  │
 │  │ • Strategy: Recreate                                  │                            │  │
 │  │ • CPU: 50m req / 250m lim                             │                            │  │
 │  │ • RAM: 128Mi req / 256Mi lim                          │                            │  │
 │  │ • Port: 3002 (Internal Health & Prometheus Metrics)   │                            │  │
 │  │ • Probes: /ready, /health                             │                            │  │
 │  └──────────────────────────┬────────────────────────────┘                            │  │
 └─────────────────────────────┼─────────────────────────────────────────────────────────┘
                               │ Outbound Push Notifications
                               ▼
            [ Expo Push Notification Servers (APNs / FCM) ]
```

---

## 2. API Service (`nst-api`)

The API service is the core business engine. It processes authentication, event registrations, ticket issuing, leaderboard updates, club management, and live event data.

### Deployment Specifications
- **Runtime**: Node.js 20 on Alpine Linux (`docker/Dockerfile.api`).
- **Framework**: Express.js with TypeScript (`apps/api`).
- **Replica Count**: **2 replicas** running continuously for High Availability (HA).
- **Service Exposure**: Kubernetes `ClusterIP` named `nst-api-service` exposing port 80 (routing to container port 3001).
- **Security Hardening**: `automountServiceAccountToken: false` to ensure compromised containers cannot steal cluster API tokens.

### Health, Readiness & Disruption Guard
- **Readiness Probe (`/ready`)**:
  - Pings the database with `SELECT 1` via Prisma.
  - If PostgreSQL is temporarily restarting or uncontactable, the pod drops out of the Traefik load balancer pool within 10 seconds, preventing 500 errors from reaching students.
- **Liveness Probe (`/health`)**:
  - Verifies the Node.js event loop is responsive. Initial delay: 15s, period: 20s.
- **PodDisruptionBudget (`nst-api-pdb`)**:
  - Enforces `minAvailable: 1`. During node maintenance or rolling cluster upgrades, Kubernetes will never terminate both API pods simultaneously.

### Multi-Replica State Synchronization
Running multiple API replicas requires strict coordination for stateful operations:

1. **Stateless Authentication**:
   - Authentication tokens are signed JSON Web Tokens (JWTs) stored in HTTP-only, secure, partitioned cookies (or `Authorization: Bearer` headers for mobile).
   - Because no session state lives in container memory, any API replica can authenticate and fulfill any request without sticky sessions.
2. **Real-Time Cross-Pod SSE Broadcasting (`pgListener`)**:
   - When a student's phone opens an SSE stream (`/v1/events/:id/live`), the request hits **Replica A**.
   - If an event coordinator on the dashboard hits **Replica B** to announce a venue change or check-in count, Replica B executes a database update.
   - To ensure students connected to Replica A receive the announcement immediately, `apps/api/src/modules/sse/pg-listener.ts` utilizes **PostgreSQL `LISTEN / NOTIFY`**:
     ```text
     [ Coordinator on Replica B ] ──> UPDATE events ──> PG: NOTIFY "event:<id>"
                                                              │
                                     ┌────────────────────────┴────────────────────────┐
                                     ▼                                                 ▼
                          [ Replica A: LISTEN ]                             [ Replica B: LISTEN ]
                                     │                                                 │
                          Emits to SSE Client                               Emits to SSE Client
     ```
   - Each API replica maintains an active PostgreSQL listening socket. When a notification is emitted, PostgreSQL broadcasts it to all API pods, ensuring 100% of connected clients receive live updates regardless of load balancer distribution.

---

## 3. Web Dashboard (`nst-dashboard`)

The dashboard provides the web interface for students, club leaders, event organizers, and platform administrators.

### Deployment Specifications
- **Runtime**: Next.js 15 App Router built with `output: 'standalone'` (`docker/Dockerfile.dashboard`).
- **Replica Count**: **2 replicas** for zero-downtime rolling updates.
- **Service Exposure**: Kubernetes `ClusterIP` named `nst-dashboard-service` exposing port 80 (routing to container port 3000).
- **Resource Allocations**:
  - Requests: 50m CPU, 128Mi RAM.
  - Limits: 250m CPU, 256Mi RAM.
- **Health Probes**: Liveness and Readiness probes monitor HTTP GET `/login` on port 3000.
- **PodDisruptionBudget (`nst-dashboard-pdb`)**: Enforces `minAvailable: 1`.

### Server-Side Rendering & Internal Proxying
The dashboard does **not** communicate directly with PostgreSQL. All data fetching is executed via HTTP against `nst-api-service`:
- During Server-Side Rendering (SSR) in Node.js, Next.js calls `http://nst-api-service` over the high-speed internal pod network.
- In the student's browser, client-side requests are transparently routed through the Next.js rewrite engine to the API service.

---

## 4. Background Job Worker (`nst-worker`)

The worker handles asynchronous operations that must not block user HTTP requests:
- Delivering push notifications to student phones via Apple Push Notification service (APNs) and Google Firebase Cloud Messaging (FCM).
- Polling Expo receipt IDs to verify delivery status.
- Retrying failed notification batches with exponential backoff.
- Scheduled campus event reminders and waitlist transitions.

### Why Strictly 1 Replica? (`replicas: 1`)
Unlike the stateless API, background job processors risk **race conditions, duplicate message deliveries, and split-brain execution** if scaled horizontally without complex distributed lock managers. 
- The worker deployment enforces `replicas: 1`.
- It uses the deployment strategy:
  ```yaml
  strategy:
    type: Recreate
  ```
  During automated deployments, Kubernetes terminates the running worker pod completely **before** launching the new version. This guarantees two versions of the worker are never running concurrently against the database queue.

### Atomic Queue Polling (`FOR UPDATE SKIP LOCKED`)
The worker loop (`apps/worker/src/worker.ts`) polls the PostgreSQL `notification_jobs` table using PostgreSQL's row-level locking primitives:

```sql
SELECT * FROM notification_jobs 
WHERE 
  (status IN ('PENDING', 'RETRY_PENDING') AND available_at <= now())
  OR (status = 'PROCESSING' AND locked_at <= now() - interval '5 minutes')
  OR (status = 'WAITING_FOR_RECEIPTS' AND available_at <= now())
LIMIT 50 
FOR UPDATE SKIP LOCKED;
```

#### Why This Mechanism Is Resilient:
1. **Atomic Claiming**: Rows are selected and transitioned to `status: 'PROCESSING'` within a tight 10-second database transaction.
2. **Skip Locked**: If an ad-hoc operator script or secondary process inspects the table, conflicting rows are safely skipped without deadlocks.
3. **Dead-Worker Recovery**: If the worker pod crashes mid-execution, the `locked_at <= now() - interval '5 minutes'` clause automatically releases abandoned jobs so the restarted worker can pick them up.

### Internal Monitoring Server (Port 3002)
Although the worker has no public ingress route, it runs an internal Express server on port 3002:
- **`GET /health`**: Returns HTTP 200 indicating the event loop is responsive.
- **`GET /ready`**: Verifies the worker is initialized and connected to PostgreSQL (`SELECT 1`).
- **`GET /metrics`**: Exposes standard Prometheus metrics:
  - `queueDepth{status="PENDING"}`: Real-time count of waiting jobs.
  - `queueDepth{status="PROCESSING"}`: In-flight jobs.
  - `processingDuration{job_type="..."}`: Histogram tracking execution time per job type.

### Graceful Termination Lifecycle
When Kubernetes rolls out an update or drains a node:
1. Kubernetes dispatches a `SIGTERM` signal to the worker pod.
2. The worker sets `isShuttingDown = true` and ceases claiming new batches.
3. It waits up to `WORKER_SHUTDOWN_TIMEOUT_MS` (default: 10,000 ms) for any active batch to complete and write its status back to PostgreSQL.
4. It cleanly closes Prisma database pools and exits with code 0.
