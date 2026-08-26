# Acceptance Criteria

AC-EVENT-DETAIL-01: Event Detail renders `title`, `start_time`, `location` from `GET /v1/events/:id`. It MUST NOT use `event.name`.
AC-REG-01: Participant status is strictly derived from `GET /v1/events/:id/my-registration`.
AC-REG-02: The frontend NEVER uses `GET /v1/events/:id/registrations` to determine the current user's registration status.
AC-TEAM-01: A student may belong to at most one team per event.
AC-AUTH-01: Web access token is never persisted to localStorage.
AC-CLUB-01: Club screen renders only fields defined in DATA_CONTRACT.md (\`id, name, description, banner_url, status, created_at\`).
AC-ATTENDANCE-01: Frontend never consumes or stores \`qr_secret\`. It is strictly stripped from frontend logic.
AC-LEADERBOARD-01: Leaderboard ordering follows backend response ordering (descending by \`total_points\`) and is not recomputed client-side.
AC-SSE-01: Frontend handles only documented SSE event types (registration_count, waitlist_update, heartbeat).
AC-WEB-14-PAGINATION-01: First page renders.
AC-WEB-14-PAGINATION-02: Load More appears when `next_cursor` exists.
AC-WEB-14-PAGINATION-03: Clicking Load More requests the current `next_cursor`.
AC-WEB-14-PAGINATION-04: Returned users append to the existing list.
AC-WEB-14-PAGINATION-05: Existing users are not duplicated.
AC-WEB-14-PAGINATION-06: Load More is disabled while the request is pending.
AC-WEB-14-PAGINATION-07: Load More disappears when `next_cursor` is null.
AC-WEB-14-PAGINATION-08: Next-page errors do not erase existing users.
AC-WEB-14-PAGINATION-09: Next-page errors are visibly reported according to the state contract.
AC-REG-03: TEAM event rejects individual registration.
AC-REG-04: TEAM event allows team creation when capacity is available.
AC-REG-05: TEAM event at capacity rejects team creation.
AC-REG-06: No WAITLISTED team state is created in V1.
AC-ATTENDANCE-02: SINGLE event allows one attendance session.
AC-ATTENDANCE-03: SINGLE event rejects a second attendance session.
AC-ATTENDANCE-04: MULTI_SESSION event allows multiple attendance sessions.
AC-LOCK-01: Change browser clock backward cannot extend unlock window.
AC-LOCK-02: Change browser clock forward cannot prematurely trigger permanent lock.
AC-LOCK-03: Direct POST `/v1/events/:id/unlock` after deadline is rejected by backend.
AC-LOCK-04: Direct POST `/v1/events/:id/lock` after deadline is rejected/ignored in favor of lazy permanent lock policy.
AC-LOCK-05: Send forged timestamps in payload ignored for authorization; server time must be used.
AC-LOCK-06: Two concurrent lock/unlock requests around deadline are resolved atomically by database/server time.
AC-LOCK-07: A locked event permanently blocks new registrations (lazy enforcement regardless of physical `is_locked` toggle).
AC-ACADEMIC-01: Each student has one current AcademicBatch.
AC-ACADEMIC-02: Student cannot self-edit academic batch.
AC-ACADEMIC-03: Admin batch changes are authoritative.
AC-ACADEMIC-04: Email inference does not overwrite an admin assignment.
AC-AUDIENCE-01: Events may target ALL_STUDENTS or one/more SPECIFIC_BATCHES.
AC-AUDIENCE-02: Audience is distinct from Primary Club.
AC-AUDIENCE-03: Audience filtering occurs server-side.
AC-AUDIENCE-04: Ineligible users cannot register directly through the API.
AC-AUDIENCE-05: TEAM event membership enforces audience eligibility for every member.
AC-AUDIENCE-06: Team invite/join paths cannot bypass audience restrictions.
AC-AUDIENCE-07: Historical records survive batch changes.
AC-AUDIENCE-08: Frontend filtering is never the security boundary.
AC-TEAM-02: Creating a team places it in FORMING until minimum size is reached.
AC-TEAM-03: A team only becomes REGISTERED when minimum size is reached and sufficient capacity exists.
AC-TEAM-04: A complete team that cannot fit in available capacity becomes WAITLISTED.
AC-TEAM-05: Team waitlist ordering is FIFO.
AC-TEAM-06: Waitlist promotion never exceeds event max_capacity.
AC-TEAM-07: Every team member must satisfy event audience eligibility.
AC-TEAM-08: Maximum team size cannot be exceeded.
AC-TEAM-09: Duplicate team invitations cannot create duplicate memberships.
AC-TEAM-10: Invitation acceptance revalidates audience, capacity, team size, event state, and membership.
AC-TEAM-11: A leader cannot leave without transferring leadership.
AC-TEAM-12: Registered teams below minimum receive a 24-hour grace period.
AC-TEAM-13: Teams remaining below minimum after grace are cancelled.
AC-TEAM-14: Locked events reject all team mutations.
AC-TEAM-15: Historical team records remain readable after lock/cancellation.
AC-TEAM-16: Administrative team overrides are authorized and audited.
AC-TEAM-17: Student cannot manipulate team membership for another user.
AC-TEAM-18: Concurrent team operations cannot exceed event capacity.

### Club Branding / Banners

#### `AC-CLUB-BANNER-URL-01`
**Description**: Banner is optional during Club creation.
**Verification**: Verify that the club creation form allows submission with an empty `banner_url` field.

#### `AC-CLUB-BANNER-URL-02`
**Description**: Banner can be edited after Club creation.
**Verification**: Verify that the edit club form allows updating the `banner_url` and it reflects in the club details.

#### `AC-CLUB-BANNER-URL-03`
**Description**: Banner can be removed by setting banner_url to null.
**Verification**: Verify that clearing the `banner_url` input sends `banner_url: null` in the API payload and removes the banner.

#### `AC-CLUB-BANNER-URL-04`
**Description**: Only HTTP/HTTPS URLs are accepted.
**Verification**: Provide `http://example.com/image.jpg` or `https://example.com/image.jpg` and confirm the backend accepts the payload.

#### `AC-CLUB-BANNER-URL-05`
**Description**: Unsafe URL schemes are rejected.
**Verification**: Provide `javascript:alert(1)` or `data:image/png;base64,...` and confirm the backend responds with a 422 schema validation error.

#### `AC-CLUB-BANNER-URL-06`
**Description**: Recommended banner ratio is 4:1.
**Verification**: Verify the UI displays the recommended 4:1 aspect ratio helper text.

#### `AC-CLUB-BANNER-URL-07`
**Description**: Recommended dimensions are 1600 × 400.
**Verification**: Verify the UI displays the recommended 1600 × 400 px dimension helper text.

#### `AC-CLUB-BANNER-URL-08`
**Description**: The dashboard displays a live preview.
**Verification**: Enter a valid image URL in the `banner_url` field and confirm a live preview is rendered dynamically.

#### `AC-CLUB-BANNER-URL-09`
**Description**: Broken URLs display the No Banner fallback.
**Verification**: Enter a broken URL and confirm the preview falls back to the semantic "No Banner" token placeholder.

#### `AC-CLUB-BANNER-URL-10`
**Description**: No platform storage credentials are exposed because no platform storage exists.
**Verification**: Confirm no requests are made to `/upload/presign` and no storage SDKs are bundled.

#### `AC-CLUB-BANNER-URL-11`
**Description**: No file upload is presented in V1.
**Verification**: Confirm the UI only provides a standard URL string input and not a file dropper or cropper modal.

### AC-NOTIFICATION

* **AC-NOTIFICATION-01**: Authenticated user receives only own notification data.
* **AC-NOTIFICATION-02**: Unread count is authoritative and sourced entirely from the backend.
* **AC-NOTIFICATION-03**: Mark read updates the authoritative unread count.
* **AC-NOTIFICATION-04**: Mark all read updates the authoritative unread count.
* **AC-NOTIFICATION-05**: New notification emits realtime event accurately and solely after successful database commit.
* **AC-NOTIFICATION-06**: Strict cross-user isolation prevents subscribing to another user's stream.
* **AC-NOTIFICATION-07**: Client must use REST reconciliation to restore state after an SSE reconnect.
* **AC-NOTIFICATION-08**: Duplicate realtime delivery does not create duplicate notification records in the UI.
* **AC-NOTIFICATION-09**: Notifications persist securely even when SSE infrastructure is completely unavailable.
* **AC-NOTIFICATION-10**: Realtime mark-read propagates properly to allow multi-session read-state synchronization.

### AC-TEAM-ATTENTION

* **AC-TEAM-ATTENTION-01**: Event aggregate `below_minimum_team_count` MUST exactly match the number of `REGISTERED` teams falling strictly below `minimum_team_size`.
* **AC-TEAM-ATTENTION-02**: Event aggregate `below_minimum_team_count` MUST NOT create N+1 query execution.
* **AC-TEAM-ATTENTION-03**: `below_minimum_team_count` MUST NOT invent a new team status or persist new derived data.
* **AC-TEAM-ATTENTION-04**: The Organizer Dashboard MUST display a unified "Teams Requiring Attention" view using the aggregate signal.
* **AC-TEAM-ATTENTION-05**: Waitlisted, Cancelled, and Forming teams MUST NOT be counted in the aggregate signal.

### AC-STUDENT-DIRECTORY
* **AC-STUDENT-DIRECTORY-01**: Student Directory membership strictly dictates login eligibility.
* **AC-STUDENT-DIRECTORY-02**: CSV imports correctly resolve academic Program and Batch using institutional email.
* **AC-STUDENT-DIRECTORY-03**: CSV imports do not overwrite existing ADMIN_OVERRIDE academic profiles.
* **AC-STUDENT-DIRECTORY-04**: Duplicate emails in CSV are handled gracefully with detailed error reporting.
* **AC-STUDENT-DIRECTORY-05**: Invalid domains or formats in CSV are explicitly reported and rejected.
* **AC-STUDENT-DIRECTORY-06**: Unauthorized login attempts gracefully present an "Access Restricted" message.

### AC-GLOBAL-ROLE
* **AC-GLOBAL-ROLE-01**: Only PLATFORM_ADMIN can mutate global roles.
* **AC-GLOBAL-ROLE-02**: A user's Global Role mutation does not automatically alter Club roles or memberships.
* **AC-GLOBAL-ROLE-03**: Backend prevents removing the last remaining PLATFORM_ADMIN.
* **AC-GLOBAL-ROLE-04**: An admin cannot demote themselves.
* **AC-GLOBAL-ROLE-05**: Audit logs are generated for all global role mutations.
