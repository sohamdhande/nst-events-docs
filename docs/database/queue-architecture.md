# Queue Architecture

## PGMQ Integration
We leverage the `pgmq` extension to keep our queue collocated with our relational data. This allows transactional enqueueing (e.g., generating an event and queueing its notification in the same transaction).

## Notification Processing
Background workers pull from the queue to dispatch push notifications to external services.

## Strategies
* **Retry Strategy**: Exponential backoff up to 3 attempts.
* **Dead Letter Strategy**: Failed messages are moved to a dead letter queue (DLQ) for manual inspection by Platform Admins.
* **Autovacuum Tuning**: Queue tables experience high churn. Aggressive autovacuum settings are applied to the `pgmq` schema to prevent bloat.\n