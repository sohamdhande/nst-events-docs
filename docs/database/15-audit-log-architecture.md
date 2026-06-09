# Audit Log Architecture

## Table Structure
**`audit_logs`**
* `id BIGSERIAL PRIMARY KEY` (Using BIGSERIAL instead of UUID for strict chronological ordering and performance on massive inserts).
* `actor_id UUID` — populated by the audit trigger using `current_user_id()`, which reads the `app.user_id` session variable set by the `withUserContext` transaction wrapper. All mutating API queries must run inside this wrapper. A `NULL` actor indicates a bug.
* `action`: (e.g., "DELETE_CLUB", "OVERRIDE_ATTENDANCE")
* `previous_state` / `new_state`: JSONB snapshots.

## Tracked Actions
* Deleting any club or event.
* Promoting a user to Faculty or Platform Admin.
* Manually overriding an attendance record.

## Retention Policy
Logs are kept indefinitely in V1. A background cron job may archive logs older than 2 years into cold storage (S3) in the future.

## Examples
* Action: `OVERRIDE_ATTENDANCE`
* Previous: `{"status": "ABSENT"}`
* New: `{"status": "PRESENT"}`
* Actor: `faculty_id_123`

### Leaderboard & Merit Audit Actions
* `ASSIGN_PARTICIPATION_ROLE`: Changing a user's role.
* `REMOVE_PARTICIPATION_ROLE`: Downgrading a role.
* `SUBMIT_COMPETITION_RESULT`: Posting a competition win.
* `MODIFY_COMPETITION_RESULT`: Correcting a result.
* `ADJUST_POINTS`: Platform Admin override.
