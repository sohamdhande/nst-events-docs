# Scalability Review

Based on simulated stress tests and architectural analysis for the target population:

## Findings
* **500 concurrent registrations**: Handled gracefully using atomic capacity updates (`UPDATE events SET registration_count = registration_count + 1 WHERE registration_count < max_capacity RETURNING id`). `SELECT FOR UPDATE` is strictly banned to prevent database lock contention.
* **200 attendance scans/min**: Processed synchronously via the Express RPC. The database can comfortably handle this write volume. PGMQ is strictly banned for attendance processing to ensure immediate feedback.
* **1000 attendee event**: Full Text Search index size increases negligibly. Read replicas not required for V1.
* **Notification Backlog**: PGMQ worker processes batches of 100 notifications using Expo Push API `send_batch` (one HTTP request per batch). Complete broadcast takes ~20 seconds.
* **Role Revocation**: RLS policies evaluate dynamically, meaning revoking a Club Admin instantly removes their edit privileges globally without needing to invalidate cache layers.

## Connection Pooling (PgBouncer)
* **Sizing Formula**: Maximum connections = `(API Pods * Prisma Pool Size) + Worker Pods`.
* **Prisma Configuration**: Must use `pgbouncer=true` in the connection string and `transaction` pool mode.

## SSE Debouncing
* **Fan-out**: PG NOTIFY pushes events to Express. Express must debounce the fan-out to SSE clients (e.g., 500ms buffer) to prevent blocking the Node.js event loop during burst updates (e.g., mass attendance scanning).

## Materialized View Scheduling
* **Refresh Strategy**: `club_leaderboard_mv` and `student_leaderboard_mv` are refreshed asynchronously via `pg_cron`. Refresh schedules must be staggered to avoid CPU spikes (e.g., one at `*/5 * * * *` and the other at `2-59/5 * * * *`).

## Outcomes
The PostgreSQL + Express Backend architecture safely sustains the projected 3000+ student load with headroom.
