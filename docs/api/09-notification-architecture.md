# Notification Architecture

## Queue Flow
1. **Trigger**: An RPC (e.g., `approve_event`) inserts a message into the `pgmq` queue table atomically within the same transaction.
2. **Poll**: The `nst-worker` Kubernetes Deployment polls pgmq every 5 seconds (batch of 100 messages). See `docs/api/13-worker-deployment.md` for the full deployment model.
3. **Delivery**: The worker calls the Expo Push API.
4. **Finalization**: Success/Fail states are written back to the `notifications` table via Prisma.

## Push Delivery & In-App Lifecycle
All critical alerts are written to the `notifications` table (In-App) *first*. Push delivery is treated as an enhancement. If Push fails, the user still sees the notification inside the app.

## Retry Strategy
Exponential backoff (1m, 5m, 15m) for network failures to Expo.

## Dead Letter Strategy
If a message fails 3 times, it is moved to the DLQ.

## Rate Limiting & Delivery Tracking
To prevent spam, clubs are limited to 2 broadcast notifications per week. Delivery tracking is handled by correlating Expo receipts back to the notification records.
