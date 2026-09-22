# Data Contract

**Status: RE-VERIFIED AGAINST ACTUAL BACKEND SOURCE (Phase 22C)**
**Backend HEAD: `f2fcc782e3e7a27f67a19778de7f391ac2144b30`**

## How to read this document

Unlike a typical data contract, this backend does **not** return a consistent shape
per resource. The same logical entity (Event, Club, User, Registration) can return
different field casing — and in some cases different fields entirely — depending on
**which endpoint** you call. This was discovered during a Phase 22C ground-truth audit
against actual service code (not Prisma schema, not prior docs).

**Rule for engineers: identify the exact endpoint first, then use ONLY the field list
documented under that endpoint. Do not assume a shape from one endpoint applies to
another endpoint for the "same" resource.**

Every field below is sourced from the literal object passed to `res.json()` in the
corresponding service/controller — verified file:line citations are included. Where
Prisma raw output is used directly, fields are `camelCase` (Prisma's default). Where
the backend manually constructs/transforms the response object, casing may be
`snake_case` or mixed — this is called out per endpoint.

---

## Event

### `GET /v1/events` (list) — camelCase, Prisma-raw + 1 manual field
| Field | Type | Notes |
|---|---|---|
| `id`, `title`, `description`, `state`, `visibility` | string / enum | Prisma-raw |
| `startTime`, `endTime`, `locationName` | string (ISO) / string | Prisma-raw |
| `eventType`, `registrationType`, `attendanceType` | enum | Prisma-raw |
| `isLocked`, `maxCapacity`, `registrationCount` | boolean / number | Prisma-raw |
| `metadata`, `createdBy`, `createdAt`, `updatedAt` | object / string | Prisma-raw |
| `location_geofence` | object \| null | **Manually appended, snake_case** — the one exception |

Source: `apps/api/src/modules/events/events.service.ts:136` (list query), `:132` (geofence append)

### `GET /v1/events/:id` (detail) — same shape as list
Identical field set and casing to the list endpoint above, including the same
`location_geofence` snake_case exception.

Source: `apps/api/src/modules/events/events.service.ts:160` (detail query), `:158` (geofence append)

> **Derived fields (client-side only, not backend fields):**
> - "Open for registration" = `state === 'PUBLISHED' && (maxCapacity === null || registrationCount < maxCapacity) && !isLocked`
> - Do not expect a single backend enum for this — compute it client-side.

---

## AttendanceSession

### `GET /v1/events/:id/sessions` (list) — camelCase, Prisma-raw
| Field | Type |
|---|---|
| `id`, `eventId`, `title` | string |
| `startTime`, `endTime`, `openAt`, `closeAt` | string (ISO) |
| `geofenceRadius` | number |

Source: `apps/api/src/modules/events/events.service.ts:444` (`eventsService.listSessions`)

> Note: this is a **separate route** from attendance records below. Confirmed present
> in `docs/api/02-api-routing-matrix.md` line 66 (`GET /v1/events/:id/sessions` →
> `eventsService.listSessions`). Do not confuse with `/attendance`.

## AttendanceRecord

### `GET /v1/events/:id/attendance` — camelCase, Prisma-raw
**Note**: The response is a paginated envelope: `{ data: AttendanceRecord[], nextCursor: string | null }`, not a bare array. The fields below apply to the objects in the `data` array.

| Field | Type |
|---|---|
| `id`, `sessionId`, `userId` | string |
| `markedBy` | string \| null |
| `markedAt` | string (ISO) |
| `method` | enum (`QR`, `MANUAL`, etc.) |
| `status` | enum (`PRESENT`, `ABSENT`, `EXCUSED`) |
| `auditMetadata` | object |

Source: `apps/api/src/modules/attendance/attendance.service.ts:194`

---

## Registration

**⚠️ Shape differs significantly between these two endpoints — they are not the same object.**

### `GET /v1/events/:id/my-registration` — manually transformed, minimal
| Field | Type |
|---|---|
| `status` | enum: `REGISTERED`, `WAITLISTED`, `CANCELLED`, `NOT_REGISTERED` |

Source: `apps/api/src/modules/registrations/registrations.service.ts:94-97`

> Note: If the user is not registered, the endpoint returns `{ status: "NOT_REGISTERED" }` with a 200 OK status. It does NOT return a 404.
>
> This endpoint does **not** return `team_id`. If team association is needed for the
> current user's registration, it is not available from this endpoint — flag as a
> SPECIFICATION GAP if a screen spec requires it here.

### `GET /v1/events/:id/registrations` (organizer/admin list) — camelCase, Prisma-raw
| Field | Type |
|---|---|
| `id`, `eventId`, `userId`, `teamId` | string |
| `registrationStatus` | enum |
| `participationRole` | string |
| `registeredAt` | string (ISO) |
| `user.fullName`, `user.globalRole` | nested object, camelCase |

Source: `apps/api/src/modules/registrations/registrations.service.ts:66`

> Use `filter_status=WAITLISTED` query param on this endpoint for organizer/admin
> waitlist views — there is no separate waitlist endpoint.

---

## Team

### `GET /v1/events/:id/teams` — manually transformed, snake_case
| Field | Type |
|---|---|
| `id`, `name` | string |
| `leader_id`, `leader_name` | string |
| `member_count` | number |
| `members[].user_id`, `members[].full_name`, `members[].registration_status` | string |

Source: `apps/api/src/modules/teams/teams.service.ts:37-47`

---

## Club

**⚠️ Shape differs between list and detail — this is the same pattern as Registration.**

### `GET /v1/clubs` (list) and `GET /v1/clubs/search` — manually transformed, snake_case
| Field | Type |
|---|---|
| `id`, `name`, `description`, `status` | string / enum |
| `banner_url` | string \| null |
| `event_count`, `member_count` | number |

Source: `apps/api/src/modules/clubs/clubs.service.ts:80`, `:260`

### `GET /v1/clubs/:id` (detail) — manually transformed, snake_case
| Field | Type |
|---|---|
| `id`, `name`, `description`, `status` | string / enum |
| `banner_url` | string \| null |
| `event_count` | number |
| `members` | array |
| `members[].user_id`, `members[].full_name`, `members[].avatar_url` | string |

Source: `apps/api/src/modules/clubs/clubs.service.ts:107-123`

> **Critical for implementers**: do not reuse a single `Club` TS type across both
> endpoints. Define `ClubListItem` and `ClubDetail` as distinct types.

### `PATCH /v1/clubs/:id` (update) — same shape as detail (GET /clubs/:id)
**Request:**
```json
{
  "name": "string?",
  "description": "string | null?",
  "banner_url": "string | null?"
}
```
**Response:** Same shape as `GET /v1/clubs/:id`

---

## User

**⚠️ Shape differs between self-profile and admin list.**

### `GET /v1/users/me` — manually transformed, snake_case
| Field | Type |
|---|---|
| `id`, `email` | string |
| `full_name` | string \| null |
| `avatar_url` | string \| null |
| `global_role` | enum: `STUDENT`, `FACULTY_ADMIN`, `PLATFORM_ADMIN` |
| `club_memberships` | array |

Source: `apps/api/src/modules/users/users.service.ts:30-41`

### `GET /v1/admin/users` — camelCase, context-dependent expansion

**Query**: `q` (optional string), `cursor` (optional user ID), `limit` (1–100;
default 20), and `scope` (optional; `administrators`).

**Authorization**: `PLATFORM_ADMIN` or `FACULTY_ADMIN`.

**Envelope**: `{ data, pagination }`; `pagination.next_cursor` is a string when
another page exists and omitted when it does not.

Every item in `data` contains:

| Field | Type |
|---|---|
| `id`, `email` | string |
| `fullName` | string \| null |
| `globalRole` | `STUDENT` \| `FACULTY_MENTOR` \| `FACULTY_ADMIN` \| `PLATFORM_ADMIN` |
| `academicProfile` | `{ batchId, assignmentSource, assignedAt, batch }` \| null |
| `academicProfile.assignmentSource` | `INSTITUTIONAL_EMAIL_INFERENCE` \| `ADMIN_OVERRIDE` |
| `academicProfile.batch` | `{ id, programId, admissionYear, graduationYear, program }` \| null |
| `academicProfile.batch.program` | `{ id, name, code }` |

With `scope=administrators`, the server returns only users who have a global
administrative role (`PLATFORM_ADMIN`, `FACULTY_ADMIN`, or `FACULTY_MENTOR`) or
an active `CLUB_ADMIN` membership. In that scope only, every item also includes
`clubMemberships: Array<{ id, role: 'CLUB_ADMIN', club: { id, name } }>` and
the top-level envelope includes `platform_admin_count`.

`CLUB_ADMIN` is a club-membership role, not a value of `globalRole`; do not
combine them in a shared role type.

Source: `apps/api/src/modules/admin/users.router.ts:12-24`,
`apps/api/src/modules/admin/users.service.ts:7-105`

> **Critical for implementers**: `/users/me` and `/admin/users` are NOT
> interchangeable types. `full_name` (self) vs `fullName` (admin) — do not merge into
> one `User` interface used across both.

---

## Notification

### `GET /v1/notifications` — camelCase, Prisma-raw
| Field | Type |
|---|---|
| `id`, `title`, `body`, `type` | string |
| `metadata` | object |
| `readAt` | string (ISO) \| null |
| `createdAt` | string (ISO) |

Source: `apps/api/src/modules/notifications/notifications.service.ts:44`

> Read state is derived client-side as `readAt != null`.

---

## Leaderboard

### `GET /v1/leaderboard/students` — snake_case, raw SQL projection
| Field | Type |
|---|---|
| `user_id`, `display_name` | string |
| `total_points` | number |
| `attendance_points`, `contribution_points`, `competition_points` | number — **STUBBED, hardcoded to 0 in the materialized view; do not treat as real yet** |
| `last_refreshed_at` | string (ISO) |

Source: `apps/api/src/modules/leaderboard/leaderboard.service.ts:17-24` (`$queryRaw`)

### `GET /v1/leaderboard/clubs` — snake_case, raw SQL projection
| Field | Type |
|---|---|
| `club_id`, `club_name` | string |
| `total_points`, `event_count`, `member_count` | number |
| `last_refreshed_at` | string (ISO) |

Source: `apps/api/src/modules/leaderboard/leaderboard.service.ts:52-58` (`$queryRaw`)

---

## Dashboard Summary

### `GET /v1/dashboard/summary` — manually constructed, snake_case
| Field | Type |
|---|---|
| `upcoming_events` | array — each item includes `start_time` |
| `pending_approvals` | array of objects (`{ id, title }`) |
| `my_clubs` | array — each item includes `member_count` |

Source: `apps/api/src/modules/dashboard/dashboard.service.ts:96-100`, `:29-33`, `:90-94`



---

## AuditLog

### `GET /v1/admin/audit-logs` — camelCase, Prisma-raw
| Field | Type |
|---|---|
| `id` | string |
| `action`, `actorId`, `entityType`, `entityId` | string |
| `previousState`, `newState` | object \| null |
| `ipAddress` | string \| null |
| `createdAt` | string (ISO) |

Source: `apps/api/src/modules/admin/audit-logs.service.ts:56-62`

---

## SSE / Live Events

- Source Endpoint: `GET /v1/events/:id/live`
- Connection: EventSource / Server-Sent Events
- Authentication: Bearer token via query parameter (`?token=...`)
- Keep-Alive: emits `heartbeat` every 30s

**CONFIRMED (Phase 22C): exactly 3 event types are emitted.** Verified against actual
`pg_notify` trigger code and the SSE router, not against prior documentation claims.

All events are emitted as stringified JSON in the `data` field:
`{ "type": string, "payload": object }`

### 1. `registration_count`
- `payload.count` (number, non-null)
- Emitted when: a user registers, joins a team, leaves, or cancels
- Source: `packages/database/prisma/migrations/20260807212000_phase8_realtime_listen_notify/migration.sql:81,133,174`

### 2. `waitlist_update`
- `payload.user_id` (string UUID, non-null)
- `payload.status` (string, non-null) — always `"REGISTERED"`
- Emitted when: a waitlisted user is auto-promoted due to a cancellation
- Source: `packages/database/prisma/migrations/20260807212000_phase8_realtime_listen_notify/migration.sql:44-47`

### 3. `heartbeat`
- `payload.timestamp` (string ISO, non-null)
- Emitted every 30s by the Express server
- Source: `apps/api/src/modules/sse/sse.router.ts:50-52`

**NOT implemented — do not build UI depending on these:** `attendance_count`,
`session_opened`, `session_closed`, `lock_status`. These appear in older docs/ADRs but
have no emission code anywhere in the backend.

---

## Announcement

No CRUD endpoints exist in V1 — deferred. See `BACKEND_CHANGES_REQUIRED.md`
(`BE-CONFIRMED-011`). Do not build a client for this resource.

---

## Outstanding follow-ups from this audit

1. All list/detail shape mismatches (Club, User, Registration) should be flagged to
   backend as a code-quality issue, but are NOT to be "fixed" from the frontend —
   build against the shapes as they actually are, documented above.