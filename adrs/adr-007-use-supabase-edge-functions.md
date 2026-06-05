# ADR-007: Use Supabase Edge Functions

## Status
Accepted

## Context
We have workflows that cannot run securely on the client, such as sending notifications and processing attendance validations.

## Decision
Use Supabase Edge Functions.

## Consequences
Positive: Needed for notifications, attendance validation, media processing, and asynchronous workflows. Low latency.
Negative: Functions must be written in Deno.

## Alternatives Considered
* **AWS Lambda**: Rejected to keep the stack consolidated in Supabase.

## Future Impact
Will handle all secure server-side integrations.
