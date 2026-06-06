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

## `approve_event`
* **Caller**: Faculty Mentor, Faculty Admin, Platform Admin
* **Input**: `event_id`
* **Behavior**: Transitions event status from `PENDING_APPROVAL` to `PUBLISHED`. Triggers push notifications to eligible students.
* **Security**: Faculty Mentor must be mentor of a club attached to the event.

## `mark_attendance(p_session_id, p_totp, p_lat, p_lng)`
* **Purpose**: Validates physical presence and issues points.
* **Inputs**: Session ID, QR TOTP, user GPS coords.
* **Outputs**: `attendance_id`, `points_awarded`
* **Permissions**: Authenticated student.
* **Tables Touched**: `attendance_records`, `audit_logs`.
* **Behavior**:
  1. Geofence validation.
  2. TOTP validation.
  3. **Device Collision Check**:
     * Query `attendance_records` where: `session_id` = input `session_id` AND `audit_metadata->>'device_id'` = input `device_id` AND `user_id` != input `user_id`.
     * If result count > 0:
       * Proceed with attendance write (do not reject).
       * Set `audit_metadata.flagged = true`.
       * Set `audit_metadata.flag_reason = "device_collision"`.
       * Insert into `audit_logs`: action: `ATTENDANCE_DEVICE_COLLISION`, `session_id`: input `session_id`, `flagged_user_id`: input `user_id`, `colliding_user_id`: the `user_id` found in collision query, `device_id`: input `device_id`.

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

## `submit_attendance_dispute`
* **Caller**: Student (authenticated)
* **Input**: `session_id`, `reason`, `evidence_urls[]`
* **Behavior**: Creates `attendance_disputes` row with status `PENDING`. Sets `dispute_window_expires_at` = event `end_time` + 24 hours. Fails if current time > `dispute_window_expires_at`.
* **Security**: RLS — student can only submit for their own attendance.

## `resolve_attendance_dispute`
* **Caller**: Club Admin, Faculty Mentor, Faculty Admin, Platform Admin
* **Input**: `dispute_id`, `resolution` (APPROVED | REJECTED), `review_notes`
* **Behavior**: Updates dispute status. If `APPROVED`: creates or updates `attendance_records` row for the student. Updates `reviewed_at` and `reviewed_by`. Appends to `audit_logs`.
* **Security**: SECURITY DEFINER — this RPC bypasses RLS on `attendance_records` because Club Admin cannot write to that table under normal RLS. Must validate caller role before executing.

## `initiate_leadership_transfer`
* **Caller**: Club Admin
* **Input**: `club_id`, `successor_id`
* **Behavior**: Creates `leadership_handover_requests` row with status `PENDING`. Notifies Faculty Mentor assigned to club.
* **Security**: Caller must be current Club Admin of the specified club.

## `approve_leadership_transfer`
* **Caller**: Faculty Mentor
* **Input**: `handover_request_id`
* **Behavior**: Sets status `APPROVED`. Transfers Club Admin role to `successor_id`. Demotes initiator to Core Member. Appends to `audit_logs`.
* **Security**: Caller must be Faculty Mentor of the club in the request.

## `reject_leadership_transfer`
* **Caller**: Faculty Mentor
* **Input**: `handover_request_id`, `review_notes`
* **Behavior**: Sets status `REJECTED`. Initiator retains Club Admin role.
* **Security**: Caller must be Faculty Mentor of the club in the request.

## `force_transfer_leadership`
* **Caller**: Platform Admin only
* **Input**: `club_id`, `new_admin_id`
* **Behavior**: Bypasses Faculty Mentor approval. Immediately transfers role. Appends to `audit_logs` with FORCED flag.
* **Security**: Platform Admin only. SECURITY DEFINER.

## `submit_event_for_approval`
* **Caller**: Club Admin, Core Member
* **Input**: `event_id`
* **Behavior**: Transitions event status from `DRAFT` to `PENDING_APPROVAL`. Notifies assigned Faculty Mentor.
* **Security**: Caller must be member of a club attached to the event.

## `reject_event`
* **Caller**: Faculty Mentor, Faculty Admin, Platform Admin
* **Input**: `event_id`, `rejection_reason`
* **Behavior**: Transitions event status from `PENDING_APPROVAL` back to `DRAFT`. Notifies Club Admin with `rejection_reason`.
* **Security**: Faculty Mentor must be mentor of a club attached to the event.

## `lock_event`
* **Caller**: Club Admin, Faculty Mentor, Faculty Admin, Platform Admin
* **Input**: `event_id`
* **Behavior**: Sets `events.is_locked = true`. Halts all new attendance scans and registrations for the event. Only affects active sessions — does not delete existing records.
* **Security**: Caller must have appropriate role permissions for the event.

## `unlock_event`
* **Caller**: Club Admin, Faculty Mentor, Faculty Admin, Platform Admin
* **Input**: `event_id`
* **Behavior**: Sets `events.is_locked = false`. Resumes normal event operations.
* **Security**: Caller must have appropriate role permissions for the event.
