# RPC Catalog

> **Execution Context**: All RPCs listed here are called from Express route handlers via Prisma `$queryRaw` / `$executeRaw`. Every call **must** be wrapped in `withUserContext(req.user.id, ...)` (see `docs/security/01-rls-architecture.md`) so that `current_user_id()` resolves to the authenticated user inside the transaction. Calling an RPC outside `withUserContext` results in `current_user_id()` returning `NULL`, which will silently bypass all internal role checks in `SECURITY DEFINER` functions.

---

## `register_event(p_event_id)`

* **Purpose**: Registers a user for an individual event.
* **Inputs**: `p_event_id`
* **Outputs**: `registration_id`, `status` (`REGISTERED` or `WAITLISTED`)
* **Permissions**: Any authenticated student.
* **Transaction Requirements**: Locks event row with `SELECT * FROM events WHERE id = p_event_id FOR UPDATE`. Checks `max_capacity` against `registration_count`. If capacity available (or unlimited): increments `registration_count` and inserts with `REGISTERED`. If full: inserts with `WAITLISTED` without incrementing count. Entire operation is atomic.
* **Tables Touched**: `events` (lock + update `registration_count`), `event_registrations` (insert).

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
* **Execution**: `SECURITY DEFINER` — runs as DB owner, bypasses RLS on `event_registrations`.
* **⚠️ Precondition**: Must be called inside `withUserContext(userId, ...)`. `current_user_id()` must resolve to the actor performing the cancellation (or the system actor if called from a trigger). If called outside `withUserContext`, the audit log will record a `NULL` actor_id, which is a bug.
* **Caller**: Called internally by `delete_registration` flow after a cancellation. Not called directly by clients.

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
* **Behavior**: Updates dispute status. If `APPROVED`: creates or updates `attendance_records` row for the student with `status = 'EXCUSED'`. Updates `reviewed_at` and `reviewed_by`. Appends to `audit_logs`.
* **Execution**: `SECURITY DEFINER` — bypasses RLS on `attendance_records` because Club Admin cannot write to that table under normal RLS.
* **⚠️ Precondition**: **Must be called inside `withUserContext(req.user.id, ...)`**. The function reads `current_user_id()` internally to resolve the caller's role before executing. If called outside `withUserContext`, the role check reads `NULL` and the security guard fails silently. Express middleware must verify the caller role before invoking this RPC, and `withUserContext` must wrap the call.

## `initiate_leadership_transfer`
* **Caller**: Club Admin
* **Input**: `club_id`, `successor_id`
* **Behavior**: Creates `leadership_handover_requests` row with status `PENDING`. Notifies Faculty Mentor assigned to club.
* **Security**: Caller must be current Club Admin of the specified club.

## `approve_leadership_transfer`
* **Caller**: Faculty Mentor, Faculty Admin (fallback), Platform Admin
* **Input**: `handover_request_id`
* **Behavior**: Sets status `APPROVED`. Transfers Club Admin role to `successor_id`. Demotes initiator to Core Member. Appends to `audit_logs`.
* **Security**: Caller must be Faculty Mentor of the club in the request. If no Faculty Mentor is assigned to the club, any user with the `FACULTY_ADMIN` global role may approve as a fallback. Platform Admin may always approve.

## `reject_leadership_transfer`
* **Caller**: Faculty Mentor, Faculty Admin (fallback), Platform Admin
* **Input**: `handover_request_id`, `review_notes`
* **Behavior**: Sets status `REJECTED`. Initiator retains Club Admin role.
* **Security**: Caller must be Faculty Mentor of the club in the request. Faculty Admin fallback applies when no Faculty Mentor is assigned.

## `force_transfer_leadership`
* **Caller**: Platform Admin only
* **Input**: `club_id`, `new_admin_id`
* **Behavior**: Bypasses Faculty Mentor approval. Immediately transfers role. Appends to `audit_logs` with FORCED flag.
* **Execution**: `SECURITY DEFINER` — bypasses RLS on `club_memberships`.
* **⚠️ Precondition**: **Must be called inside `withUserContext(req.user.id, ...)`**. The function reads `current_user_id()` internally to validate the caller is PLATFORM_ADMIN. Express middleware (`requireRole('PLATFORM_ADMIN')`) must gate the route. `withUserContext` must wrap the Prisma `$queryRaw` call so that `current_user_id()` resolves before the internal role check executes.

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
