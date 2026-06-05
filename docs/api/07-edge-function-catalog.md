# Edge Function Catalog

## `generate_qr`
* **Purpose**: Generates the live rotating TOTP payload for display on the organizer's screen.
* **Trigger**: HTTP GET
* **Inputs**: `session_id`
* **Outputs**: Encrypted QR Payload, Expiry.
* **Dependencies**: Redis (Rate limiting / Caching), Crypto modules.
* **Security Considerations**: Enforces `CLUB_ADMIN` role check.

## `notification_worker`
* **Purpose**: Empties the `pgmq` and sends payloads to Expo.
* **Trigger**: Cron / Webhook.
* **Dependencies**: Expo Push Server.
* **Security Considerations**: Runs with internal service keys.

## `slack_webhook`
* **Purpose**: Notifies the Faculty channel when a new event requires approval.
* **Trigger**: Database Webhook on `events` table (status = PENDING_APPROVAL).

## `leaderboard_recalculation`
* **Purpose**: Re-aggregates club points across the entire season if anomalies are detected.
* **Trigger**: Manual HTTP by Platform Admin.
