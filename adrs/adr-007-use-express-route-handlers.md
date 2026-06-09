# ADR-007: Use Express Route Handlers and Service Modules for All Server-Side Logic

## Status
Accepted

## Context
Certain workflows cannot run securely on the client. QR token generation must be cryptographic and server-side to prevent clients from forging valid tokens. Attendance validation involves HMAC checks, PostGIS geofence verification, and atomic database writes — all of which require trusted server execution. Notification dispatch and background queue processing must run independently of any user session.

In the previous architecture, these tasks were handled by Supabase Edge Functions, which ran in a Deno runtime. With the move to a custom Express backend, all of this logic is co-located in the Express codebase as standard Node.js route handlers and service modules.

## Decision
All server-side logic runs inside the **Express backend** as route handlers or service modules written in **TypeScript for Node.js**. There is no separate serverless runtime.

Key server-side responsibilities:
- **QR Token Generation** (`POST /attendance/generate-qr`): Cryptographically generates the rotating TOTP seed. Enforces `CLUB_ADMIN` role check.
- **Attendance Validation** (`POST /attendance/mark`): HMAC validation, PostGIS geofence check, device collision detection, and atomic record write via Prisma.
- **Notification Worker**: A background service (setInterval or pg_cron triggered webhook) that polls `pgmq` and dispatches payloads to the Expo Push API.
- **Leaderboard Recalculation** (`POST /admin/leaderboard/recalculate`): Manual trigger for Platform Admin to re-aggregate points.
- **Slack/Webhook Integration**: Posts to Faculty channel when a new event enters `PENDING_APPROVAL` status.

## Consequences

**Positive:**
- All logic lives in one codebase — standard Node.js/TypeScript. No separate Deno runtime to learn, configure, or deploy.
- Express route handlers are unit-testable with standard tools (Jest, Supertest).
- Simpler local development setup: one `docker compose up` starts the Express server and PostgreSQL together.
- Easier debugging: structured logs, standard Node.js error handling, familiar ecosystem.

**Negative:**
- Team is responsible for scaling the Express server under high concurrency. Unlike serverless, a single Express instance does not auto-scale per request.
- Background workers (notification polling) must be managed carefully to avoid blocking the event loop.

## Alternatives Considered
* **AWS Lambda**: Rejected to keep the entire stack on the NST Cluster, avoiding external cloud dependencies and cost.
* **Supabase Edge Functions (Deno)**: Rejected along with the Supabase platform. Deno Edge Functions are tightly coupled to the Supabase runtime and cannot be self-hosted on the NST Cluster.

## Future Impact
Positions all server-side logic for straightforward testing, containerization, and horizontal scaling behind a load balancer as the platform grows beyond V1.

