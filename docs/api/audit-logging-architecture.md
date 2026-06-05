# Audit Logging Architecture

## Implementation
Audit logging is entirely handled within PostgreSQL to guarantee immutability.

## Database Triggers & Event Triggers
We attach `AFTER INSERT OR UPDATE OR DELETE` triggers to high-value tables (`events`, `club_memberships`, `leaderboard_scores`).

## Audit Log Structure
```sql
CREATE TABLE audit_logs (
    id BIGSERIAL PRIMARY KEY,
    actor_id UUID REFERENCES auth.users(id),
    action TEXT,
    entity_type TEXT,
    entity_id UUID,
    previous_state JSONB,
    new_state JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

## Tracked Actions
* Event state transitions.
* Club creation/deletion.
* Role assignment (Promoting to Admin/Faculty).
* Manual attendance overrides.

## Retention
Logs are kept indefinitely in V1.

## Investigation Workflow
Platform Admins use a dedicated dashboard to search the audit logs by `entity_id` to replay exactly who modified a record and when.\n