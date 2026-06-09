# Notification Worker — Deployment Model

## Architecture Decision: Separate Kubernetes Deployment

The notification worker runs as a **separate Kubernetes Deployment** (`nst-worker`) on the NST Cluster. It is not co-located with the API server (`nst-api`). This separation:

- Prevents pgmq polling from competing with HTTP request processing for CPU/memory
- Allows the worker to be scaled, restarted, and deployed independently from the API
- Allows the API to restart during deployments without dropping in-flight queue consumers

---

## Deployment Topology

```text
NST Cluster (K3s, Worker Nodes: 8GB RAM limit)
│
├── Deployment: nst-api
│   ├── Replicas: 2–3
│   ├── Handles: All HTTP requests from Expo + Next.js clients
│   └── Does NOT poll pgmq
│
├── Deployment: nst-worker
│   ├── Replicas: 1
│   ├── Handles: pgmq polling + Expo Push delivery
│   └── Does NOT handle HTTP traffic
│
└── StatefulSet / External: PostgreSQL
    └── Shared by both nst-api and nst-worker
    └── Requires dedicated node or scale down to 2-node CNPG to fit in 8GB RAM constraints.
```

**Worker replica count is intentionally 1.** pgmq uses visibility timeouts to prevent duplicate message processing, but running multiple consumers polling the same queue can still cause race conditions in low-throughput scenarios. For V1 at NST scale, a single worker replica is sufficient (handles 50 notifications/sec).

---

## Worker Internals

The worker is a separate Node.js process (`worker/index.ts`) in the same monorepo. It runs an event loop that:

```
1. Poll pgmq for up to 100 messages (batch read with visibility timeout = 30s)
2. For each message, call Expo Push API
3. On success: delete message from queue, write delivery status to notifications table
4. On failure: leave message in queue (visibility timeout resets after 30s = automatic retry)
5. After 3 failures: pgmq moves message to Dead Letter Queue (DLQ)
6. Wait 5 seconds, repeat
```

---

## Operational Constraints & Resource Limits

**CRITICAL:** The NST Cluster worker nodes are limited to **8GB RAM** each.
- A strict 3-node CNPG + Distributed MinIO deployment **will cause Out-Of-Memory (OOM) cascading failures** on these nodes.
- **V1 Deployment Model:** Downgrade to a 2-node CNPG cluster and use external S3 for backups, OR request dedicated infrastructure for the database.
- Multi-replica SSE fan-out (via PG NOTIFY) and strict 3-node HA requirements are considered "Future Scaling Options" if the cluster is upgraded.

---

## Retry & Failure Handling

| Scenario | Behavior |
|---|---|
| Expo Push API returns 5xx | Message returns to queue after 30s visibility timeout |
| Expo Push API returns `DeviceNotRegistered` | Mark notification as permanently undeliverable; do not retry |
| Worker pod crashes | pgmq visibility timeout expires after 30s; another pod (on restart) picks it up |
| 3 consecutive delivery failures | Message moved to Dead Letter Queue (`pgmq_dlq`) for Platform Admin review |
| Network partition | Messages accumulate in pgmq; delivered in batch once connectivity restores |

---

## Queue Consumer Deduplication

pgmq's visibility timeout mechanism prevents duplicate processing:
- When the worker reads a message, it becomes invisible to other consumers for 30 seconds
- If the worker crashes before deleting the message, it reappears after 30s
- Messages are deleted **only after confirmed delivery or permanent failure**

---

## Delivery Status Write-Back

After each delivery attempt, the worker updates the `notifications` table via Prisma:

```typescript
await prisma.notification.update({
  where: { id: notificationId },
  data: {
    delivered_at: new Date(),         // on success
    delivery_failed_at: new Date(),   // on permanent failure
    delivery_error: errorMessage,
  }
});
```

---

## K3s Manifest Summary

```yaml
# nst-worker Deployment (abbreviated)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nst-worker
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: worker
          image: ghcr.io/nst/nst-worker:latest
          command: ["node", "dist/worker/index.js"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: nst-secrets
                  key: DATABASE_URL
            - name: EXPO_ACCESS_TOKEN
              valueFrom:
                secretKeyRef:
                  name: nst-secrets
                  key: EXPO_ACCESS_TOKEN
      restartPolicy: Always
```

---

## What Has Changed From the Previous Description

Prior documentation described the worker as "a background service module in the Express process." That model was a placeholder. The correct model for production is a **separate Deployment**, as documented here. Any reference to an in-process background worker in other documents should be read as referring to the `nst-worker` Deployment described in this file.
