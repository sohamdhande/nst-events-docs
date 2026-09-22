# API Routing Matrix

> [!WARNING]
> **For current API integration, `docs/api/02-api-routing-matrix.md` is authoritative.**
> Historical/roadmap documents must not be used to infer current endpoints.

All operations are handled by the **Express backend**. Clients never call the database directly. The "Execution Type" column describes what Express does internally.

| Operation | Express Internal Execution | Reason | Security Requirements |
|---|---|---|---|
| View Events | Prisma Query | Simple read; RLS filters rows by user context | Valid JWT + RBAC middleware |
| Register Event | PostgreSQL RPC (`register_event`) | Requires lock-free atomic capacity update | RBAC check + RPC atomic transaction |
| Join Team | PostgreSQL RPC (`join_team`) | Team size validation and atomic inserts | RBAC check + Validation |
| Leave Team | PostgreSQL RPC (`leave_team`) | Team leadership reassignment logic | RBAC check + Ownership Check |
| Approve Event | PostgreSQL RPC (`approve_event`) | State machine transition + audit logs | `FACULTY_MENTOR` role via RBAC |
| Reject Event | PostgreSQL RPC (`reject_event`) | State transition + notification trigger | `FACULTY_MENTOR` role via RBAC |
| Mark Attendance | Express Route Handler → PostgreSQL RPC | ADR-005 TOTP validation, geofence check, device collision | RBAC + Geofence + Valid Token |
| Generate QR | Express Route Handler | Cryptographic `v1` TOTP seed generation (per ADR-005) | `CLUB_ADMIN` / Core via RBAC |

| Send Notification | Express → PostgreSQL RPC → native queue | Defers high-latency push delivery to background worker | `CLUB_ADMIN` / Platform via RBAC |
| Promote Member | PostgreSQL RPC | Inserts into `club_memberships` + audit | `CLUB_ADMIN` via RBAC |
| Assign Club Admin | PostgreSQL RPC | High-privilege role escalation | `PLATFORM_ADMIN` via RBAC |
| Create Club | PostgreSQL RPC | Complex setup (memberships, roles, audit) | `PLATFORM_ADMIN` via RBAC |
| Archive Event | PostgreSQL RPC | State machine transition | Club Admin, Faculty Mentor, Faculty Admin, Platform Admin via RBAC |


## Audience & Visibility Rules
* **Feed Visibility (`GET /v1/events`)**: Audience filtering MUST occur server-side. The frontend/mobile client is NOT the security boundary. For `PUBLIC` + `SPECIFIC_BATCHES` events, they are visible only to users in those batches. Unassigned users only see `ALL_STUDENTS` events.
* **Direct Access (`GET /v1/events/:id`)**: An unauthorized student trying to access an audience-restricted event directly will receive a `404 Not Found` (hidden semantics).
* **Registration & Teams**: `AUDIENCE_NOT_ELIGIBLE` error is returned when an ineligible user tries to register or join a team for a targeted event.
* **Team Registration Constraints**: `DELETE /v1/events/:id/register` is explicitly invalid for TEAM events and returns a `400 Bad Request`. Team cancellation must be handled via `POST /v1/teams/:id/cancel` by the team leader.
* **Admin Bypass**: Platform Admin, Faculty Admin, and authorized club operators bypass student audience restrictions.

## Current API (Phase 21J)

