# Registration Data Model

## Architecture
Handles both individuals and teams efficiently.

* **`event_registrations`**: Connects `user_id` to `event_id`.
* **`teams`**: Represents a grouped entity with a `leader_id`.
* **`team_members`**: Connects `user_id` to `team_id`.

## Individual vs Team Registration
If an event is a Hackathon, users create a `teams` record first, then they are inserted into `team_members`. The `event_registration` row contains a non-null `team_id`. For individual registrations, `team_id` is null.

## Capacity Management & Waitlists
The `events` table contains `max_capacity` (nullable INTEGER; `NULL` means unlimited) and `registration_count` (INTEGER, default 0). When a student registers:

1. The `register_event` RPC locks the event row: `SELECT * FROM events WHERE id = p_event_id FOR UPDATE`.
2. If `max_capacity IS NULL` OR `registration_count < max_capacity`: increments `registration_count` and inserts an `event_registrations` row with `registration_status = 'REGISTERED'`.
3. If `registration_count >= max_capacity`: inserts with `registration_status = 'WAITLISTED'` (does NOT increment `registration_count`).
4. The entire operation is atomic within a single transaction.

### `SELECT FOR UPDATE` Strategy
The `FOR UPDATE` row-level lock on the `events` row ensures that concurrent requests from 500+ students at exactly the same millisecond do not oversell a limited-capacity event. The lock serializes capacity checks without table-wide locking.
