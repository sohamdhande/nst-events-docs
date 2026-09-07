# RPC Catalog

> **Execution Context**: All RPCs listed here are called from Express route handlers via Prisma `$queryRaw` / `$executeRaw`. Every call **must** be wrapped in `withUserContext(req.user.id, ...)` (see `docs/security/01-rls-architecture.md`) so that `current_user_id()` resolves to the authenticated user inside the transaction. Calling an RPC outside `withUserContext` results in `current_user_id()` returning `NULL`, which will silently bypass all internal role checks in `SECURITY DEFINER` functions.

---

## Canonical Global Lock Ordering

> **Mandatory Rule**: All RPCs MUST acquire locks in the exact global order to prevent deadlocks:
> 1. Event row
> 2. Team row (if applicable)
> 3. Registration rows
> 4. Waitlist rows

---

## `register_event(p_event_id)`

* **Purpose**: Registers a user for an individual event.
* **Inputs**: `p_event_id`
* **Outputs**: `registration_id`, `status` (`REGISTERED` or `WAITLISTED`)
* **Permissions**: Any authenticated student. Event must be `PUBLISHED`. Registration is implicitly rejected if the event is permanently locked (lazy enforcement) or if `is_locked = true`.
* **Transaction Timeline**:
  1. `BEGIN`
  2. Execute Lock-Free Atomic Increment (`UPDATE events SET registration_count = registration_count + 1 WHERE ... RETURNING`)
  3. Validate registration constraints (already registered?, is published?, MUST reject if `registration_type` is `TEAM`, MUST reject with `AUDIENCE_NOT_ELIGIBLE` if user fails `SPECIFIC_BATCHES` audience requirements).
  4. Allocate seat (if `registration_count < max_capacity`) OR waitlist
  5. Insert registration
  6. Update `registration_count` (if seat allocated)
  7. Emit `registration_count` realtime event (per `docs/database/21-realtime-listen-notify-contract.md`)
  8. `COMMIT`
* Note: Notification Producer is invoked by the API layer post-commit if applicable.

## `cancel_registration(p_event_id, p_user_id)`

* **Purpose**: Soft deletes a registration and processes waitlist promotion.
* **Inputs**: `p_event_id`, `p_user_id`
* **Outputs**: Array of `promoted_user_ids`
* **Permissions**: Any authenticated student (own registration) or admin.
* **Transaction Timeline**:
  1. `BEGIN`
  2. Lock event (`FOR UPDATE`)
  3. Soft-delete registration
  4. Invoke `process_waitlist` internal RPC
  5. If `process_waitlist` returns 0 promoted users, decrement `events.registration_count` by 1
  6. Emit `registration_count` realtime event (per `docs/database/21-realtime-listen-notify-contract.md`)
  7. `COMMIT`
* Note: API handler uses returned `promoted_user_ids` to invoke Notification Producer.

## `process_waitlist(p_event_id)` (Internal)
* **Purpose**: FIFO promotion of waitlisted members.
* **Transaction Timeline**:
  1. Select oldest active `WAITLISTED` registration `FOR UPDATE SKIP LOCKED`
  2. **CRITICAL**: Ignore any registration where `team_id` is NOT NULL and the referenced team is soft-deleted.
  3. Promote to `REGISTERED`
  4. Emit `waitlist_update` realtime event (per `docs/database/21-realtime-listen-notify-contract.md`)
  5. Return `promoted_user_ids`

### `assign_participation_role`
* **Purpose**: Upgrades a user's role from ATTENDEE to VOLUNTEER, ORGANIZER, etc.
* **Security**: Only callable by Club Admin, Faculty Mentor, Faculty Admin, Platform Admin.

### `create_team(p_event_id, p_team_name)`
* **Purpose**: Initializes a new team for a group-based event.
* **Transaction Timeline**:
  1. `BEGIN`
  2. Lock event (`FOR UPDATE`)
  3. Validate:
     - state = 'PUBLISHED'
     - registration_type = 'TEAM'
     - caller satisfies event audience eligibility (`AUDIENCE_NOT_ELIGIBLE` if rejected)
  4. Insert team (state: `FORMING`) and creator as team lead and accepted member.
  5. Note: DOES NOT consume confirmed capacity or increment `registration_count`.
  6. `COMMIT`

### `join_team(p_event_id, p_team_id)`
* **Purpose**: Adds user to an existing team.
* **Transaction Timeline**:
  1. `BEGIN`
  2. Lock event (`FOR UPDATE`), Lock team (`FOR UPDATE`)
  3. Validate:
     - state = 'PUBLISHED'
     - registration_type = 'TEAM'
     - active team members < events.metadata->>'maximum_team_size'
     - user is not already a member of a team for this event
     - user satisfies event audience eligibility
  4. Insert team membership.
  5. If team reaches `minimum_team_size` (events.metadata->>'minimum_team_size'):
     - Check event capacity. If available, transition team to `REGISTERED` and increment `events.registration_count` by team size.
     - If not available, transition team to `WAITLISTED` (0 capacity consumed).
  6. `COMMIT`