| Method | Mount Prefix | Router | Local Path | Final Path | Auth | Authorization | Service | Response |
|--------|--------------|--------|------------|------------|------|---------------|---------|----------|
| GET | `/v1/admin` | `adminQueueRouter` | `/queue/monitoring` | `/v1/admin/queue/monitoring` | Required | `PLATFORM_ADMIN` | `adminService` | 200 |
| GET | `/v1/admin` | `adminQueueRouter` | `/queue/jobs` | `/v1/admin/queue/jobs` | Required | `PLATFORM_ADMIN` | `adminService` | 200 |
| GET | `/v1/admin` | `adminQueueRouter` | `/queue/jobs/:id` | `/v1/admin/queue/jobs/:id` | Required | `PLATFORM_ADMIN` | `adminService` | 200 |
| POST | `/v1/admin` | `adminQueueRouter` | `/queue/jobs/:id/retry` | `/v1/admin/queue/jobs/:id/retry` | Required | `PLATFORM_ADMIN` | `adminService` | 200 |
| GET | `/v1/admin` | `adminQueueRouter` | `/queue/dead-letters` | `/v1/admin/queue/dead-letters` | Required | `PLATFORM_ADMIN` | `adminService` | 200 |
| POST | `/v1/admin` | `adminQueueRouter` | `/queue/dead-letters/:id/replay` | `/v1/admin/queue/dead-letters/:id/replay` | Required | `PLATFORM_ADMIN` | `adminService` | 200 |
| POST | `/v1/admin/users` | `adminUsersRouter` | `/:userId/revoke-sessions` | `/v1/admin/users/:userId/revoke-sessions` | Required | `PLATFORM_ADMIN` | Direct Prisma | 200 |
| GET | `/v1` | `academicBatchesRouter` | `/academic-batches` | `/v1/academic-batches` | Required | `PLATFORM_ADMIN, FACULTY_ADMIN, CLUB_ADMIN, CORE_MEMBER` | `academicBatchesService.getAcademicBatches` | 200 |
| GET | `/v1` | `academicProgramsRouter` | `/academic-programs` | `/v1/academic-programs` | Required | `PLATFORM_ADMIN, FACULTY_ADMIN, Active Club Organizer (CLUB_ADMIN/CORE_MEMBER)` | `academicProgramsService.getAcademicPrograms` | 200 |
| POST | `/v1` | `attendanceRouter` | `/attendance/generate-qr` | `/v1/attendance/generate-qr` | Required | `CLUB_ADMIN, CORE_MEMBER` | `attendanceService.generateQr` | 200 |
| POST | `/v1` | `attendanceRouter` | `/attendance/mark` | `/v1/attendance/mark` | Required | None | `attendanceService.markAttendance` | 200/201 |
| POST | `/v1` | `attendanceRouter` | `/attendance/sync-offline` | `/v1/attendance/sync-offline` | Required | None | `attendanceService.syncOffline` | 200 |
| GET | `/v1` | `attendanceRouter` | `/events/:id/attendance` | `/v1/events/:id/attendance` | Required | `CLUB_ADMIN, CORE_MEMBER, FACULTY_MENTOR` | `attendanceService.getEventAttendance` | 200 |
| GET | `/v1` | `attendanceRouter` | `/users/me/attendance` | `/v1/users/me/attendance` | Required | None | `attendanceService.getMyAttendance` | 200 |
| POST | `/v1` | `attendanceRouter` | `/events/:id/attendance/manual` | `/v1/events/:id/attendance/manual` | Required | `PLATFORM_ADMIN, FACULTY_ADMIN, CLUB_ADMIN (primary club)` | `attendanceService.manualMarkAttendance` | 200/201 |
| POST | `/v1` | `attendanceRouter` | `/attendance/disputes` | `/v1/attendance/disputes` | Required | None | `attendanceService.submitAttendanceDispute` | 201 |
| GET | `/v1` | `attendanceRouter` | `/attendance/disputes` | `/v1/attendance/disputes` | Required | None | `attendanceService.getAttendanceDisputes` | 200 |
| PATCH | `/v1` | `attendanceRouter` | `/attendance/disputes/:id` | `/v1/attendance/disputes/:id` | Required | None | `attendanceService.resolveAttendanceDispute` | 200 |
| GET | `/auth` | `authRouter` | `/google` | `/auth/google` | None | None | `authService.loginWithGoogle` | 200 |
| GET | `/auth` | `authRouter` | `/google/callback` | `/auth/google/callback` | None | None | `authService.loginWithGoogle` | 200 |
| POST | `/auth` | `authRouter` | `/refresh` | `/auth/refresh` | None | None | `authService.refreshTokens` | 200 |
| POST | `/auth` | `authRouter` | `/logout` | `/auth/logout` | Required | None | `authService.logout` | 204 |
| GET | `/clubs` | `clubsRouter` | `/search` | `/clubs/search` | Required | None | `clubsService.searchClubs` | 200 |
| GET | `/clubs` | `clubsRouter` | `/` | `/clubs` | Required | None | `clubsService.getClubs` | 200 |
| GET | `/clubs` | `clubsRouter` | `/:id` | `/clubs/:id` | Required | None | `clubsService.getClub` | 200 |
| POST | `/clubs` | `clubsRouter` | `/` | `/clubs` | Required | `PLATFORM_ADMIN` | `clubsService.createClub` | 201 |
| PATCH | `/clubs` | `clubsRouter` | `/:id` | `/clubs/:id` | Required | `CLUB_ADMIN, PLATFORM_ADMIN` | `clubsService.updateClub` | 200 |
| PATCH | `/clubs` | `clubsRouter` | `/:id/status` | `/clubs/:id/status` | Required | `PLATFORM_ADMIN` | `clubsService.updateClubStatus` | 200 |
| POST | `/clubs` | `clubsRouter` | `/:id/members` | `/clubs/:id/members` | Required | `CLUB_ADMIN, FACULTY_MENTOR` | `clubsService.addMember` | 201 |
| PATCH | `/clubs` | `clubsRouter` | `/:id/members/:userId` | `/clubs/:id/members/:userId` | Required | `CLUB_ADMIN, FACULTY_MENTOR` | `clubsService.updateMemberRole` | 200 |
| DELETE | `/clubs` | `clubsRouter` | `/:id/members/:userId` | `/clubs/:id/members/:userId` | Required | `CLUB_ADMIN, FACULTY_MENTOR` | `clubsService.removeMember` | 204 |
| GET | `/v1/events` | `eventsRouter` | `/` | `/v1/events` | Required | None | `eventsService.listEvents` | 200 |
| GET | `/v1/events` | `eventsRouter` | `/:id` | `/v1/events/:id` | Required | None | `eventsService.getEventById` | 200 |
| POST | `/v1/events` | `eventsRouter` | `/` | `/v1/events` | Required | None | `eventsService.createEvent` | 201 |
| PATCH | `/v1/events` | `eventsRouter` | `/:id` | `/v1/events/:id` | Required | `CLUB_ADMIN` | `eventsService.updateEvent` | 200 |
| DELETE | `/v1/events` | `eventsRouter` | `/:id` | `/v1/events/:id` | Required | `CLUB_ADMIN` | `eventsService.deleteEvent` | 204 |
| POST | `/v1/events` | `eventsRouter` | `/:id/submit-for-approval` | `/v1/events/:id/submit-for-approval` | Required | `CLUB_ADMIN, CORE_MEMBER` | `eventsService.submitForApproval` | 200 |
| POST | `/v1/events` | `eventsRouter` | `/:id/approve` | `/v1/events/:id/approve` | Required | `FACULTY_MENTOR` | `eventsService.approveEvent` | 200 |
| POST | `/v1/events` | `eventsRouter` | `/:id/reject` | `/v1/events/:id/reject` | Required | `FACULTY_MENTOR` | `eventsService.rejectEvent` | 200 |
| POST | `/v1/events` | `eventsRouter` | `/:id/lock` | `/v1/events/:id/lock` | Required | `CLUB_ADMIN, FACULTY_MENTOR` | `eventsService.lockEvent` (Rejected if after final lock deadline) | 200 |
| POST | `/v1/events` | `eventsRouter` | `/:id/unlock` | `/v1/events/:id/unlock` | Required | `CLUB_ADMIN, FACULTY_MENTOR` | `eventsService.unlockEvent` (Rejected if after final lock deadline) | 200 |
| GET | `/v1/events` | `eventsRouter` | `/:id/sessions` | `/v1/events/:id/sessions` | Required | None | `eventsService.listSessions` | 200 |
| POST | `/v1/events` | `eventsRouter` | `/:id/sessions` | `/v1/events/:id/sessions` | Required | `CLUB_ADMIN, CORE_MEMBER` | `eventsService.createSession` (Must enforce SINGLE session cardinality if AttendanceType is SINGLE) | 201 |
| PATCH | `/v1/events` | `eventsRouter` | `/:id/sessions/:sessionId` | `/v1/events/:id/sessions/:sessionId` | Required | `CLUB_ADMIN, CORE_MEMBER` | `eventsService.updateSession` | 200 |
| GET | `/v1/leaderboard` | `leaderboardRouter` | `/students` | `/v1/leaderboard/students` | Required | None | `leaderboardService.getStudentLeaderboard` | 200 |
| GET | `/v1/leaderboard` | `leaderboardRouter` | `/clubs` | `/v1/leaderboard/clubs` | Required | None | `leaderboardService.getClubLeaderboard` | 200 |
| POST | `/v1/admin/leaderboard` | `adminLeaderboardRouter` | `/recalculate` | `/v1/admin/leaderboard/recalculate` | Required | `PLATFORM_ADMIN` | `leaderboardService.refreshLeaderboards` | 200 |
| GET | `/v1/notifications` | `notificationsRouter` | `/` | `/v1/notifications` | Required | None | `notificationsService.getNotifications` | 200 |
| GET | `/v1/notifications` | `notificationsRouter` | `/unread-count` | `/v1/notifications/unread-count` | Required | None | `notificationsService.getUnreadCount` | 200 |
| GET | `/v1/notifications` | `sseRouter` | `/live` | `/v1/notifications/live` | Required | None | `SSE Connection` | `text/event-stream` |
| PATCH | `/v1/notifications` | `notificationsRouter` | `/read-all` | `/v1/notifications/read-all` | Required | None | `notificationsService.markAllAsRead` | 204 |
| PATCH | `/v1/notifications` | `notificationsRouter` | `/:id/read` | `/v1/notifications/:id/read` | Required | None | `notificationsService.markAsRead` | 200 |
| GET | `/v1/notifications` | `notificationsRouter` | `/preferences` | `/v1/notifications/preferences` | Required | None | `notificationsService.getPreferences` | 200 |
| PATCH | `/v1/notifications` | `notificationsRouter` | `/preferences` | `/v1/notifications/preferences` | Required | None | `notificationsService.updatePreferences` | 200 |
| POST | `/v1` | `registrationsRouter` | `/events/:id/register` | `/v1/events/:id/register` | Required | None | `registrationsService.registerEvent` | 201 |
| DELETE | `/v1` | `registrationsRouter` | `/events/:id/register` | `/v1/events/:id/register` | Required | None | `registrationsService.cancelRegistration` | 204 |
| POST | `/v1` | `registrationsRouter` | `/events/:id/teams` | `/v1/events/:id/teams` | Required | None | `teamsService.createTeam` | 201 |
| GET | `/v1` | `registrationsRouter` | `/events/:id/registrations` | `/v1/events/:id/registrations` | Required | `CLUB_ADMIN, CORE_MEMBER` | `registrationsService.getEventRegistrations` | 200 |
| GET | `/v1` | `registrationsRouter` | `/users/me/registrations` | `/v1/users/me/registrations` | Required | None | `registrationsService.getMyRegistrations` | 200 |
| GET | `/v1` | `registrationsRouter` | `/events/:id/my-registration` | `/v1/events/:id/my-registration` | Required | None | `registrationsService.getMyRegistrationStatus` | 200 |
| GET | `/v1/events` | `sseRouter` | `/:id/live` | `/v1/events/:id/live` | Required | None | `SSE Connection` | `text/event-stream` |
| POST | `/v1/teams` | `teamsRouter` | `/:id/join` | `/v1/teams/:id/join` | Required | None | `teamsService.joinTeam` | 201 |
| DELETE | `/v1/teams` | `teamsRouter` | `/:id/leave` | `/v1/teams/:id/leave` | Required | None | `teamsService.leaveTeam` | 204 |
| POST | `/v1/teams` | `teamsRouter` | `/:id/invitations` | `/v1/teams/:id/invitations` | Required | None (must be LEADER) | `teamsService.inviteMember` | 201 |
| GET | `/v1/teams` | `teamsRouter` | `/:id` | `/v1/teams/:id` | Required | None | `teamsService.getTeam` | 200 |
| POST | `/v1/teams` | `teamsRouter` | `/:id/invitations/:invitationId/accept` | `/v1/teams/:id/invitations/:invitationId/accept` | Required | None | `teamsService.acceptInvitation` | 200 |
| POST | `/v1/teams` | `teamsRouter` | `/:id/invitations/:invitationId/decline` | `/v1/teams/:id/invitations/:invitationId/decline` | Required | None | `teamsService.declineInvitation` | 200 |
| DELETE | `/v1/teams` | `teamsRouter` | `/:id/invitations/:invitationId` | `/v1/teams/:id/invitations/:invitationId` | Required | None (must be LEADER) | `teamsService.cancelInvitation` | 204 |
| POST | `/v1/teams` | `teamsRouter` | `/:id/cancel` | `/v1/teams/:id/cancel` | Required | None (must be LEADER) | `teamsService.cancelTeam` | 200 |
| POST | `/v1/teams` | `teamsRouter` | `/:id/transfer-leadership` | `/v1/teams/:id/transfer-leadership` | Required | None (must be LEADER) | `teamsService.transferLeadership` | 200 |
| DELETE | `/v1/teams` | `teamsRouter` | `/:id/members/:userId` | `/v1/teams/:id/members/:userId` | Required | None (must be LEADER) | `teamsService.removeMember` | 204 |
| POST | `/v1/admin/teams` | `adminTeamsRouter` | `/:id/promote-waitlist` | `/v1/admin/teams/:id/promote-waitlist` | Required | `PLATFORM_ADMIN`, `FACULTY_ADMIN` | `adminTeamsService.manualWaitlistPromotion` | 200 |
| POST | `/v1/admin/teams` | `adminTeamsRouter` | `/:id/cancel` | `/v1/admin/teams/:id/cancel` | Required | `PLATFORM_ADMIN`, `FACULTY_ADMIN`, `CLUB_ADMIN` | `adminTeamsService.cancelTeam` | 200 |
| DELETE | `/v1/admin/teams` | `adminTeamsRouter` | `/:id/members/:userId` | `/v1/admin/teams/:id/members/:userId` | Required | `PLATFORM_ADMIN`, `FACULTY_ADMIN`, `CLUB_ADMIN` | `adminTeamsService.removeMember` | 204 |
| POST | `/v1/admin/teams` | `adminTeamsRouter` | `/:id/transfer-leadership` | `/v1/admin/teams/:id/transfer-leadership` | Required | `PLATFORM_ADMIN`, `FACULTY_ADMIN`, `CLUB_ADMIN` | `adminTeamsService.transferLeadership` | 200 |
| GET | `/v1/admin/teams` | `adminTeamsRouter` | `/:id/invitations` | `/v1/admin/teams/:id/invitations` | Required | `PLATFORM_ADMIN`, `FACULTY_ADMIN`, `CLUB_ADMIN` | `adminTeamsService.getSentTeamInvitations` | 200 |
| DELETE | `/v1/admin/teams` | `adminTeamsRouter` | `/:id/invitations/:invitationId` | `/v1/admin/teams/:id/invitations/:invitationId` | Required | `PLATFORM_ADMIN`, `FACULTY_ADMIN`, `CLUB_ADMIN` | `adminTeamsService.cancelInvitation` | 204 |
| GET | `/users` | `usersRouter` | `/me` | `/users/me` | Required | None | `usersService.getMe` | 200 |
| PATCH | `/users` | `usersRouter` | `/me` | `/users/me` | Required | None | `usersService.updateMe` | 200 |
| GET | `/users` | `usersRouter` | `/:id/profile` | `/users/:id/profile` | Required | None | `usersService.getPublicProfile` | 200 |
| POST | `/users` | `usersRouter` | `/me/push-token` | `/users/me/push-token` | Required | None | `usersService.registerPushToken` | 200 |
| GET | `/v1/admin` | `adminAuditLogsRouter` | `/` | `/v1/admin/audit-logs` | Required | `PLATFORM_ADMIN` | `auditLogsService.listLogs` | 200 |
| GET | `/v1/admin/users` | `adminUsersRouter` | `/` | `/v1/admin/users` | Required | `PLATFORM_ADMIN, FACULTY_ADMIN` | `adminUsersService.listUsers` | 200 |
| POST | `/v1/admin/users` | `adminUsersRouter` | `/:userId/role` | `/v1/admin/users/:userId/role` | Required | `PLATFORM_ADMIN` | `adminUsersService.updateUserRole` | 200 |
| PATCH | `/v1/admin/users` | `adminUsersRouter` | `/:userId/academic-batch` | `/v1/admin/users/:userId/academic-batch` | Required | `PLATFORM_ADMIN, FACULTY_ADMIN` (Target: ordinary STUDENT only) | `adminUsersService.updateAcademicBatch` | 200 |
| GET | `/v1` | `attendanceRouter` | `/events/:id/attendance/export` | `/v1/events/:id/attendance/export` | Required | `CLUB_ADMIN, CORE_MEMBER, FACULTY_MENTOR` | `attendanceService.exportEventAttendance` | `text/csv` |
| GET | `/v1/dashboard` | `dashboardRouter` | `/summary` | `/v1/dashboard/summary` | Required | None | `dashboardService.getSummary` | 200 |
| GET | `/v1` | `registrationsRouter` | `/events/:id/teams` | `/v1/events/:id/teams` | Required | None | `teamsService.listTeams` | 200 |

