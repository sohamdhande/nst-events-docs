# Notification Data Model

## Strategy
Push notifications are ephemeral via Expo Push Delivery; in-app notifications are persistent. Push delivery requires a stored Expo push token per device, which is managed separately from the notification inbox.

## Tables
* **`notifications`**: Persistent inbox. Includes `read_at` timestamp and delivery tracking fields (`delivered_at`, `delivery_failed_at`, `delivery_error`).
* **`notification_preferences`**: Allows users to opt-out of specific notification types (e.g., "mute club announcements"). Controls whether `push_enabled` is true/false per category.
* **`push_tokens`**: Stores Expo push tokens per device. One row per `(user_id, device_id)` pair. Upserted on login, hard-deleted when Expo returns `DeviceNotRegistered`, and periodically cleaned up by `pg_cron` for tokens idle >90 days. This table is **required** for nst-worker to dispatch push notifications — without it, the worker has no delivery address for a given user's device.

## Push Token Flow
1. Mobile app obtains an Expo push token via `expo-notifications`.
2. On first authenticated request (or after token rotation), the mobile app sends the token to Express via `PATCH /users/me` or a dedicated endpoint.
3. Express upserts a `push_tokens` row: `ON CONFLICT (user_id, device_id) DO UPDATE SET expo_token = $3, last_seen_at = now()`.
4. When `nst-worker` dispatches a push notification, it queries `push_tokens WHERE user_id = $1` to get all active device tokens for that user.
5. If Expo returns `DeviceNotRegistered` for a specific token, nst-worker hard-deletes that `push_tokens` row.

## Processing Pipeline (pgmq integration)
We utilize **PGMQ** (Postgres Message Queue) to handle heavy broadcast notifications (e.g., "Event starts in 15 mins" sent to 1000 registered users). The queue worker processes batches and interfaces with the Expo Push API, updating the `notifications` table upon success.

