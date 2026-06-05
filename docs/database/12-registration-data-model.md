# Registration Data Model

## Architecture
Handles both individuals and teams efficiently.

* **`event_registrations`**: Connects `user_id` to `event_id`.
* **`teams`**: Represents a grouped entity with a `leader_id`.
* **`team_members`**: Connects `user_id` to `team_id`.

## Individual vs Team Registration
If an event is a Hackathon, users create a `teams` record first, then they are inserted into `team_members`. The `event_registration` row contains a non-null `team_id`. For individual registrations, `team_id` is null.

## Capacity Management & Waitlists
When an event hits max capacity, `registration_status` defaults to `WAITLISTED`. 

### `SELECT FOR UPDATE` Strategy
To prevent overselling limited capacity events (e.g., only 50 seats), the registration RPC function utilizes `SELECT ... FOR UPDATE` row-level locks on the `events` table to safely increment the internal `registration_count` concurrently.
