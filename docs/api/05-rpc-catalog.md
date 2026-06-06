# RPC Catalog

## `register_event(p_event_id)`
* **Purpose**: Registers a user for an individual event.
* **Inputs**: `p_event_id`
* **Outputs**: `registration_id`, `status`
* **Permissions**: Any authenticated student.
* **Transaction Requirements**: Locks event row, checks capacity, inserts registration.
* **Tables Touched**: `events`, `event_registrations`.

## `create_team(p_event_id, p_team_name)`
* **Purpose**: Creates a team for a hackathon and sets the caller as leader.
* **Inputs**: `p_event_id`, `p_team_name`
* **Outputs**: `team_id`
* **Permissions**: Authenticated student.
* **Tables Touched**: `events`, `teams`, `team_members`.

## `approve_event(p_event_id)`
* **Purpose**: Faculty approval of an event.
* **Inputs**: `p_event_id`
* **Outputs**: Boolean.
* **Permissions**: `FACULTY_MENTOR` tied to the event's club, or `FACULTY_ADMIN`.
* **Tables Touched**: `events`, `audit_logs`, `notifications` (queued).

## `mark_attendance(p_session_id, p_totp, p_lat, p_lng)`
* **Purpose**: Validates physical presence and issues points.
* **Inputs**: Session ID, QR TOTP, user GPS coords.
* **Outputs**: `attendance_id`, `points_awarded`
* **Permissions**: Authenticated student.
* **Tables Touched**: `attendance_records`, `leaderboard_scores`, `audit_logs`.

## `process_waitlist(p_event_id)`
* **Purpose**: Moves waitlisted users to registered if capacity frees up.
* **Permissions**: `SECURITY DEFINER` (Called internally via triggers).

### `assign_participation_role`
* **Purpose**: Upgrades a user's role from ATTENDEE to VOLUNTEER, ORGANIZER, etc.
* **Security**: Only callable by Club Admin, Faculty Mentor, Faculty Admin, Platform Admin.

### `submit_competition_result`
* **Purpose**: Records a verified placement (e.g., WINNER) in the `event_results` table.
* **Security**: Only callable by Club Admin, Faculty Mentor, Faculty Admin, Platform Admin. Students may NEVER submit results.

### `adjust_points_disciplinary`
* **Purpose**: Corrective manual point adjustment.
* **Security**: Only callable by Platform Admin. Automatically creates an `ADJUST_POINTS` audit log.
