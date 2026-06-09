# Audit Logging Architecture

## Implementation
Audit logging is entirely handled within PostgreSQL to guarantee immutability.

## Database Triggers & Event Triggers
We attach `AFTER INSERT OR UPDATE OR DELETE` triggers to high-value tables (`events`, `club_memberships`, `attendance_records`).

## Actor Identity Resolution — Critical Implementation Detail

Audit triggers must capture the identity of the API user who initiated the action, not the database connection user. Triggers extract `actor_id` using the shared `current_user_id()` helper function:

```sql
-- Inside the trigger function:
INSERT INTO audit_logs (actor_id, action, entity_type, entity_id, previous_state, new_state)
VALUES (current_user_id(), TG_OP, TG_TABLE_NAME, NEW.id, row_to_json(OLD), row_to_json(NEW));
```

This requires that **all mutating API queries** execute inside the `withUserContext(userId, ...)` transaction wrapper (see `docs/security/01-rls-architecture.md`). If a query runs outside the wrapper, `current_user_id()` returns `NULL` and the audit log will record a `NULL` actor — this is a bug and must be treated as a defect during development.

## Audit Log Structure
```sql
CREATE TABLE audit_logs (
    id BIGSERIAL PRIMARY KEY,
    actor_id UUID REFERENCES users(id),
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
Platform Admins use a dedicated dashboard to search the audit logs by `entity_id` to replay exactly who modified a record and when.

### Leaderboard & Merit Audit Actions
* `ASSIGN_PARTICIPATION_ROLE`: Changing a user's role.
* `REMOVE_PARTICIPATION_ROLE`: Downgrading a role.
* `SUBMIT_COMPETITION_RESULT`: Posting a competition win.
* `MODIFY_COMPETITION_RESULT`: Correcting a result.
* `ADJUST_POINTS`: Platform Admin override.
