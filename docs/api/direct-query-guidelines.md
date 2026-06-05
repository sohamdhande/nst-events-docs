# Direct Query Guidelines

## What IS Allowed
Direct Supabase PostgREST queries should be used for simple data retrieval and basic inserts where no complex business logic or capacity locking is required.

**Examples:**
* Event listings feed (`SELECT * FROM events WHERE status = 'PUBLISHED'`)
* Club listing and discovery
* Fetching the individual and club leaderboards
* Fetching a user's own profile and club memberships
* Fetching personal unread notifications
* Updating user profile display name

## What Must NEVER Use Direct Queries
Operations that require atomic state changes, capacity validation, deep role resolution, or interacting with external services.

**Examples of FORBIDDEN Direct Queries:**
* **`INSERT INTO event_registrations`**: Bypasses capacity limits. MUST use RPC.
* **`INSERT INTO attendance_records`**: Bypasses QR and Geofence checks. MUST use RPC/Edge.
* **`UPDATE events SET status = 'PUBLISHED'`**: Bypasses the approval workflow logic. MUST use RPC.
* **`DELETE FROM club_memberships`**: Can orphan teams or events. MUST use RPC.\n