## Deferred / Future API

The following endpoints are **NOT IMPLEMENTED**, are **NOT PART OF CURRENT V1**, and the frontend **MUST NOT CALL** them. They are documented here solely for historical tracking and future backlog grooming.

| Method | Expected Path | Status | Reason |
|--------|---------------|--------|--------|
| GET | `/v1/home/feed` | DEFERRED | Mobile Home will compose data from existing APIs in parallel. The frontend MUST NOT call this. |
| GET | `/v1/events/:id/waitlist` | DEFERRED | Organizer/admin waitlist views use `GET /v1/events/:id/registrations?filter_status=WAITLISTED`. No participant waitlist UI in V1. |
| POST | `/v1/admin/points/adjust` | DEFERRED | Leaderboard scores are derived/immutable. Deferred until complete domain model exists. |
| GET | `/v1/analytics/*` | DEFERRED | Deferred until concrete analytics screen, metrics, and auth model are formally specified. |
| GET/POST | `/v1/announcements` | DEFERRED | Global messages use existing notifications infrastructure. No bulletin-board product surface in V1. |
| POST | `/upload/presign` (Example) | DEFERRED | File uploads are deferred (ADR-008). V1 Club Branding uses an externally hosted image URL, so this endpoint is not required for V1. |
| GET | `/v1/admin/students` | CURRENT | Required for NST Student Directory management (BACKEND GAP). |
| POST | `/v1/admin/students` | CURRENT | Required for manual student additions (BACKEND GAP). |
| POST | `/v1/admin/students/import` | CURRENT | Required for CSV imports (BACKEND GAP). |
| DELETE | `/v1/admin/students/:id` | CURRENT | Required to remove from Student Directory (BACKEND GAP). |

