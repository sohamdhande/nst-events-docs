# Express Route Handlers Architecture

## Why Express Route Handlers Handle Server-Side Logic
The Express backend is responsible for all tasks that cannot safely or correctly execute on the client. This includes cryptographic operations, external API calls, and background queue processing.

There is no separate serverless runtime. All server-side logic runs within the Express Node.js process as route handlers or in the separate `nst-worker` deployment.

## When to Use an Express Route Handler
* **Cryptographic Generation**: Securely generating the rotating TOTP seeds for QR attendance (`POST /attendance/generate-qr`). The TOTP secret never leaves the server.
* **External API Integration**: Sending webhooks to Slack or Discord when events enter `PENDING_APPROVAL`; dispatching emails via providers like Resend.
* **Queue Workers**: Handled by the separate `nst-worker` deployment, which polls `pgmq` to dispatch Push Notifications to the Expo Push API.
* **Attendance Validation**: HMAC validation + PostGIS geofence check + device collision detection before writing the attendance record via Prisma.

## When NOT to Use a Route Handler for Heavy Logic
* **Standard CRUD reads**: Use a plain Prisma query in the route handler. Do not over-engineer simple data retrieval.
* **Complex Data Aggregation**: Delegate JOIN-heavy queries to PostgreSQL RPCs or Prisma's native relational queries. Let the database engine handle set operations.

## When NOT to Use Client-Side Logic
Clients must never:
- Generate or validate TOTP tokens
- Call the database directly
- Issue their own JWTs
- Access storage with raw credentials

All such operations are gated behind Express route handlers.

## Queue Consumer (nst-worker)

The `pgmq` queue consumer runs as a **separate Kubernetes Deployment** (`nst-worker`), not inside the API process. See [13-worker-deployment.md](./13-worker-deployment.md) for the full deployment model, retry strategy, and failure handling.

The API process (`nst-api`) only **enqueues** messages via RPCs. It never polls the queue.
