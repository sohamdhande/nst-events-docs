# Notification Data Model

## Strategy
Push notifications are ephemeral via Expo Push Delivery; in-app notifications are persistent.

## Tables
* **`notifications`**: Persistent inbox. Includes `read_at` timestamp.
* **`notification_preferences`**: Allows users to opt-out of specific notification types (e.g., "mute club announcements").

## Processing Pipeline (pgmq integration)
We utilize **PGMQ** (Postgres Message Queue) to handle heavy broadcast notifications (e.g., "Event starts in 15 mins" sent to 1000 registered users). The queue worker processes batches and interfaces with the Expo Push API, updating the `notifications` table upon success.