---

## Realtime Notifications API Contract

### `GET /v1/notifications/unread-count`
*   **Authentication**: Required (Bearer Token)
*   **Authorization**: Scoped to the authenticated user's ID.
*   **Response**: `{ "unread_count": number }`
*   **Behavior**: Queries authoritative `COUNT(*)` where `read_at IS NULL`. Relies on native `@@index([userId, readAt])`.

### `GET /v1/notifications/live`
*   **Authentication**: Required (Bearer Token or `?token=`)
*   **Authorization**: Isolated entirely to the authenticated user's ID. You cannot subscribe to another user's channel.
*   **Protocol**: Server-Sent Events (SSE). Keep-alive interval of 30 seconds.
*   **Behavior**: Establishes a session-bound PostgreSQL `LISTEN` to `user_<uuid>_notifications_live`.
*   **Reconnect Reconciliation**: The SSE stream is **not durable**. Upon reconnecting, the client MUST refetch `/v1/notifications` and `/v1/notifications/unread-count` via REST to reconcile missed state.
*   **Event Envelopes**:
    *   **Event:** `NOTIFICATION_CREATED`
        *   **Payload:** `{ "type": "NOTIFICATION_CREATED", "notification": { "id": "...", "title": "...", "body": "...", "type": "...", "metadata": {}, "createdAt": "..." } }`
    *   **Event:** `NOTIFICATION_READ`
        *   **Payload:** `{ "type": "NOTIFICATION_READ", "notification_id": "uuid" }`
    *   **Event:** `NOTIFICATIONS_READ_ALL`
        *   **Payload:** `{ "type": "NOTIFICATIONS_READ_ALL" }`

* **Event Team Attention**: `GET /v1/events` and `GET /v1/events/:id` must return `below_minimum_team_count` efficiently without N+1 queries. It represents the count of `REGISTERED` teams whose active member count is strictly less than the event's configured `minimum_team_size`.

## Event Lock State Contract

For `GET /v1/events` and `GET /v1/events/:id`:
*   **lock_state**: Responses include an authoritative lock state.
    *   `UNLOCKED`: Event is not manually locked and permanent boundary has not been reached.
    *   `MANUALLY_LOCKED`: Event is manually locked but permanent boundary has not been reached.
    *   `PERMANENTLY_LOCKED`: Event has reached the permanent lock boundary (`database_now >= event.end_time + interval '24 hours'`).
*   **Client Requirement**: Clients MUST NOT calculate permanent-lock state using device/browser time. The `lock_state` field is the single source of truth.
