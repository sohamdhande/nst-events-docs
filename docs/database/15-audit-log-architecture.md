# Audit Log Architecture

## Table Structure
**`audit_logs`**
* `id BIGSERIAL PRIMARY KEY` (Using BIGSERIAL instead of UUID for strict chronological ordering and performance on massive inserts).
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