### `leave_team(p_event_id, p_team_id)`
* **Purpose**: Removes user from a team.
* **Transaction Timeline**:
  1. `BEGIN`
  2. Lock event (`FOR UPDATE`), Lock team (`FOR UPDATE`)
  3. Validate user is NOT the leader (leaders must transfer leadership before leaving).
  4. Soft-delete user's membership.
  5. If team was `REGISTERED`:
     - Decrement `events.registration_count` by 1.
     - If team drops below `minimum_team_size`, flag team with 24-hour grace period condition.
  6. Invoke `process_waitlist`.
  7. `COMMIT`

### `transfer_leadership(p_event_id, p_team_id, p_new_leader_id)`
* **Purpose**: Reassigns team leadership to another accepted member.
* **Security**: Only callable by current leader or administrators.
* **Transaction Timeline**:
  1. `BEGIN`
  2. Lock team (`FOR UPDATE`)
  3. Verify `p_new_leader_id` is an active member.
  4. Update `leader_id`.
  5. `COMMIT`

### `is_users_available_for_team(p_event_id, p_team_id, p_target_user_ids)`
* **Purpose**: Batch evaluation of team eligibility for an array of candidate users. Identifies if candidates are available to be invited to a team.
* **Outputs**: Array of `{ user_id, is_available }` rows.
* **Security**: Enforces that the caller is the leader of the active `p_team_id` or an existing member via `current_user_id()`.
* **Behavior**: Evaluates event state (published, locked, expired), team state, caller authorization, and confirms candidates are not already active in another team or holding pending invitations. Does NOT serve as the final capacity gate.

## High-Risk Attendance RPCs (Versioned)

> **RPC Versioning Rule (ADR-005/RPC-01)**: High-risk attendance functions (`mark_attendance`, `sync_offline_attendance`) must not be replaced in place to avoid breaking concurrent deployments and to ensure safe rollbacks. New versions are suffixed as `_vN` and the API layer is updated to call the specific version. Old functions remain in the database for rollback targets.

### `mark_attendance_v5(p_session_id, p_user_id, ...)` (Active Version)
* **Purpose**: Writes an attendance record after verifying TOTP HMAC, geofence bounds, device collision, and event locks.
* **Outputs**: Standard `attendance_records` row.
* **Security**: `SECURITY DEFINER` bound.
* **Previous Versions**: `mark_attendance` (unversioned legacy alias)

### `sync_offline_attendance_v9(p_session_id, p_batch_data)` (Active Version)
* **Purpose**: Batch processing of offline attendance scans synchronized by the organizer.
* **Outputs**: Array of inserted `attendance_records`.
* **Security**: `SECURITY DEFINER` bound. Enforces the caller is an active event organizer.
* **Previous Versions**: `sync_offline_attendance` (unversioned legacy alias)

### `submit_competition_result`
* **Purpose**: Records a verified placement (e.g., WINNER) in the `event_results` table.
* **Security**: Only callable by Club Admin, Faculty Mentor, Faculty Admin, Platform Admin. Students may NEVER submit results.

### `adjust_points_disciplinary`
* **Purpose**: Corrective manual point adjustment.
* **Security**: Only callable by Platform Admin. Automatically creates an `ADJUST_POINTS` audit log.

## `manual_mark_attendance`
* **Caller**: Platform Admin, Faculty Admin, Club Admin (primary club) (via `POST /events/:id/attendance/manual`)
* **Input**: `p_session_id`, `p_user_id`
* **Return Type**: `attendance_records` row
* **Responsibilities**: Manually verifies attendance for a user. Bypasses QR, TOTP, and geofence validation. Enforces registration, event published state, session validity, and leaderboard rules.
* **Security**: `SECURITY DEFINER`. Must verify caller is `PLATFORM_ADMIN`, `FACULTY_ADMIN`, or primary `CLUB_ADMIN` for the event.
* **Idempotency**: `ON CONFLICT (session_id, user_id) DO NOTHING`. If record exists, returns existing record without throwing error.
* **Leaderboard Behavior**: If a new record is inserted, automatically awards `+5` attendance points via `leaderboard_scores` table.
* **Audit Logging**: Inserts an audit log with action `ATTENDANCE_MANUAL_MARK` and populates `audit_metadata` with `{ "method": "MANUAL" }`. Sets `attendance_records.method` to `MANUAL`.

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
* **Behavior**: Sets `events.is_locked = true`. Halts all new attendance scans and registrations for the event. Only affects active sessions — does not delete existing records. Cannot be executed after the 24-hour lock grace period (lazy enforcement treats event as permanently locked).
* **Security**: Caller must have appropriate role permissions for the event. Evaluates authoritative server time.

## `unlock_event`
* **Caller**: Club Admin, Faculty Mentor, Faculty Admin, Platform Admin
* **Input**: `event_id`
* **Behavior**: Sets `events.is_locked = false`. Resumes normal event operations. Cannot be executed after the 24-hour lock grace period.
* **Security**: Caller must have appropriate role permissions for the event. Evaluates authoritative server time against the deadline.
