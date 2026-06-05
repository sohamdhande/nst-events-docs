# Edge Function Architecture

## Why Edge Functions Exist
Deno-based Supabase Edge Functions exist to handle tasks that cannot or should not execute within PostgreSQL.

## When to use them
* **Cryptographic Generation**: Securely generating the rotating TOTP seeds for QR attendance.
* **External API Integration**: Sending webhooks to Slack, Discord, or Email providers (Resend).
* **Queue Workers**: Polling `pgmq` to dispatch Push Notifications to the Expo backend.

## When NOT to use them
* **Standard CRUD**: Never use an edge function to simply insert a row. Use PostgREST.
* **Complex Data Aggregation**: Let PostgreSQL handle JOINS and groupings.

## Queue Integration
Edge functions act as the consumer for our `pgmq` messaging queue. They pull batches of notifications, process them, and update the database via the Supabase Service Role key.
