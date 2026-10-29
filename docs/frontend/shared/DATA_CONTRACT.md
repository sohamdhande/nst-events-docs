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

## Forms and Registration Fields

**Backend Reference: Phase 2 Contract Implementation**

> **⚠️ Envelope & Casing Notice**:
> 1. Single standalone form CRUD endpoints (`POST /v1/events/:eventId/forms`, `GET /v1/events/:eventId/forms/:formId`, `PATCH /v1/events/:eventId/forms/:formId`) return direct **unwrapped** JSON objects.
> 2. Collection, field attachment, response, and profile endpoints wrap their returns in `{ data: ... }`.
> 3. Field attachment on legacy registration (`PUT /v1/events/:id/form-fields`) expects `fields: [{ field_definition_id, is_required, order }]` in **snake_case**.
> 4. Field attachment on standalone forms (`PATCH /v1/events/:eventId/forms/:formId/fields`) expects `fields: [{ fieldDefinitionId, isRequired, order }]` in **camelCase**.
> 5. Registration submissions (`POST /v1/events/:id/register`, `POST /v1/events/:id/teams`, `POST /v1/teams/:id/join`) and standalone form submissions (`POST /v1/events/:eventId/forms/:formId/responses`) expect `responses: [{ fieldDefinitionId, value }]` in **camelCase**.
> 6. All returned Prisma model entities use `camelCase`.

---

### `GET /v1/events/:eventId/forms` (list)
- **Method**: `GET`
- **Path**: `/v1/events/:eventId/forms`
- **Auth**: `authenticate` (Bearer JWT)
- **Request Body**: None
- **Envelope**: `{ data: EventForm[] }`
- **Authorization & Scoping**:
  - Registered Students: returns released and open forms (`releaseAt === null || releaseAt <= now()` AND `closeAt === null || closeAt >= now()`) for the event. Student must have active registration (`registrationStatus === 'REGISTERED'`).
  - Club Organizers / Admins (`isEventAdmin`): returns all forms for the event regardless of release/close dates.
  - Non-registered students: `403 Forbidden` (`You must be registered for this event to view forms.`).
- **Response Fields** (`EventForm` items in `data` array):

| Field | Type | Nullable | Notes |
|---|---|---|---|
| `id` | string (UUID) | No | Primary key |
| `eventId` | string (UUID) | No | Parent event ID |
| `title` | string | No | Form title (1-255 chars) |
| `description` | string | Yes | Form description markdown/text |
| `purpose` | enum | No | `FEEDBACK`, `QUIZ`, `CHECKIN`, `REGISTRATION`, `CUSTOM` |
| `releaseAt` | string (ISO) | Yes | Scheduled release timestamp |
| `closeAt` | string (ISO) | Yes | Scheduled close timestamp |
| `notifiedAt` | string (ISO) | Yes | Worker notification timestamp |
| `createdAt` | string (ISO) | No | Creation timestamp |
| `updatedAt` | string (ISO) | No | Last update timestamp |

- **Errors**: `401 Unauthorized`, `403 Forbidden`, `404 Not Found` (event deleted or not found).
- **Source**: `apps/api/src/modules/event-forms/event-forms.router.ts:21-33`, `apps/api/src/modules/event-forms/event-forms.service.ts:40-79`

---

### `POST /v1/events/:eventId/forms` (create)
- **Method**: `POST`
- **Path**: `/v1/events/:eventId/forms`
- **Auth**: `authenticate` + `canManageEvent` (Club Admin, Core Member, Faculty Mentor of event's primary club, or Platform/Faculty Admin)
- **Envelope**: `EventForm` (**UNWRAPPED object**, status `201 Created`)
- **Request Body**:

```json
{
  "title": "Post-Event Survey",
  "description": "Please provide your feedback",
  "purpose": "FEEDBACK",
  "releaseAt": "2026-09-25T10:00:00.000Z",
  "closeAt": "2026-09-27T18:00:00.000Z"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `title` | string | Yes | Min 1, max 255 chars |
| `description` | string | No | Optional description |
| `purpose` | enum | Yes | `FEEDBACK`, `QUIZ`, `CHECKIN`, `REGISTRATION`, `CUSTOM` |
| `releaseAt` | string (ISO) | No | Optional; must be valid ISO datetime string |
| `closeAt` | string (ISO) | No | Optional; must be > `releaseAt` if both are provided |

- **Response Fields**: Identical to `EventForm` fields documented above (unwrapped object, not inside `{ data: ... }`).
- **Errors**: `400 Bad Request` (Zod validation error or `closeAt must be after releaseAt`), `401 Unauthorized`, `403 Forbidden`, `404 Not Found`.
- **Source**: `apps/api/src/modules/event-forms/event-forms.router.ts:95-113`, `apps/api/src/modules/event-forms/event-forms.service.ts:81-125`

---

### `GET /v1/events/:eventId/forms/:formId` (detail)
- **Method**: `GET`
- **Path**: `/v1/events/:eventId/forms/:formId`
- **Auth**: `authenticate` (Bearer JWT)
- **Request Body**: None
- **Envelope**: `EventForm & { fields: EventFormField[] }` (**UNWRAPPED object**, status `200 OK`)
- **Authorization & Scoping**:
  - Registered Students: allowed only if student is registered for event AND form is released (`releaseAt === null || releaseAt <= now()`).
  - Club Organizers / Admins: allowed at any time.
- **Response Fields**:
  - All standard `EventForm` scalar fields: `id`, `eventId`, `title`, `description`, `purpose`, `releaseAt`, `closeAt`, `notifiedAt`, `createdAt`, `updatedAt`.
  - `fields`: Array of `EventFormField` objects ordered by `order asc`:

| Field | Type | Nullable | Notes |
|---|---|---|---|
| `id` | string (UUID) | No | Field join record ID |
| `eventId` | string (UUID) | No | Event ID |
| `formId` | string (UUID) | No | Parent form ID |
| `fieldDefinitionId` | string (UUID) | No | Field definition ID |
| `isRequired` | boolean | No | Whether answer is mandatory |
| `order` | number | No | Display sorting order |
| `fieldDefinition` | object | No | Field metadata (see schema below) |
| `fieldDefinition.id` | string (UUID) | No | Field definition ID |
| `fieldDefinition.key` | string | No | Unique slug key |
| `fieldDefinition.label` | string | No | Display label |
| `fieldDefinition.fieldType` | enum | No | `TEXT`, `NUMBER`, `TEXTAREA`, `SELECT`, `MULTI_SELECT`, `CHECKBOX`, `RADIO`, `DATE`, `FILE_UPLOAD`, `PHONE`, `EMAIL`, `URL` |
| `fieldDefinition.options` | string[] | Yes | JSON array of options for `SELECT`/`MULTI_SELECT`/`RADIO` |
| `fieldDefinition.isReusable` | boolean | No | Whether answer syncs to student profile |
| `fieldDefinition.createdByClubId` | string (UUID) | Yes | Owning club ID, or `null` for platform fields |

- **Errors**: `401 Unauthorized`, `403 Forbidden` (`You must be registered...` or `This form has not been released yet`), `404 Not Found` (form or event not found).
- **Source**: `apps/api/src/modules/event-forms/event-forms.router.ts:35-51`, `apps/api/src/modules/event-forms/event-forms.service.ts:127-167`

---

### `PATCH /v1/events/:eventId/forms/:formId` (update metadata)
- **Method**: `PATCH`
- **Path**: `/v1/events/:eventId/forms/:formId`
- **Auth**: `authenticate` + `canManageEvent`
- **Envelope**: `EventForm` (**UNWRAPPED object**, status `200 OK`)
- **Request Body**: Partial update of form metadata (all fields optional):

```json
{
  "title": "Updated Survey Title",
  "description": "Updated instructions",
  "purpose": "FEEDBACK",
  "releaseAt": "2026-09-25T12:00:00.000Z",
  "closeAt": "2026-09-28T18:00:00.000Z"
}
```

- **Response Fields**: Updated unwrapped `EventForm` object.
- **Errors**: `400 Bad Request` (Zod validation failure or `closeAt must be after releaseAt`), `401 Unauthorized`, `403 Forbidden`, `404 Not Found`.
- **Source**: `apps/api/src/modules/event-forms/event-forms.router.ts:115-134`, `apps/api/src/modules/event-forms/event-forms.service.ts:169-213`

---

### `PATCH /v1/events/:eventId/forms/:formId/fields` (replace form fields)
- **Method**: `PATCH`
- **Path**: `/v1/events/:eventId/forms/:formId/fields`
- **Auth**: `authenticate` + `canManageEvent`
- **Envelope**: `{ data: EventFormField[] }` (status `200 OK`)
- **Request Body**:

```json
{
  "fields": [
    {
      "fieldDefinitionId": "a1b2c3d4-0000-4000-8000-000000000001",
      "isRequired": true,
      "order": 1
    },
    {
      "fieldDefinitionId": "a1b2c3d4-0000-4000-8000-000000000002",
      "isRequired": false,
      "order": 2
    }
  ]
}
```

- **Ownership Validation**: Every `fieldDefinitionId` must either have `createdByClubId === null` (platform field) OR `createdByClubId` matching one of the event's organizing clubs (`eventClubs`). Attempting to attach another club's private field results in `403 Forbidden`.
- **Response Fields**: Array of `EventFormField` objects (with nested `fieldDefinition`) inside `{ data: [...] }`.
- **Errors**: `400 Bad Request` (missing/invalid fieldDefinitionId), `401 Unauthorized`, `403 Forbidden` (`Cannot attach field definitions owned by other clubs`), `404 Not Found`.
- **Source**: `apps/api/src/modules/event-forms/event-forms.router.ts:155-174`, `apps/api/src/modules/event-forms/event-forms.service.ts:215-282`

---

### `GET /v1/events/:eventId/forms/:formId/responses` (organizer responses view)
- **Method**: `GET`
- **Path**: `/v1/events/:eventId/forms/:formId/responses`
- **Auth**: `authenticate` + `canManageEvent`
- **Request Body**: None
- **Envelope**: `{ data: EventRegistrationResponse[] }` (status `200 OK`)
- **Response Fields** (each item in `data` array):

| Field | Type | Nullable | Notes |
|---|---|---|---|
| `id` | string (UUID) | No | Response record ID |
| `eventRegistrationId` | string (UUID) | No | Linked registration ID |
| `fieldDefinitionId` | string (UUID) | No | Field definition ID |
| `formId` | string (UUID) | No | Strictly scoped to `:formId` |
| `value` | string | No | Submitted answer value |
| `createdAt` | string (ISO) | No | Submission timestamp |
| `updatedAt` | string (ISO) | No | Update timestamp |
| `fieldDefinition` | object | No | `{ id, key, label, fieldType }` |
| `eventRegistration` | object | No | Linked registration summary |
| `eventRegistration.id` | string (UUID) | No | Registration ID |
| `eventRegistration.userId` | string (UUID) | No | Responding student ID |
| `eventRegistration.user` | object | No | `{ id, fullName, email }` |

- **Errors**: `401 Unauthorized`, `403 Forbidden`, `404 Not Found` (form or event not found).
- **Source**: `apps/api/src/modules/event-forms/event-forms.router.ts:176-193`, `apps/api/src/modules/event-forms/event-forms.service.ts:400-438`

---

### `POST /v1/events/:eventId/forms/:formId/responses` (submit student responses)
- **Method**: `POST`
- **Path**: `/v1/events/:eventId/forms/:formId/responses`
- **Auth**: `authenticate` (student caller)
- **Envelope**: `{ data: EventRegistrationResponse[] }` (status `201 Created`)
- **Request Body**:

```json
{
  "responses": [
    {
      "fieldDefinitionId": "a1b2c3d4-0000-4000-8000-000000000001",
      "value": "My text answer"
    }
  ]
}
```

- **Submission Rules & Side Effects**:
  1. Student must have an active registration for `:eventId` (`registrationStatus === 'REGISTERED'`). Otherwise `403 Forbidden`.
  2. Form must be open (`now >= releaseAt && (!closeAt || now <= closeAt)`). Otherwise `400 Bad Request` (`This form is closed for submissions`).
  3. All required fields (`isRequired: true`) on the form must be present with non-empty values. Otherwise `400 Bad Request`.
  4. Field values are validated against `fieldType` rules (`validateFieldValue`).
  5. Profile updates: For fields where `fieldDefinition.isReusable === true`, the submitted value upserts `student_profile_values` for the user. Non-reusable fields NEVER update `student_profile_values`.
  6. Stored records have `formId = :formId` and `eventRegistrationId = studentRegistration.id`.
- **Errors**: `400 Bad Request` (closed, missing required field, format validation failure), `401 Unauthorized`, `403 Forbidden` (`You must be registered for this event`), `404 Not Found`.
- **Source**: `apps/api/src/modules/event-forms/event-forms.router.ts:53-71`, `apps/api/src/modules/event-forms/event-forms.service.ts:284-398`

---

### `GET /v1/events/:eventId/forms/:formId/my-response` (student own response)
- **Method**: `GET`
- **Path**: `/v1/events/:eventId/forms/:formId/my-response`
- **Auth**: `authenticate` (student caller)
- **Request Body**: None
- **Envelope**: `{ data: EventRegistrationResponse[] }` (status `200 OK`)
- **Authorization & Scoping**:
  - Returns responses for the calling student (`user_id = current_user_id()`) for `:formId` and `:eventId`.
  - If the student has not yet submitted responses, returns `{ data: [] }` with `200 OK`.
- **Response Fields**: Array of `EventRegistrationResponse` items with `{ id, eventRegistrationId, fieldDefinitionId, formId, value, createdAt, updatedAt, fieldDefinition: { id, key, label, fieldType } }`.
- **Errors**: `401 Unauthorized`, `403 Forbidden` (student not registered for event), `404 Not Found` (form or event not found).
- **Source**: `apps/api/src/modules/event-forms/event-forms.router.ts:73-89`, `apps/api/src/modules/event-forms/event-forms.service.ts:440-463`

---

### `GET /v1/events/:id/form-fields` (legacy registration fields)
- **Method**: `GET`
- **Path**: `/v1/events/:id/form-fields`
- **Auth**: `optionalAuthenticate` (Bearer token optional)
- **Request Body**: None
- **Envelope**: `{ data: EventFormField[] }` (status `200 OK`)
- **Scoping**: Returns only registration fields (`formId IS NULL`) for event `:id`.
- **Access Semantics**:
  - **Authenticated Organizer/Admin**: can view fields for any event they manage (including DRAFT, PRIVATE, or restricted events).
  - **Authenticated Student**: can view fields if event is `PUBLISHED` & `PUBLIC` and student is audience-eligible (`ALL_STUDENTS` or student's batch is in `event_audience_batches`). If ineligible: returns `403 Forbidden` (`AUDIENCE_NOT_ELIGIBLE`).
  - **Anonymous / Unauthenticated Caller (Option B)**: queries `eventFormField` directly under database RLS. For `PUBLISHED`, `PUBLIC`, and `audience = 'ALL_STUDENTS'` events, returns public registration fields (`{ data: [...] }`). For restricted-audience (`SPECIFIC_BATCHES`), private, draft, or deleted events, database RLS evaluates to false and returns `{ data: [] }` with `200 OK`. Anonymous users never receive restricted-event registration field data.
- **Response Fields** (`data` array):
  - Array of `EventFormField` objects ordered by `order asc`:
    `{ id, eventId, formId (null), fieldDefinitionId, isRequired, order, fieldDefinition: { id, key, label, fieldType, options, isReusable, createdByClubId, createdAt, updatedAt } }`.
- **Errors**: `401 Unauthorized` (if malformed token provided), `403 Forbidden` (`AUDIENCE_NOT_ELIGIBLE`), `404 Not Found` (for authenticated users if event missing/deleted).
- **Source**: `apps/api/src/modules/events/events.router.ts:114-126`, `apps/api/src/modules/events/events.service.ts:718-782`

---

### `PUT /v1/events/:id/form-fields` (replace registration fields)
- **Method**: `PUT`
- **Path**: `/v1/events/:id/form-fields`
- **Auth**: `authenticate` + `canManageEvent`
- **Envelope**: `{ data: EventFormField[] }` (status `200 OK`)
- **Request Body**:

```json
{
  "fields": [
    {
      "fieldDefinitionId": "a1b2c3d4-0000-4000-8000-000000000001",
      "isRequired": true,
      "order": 1
    }
  ]
}
```

- **Ownership Validation**: Every `field_definition_id` must have `createdByClubId === null` OR match one of the event's organizing clubs. If another club's private field definition is specified, returns `403 Forbidden`.
- **Scoping**: Replaces only registration fields (`formId IS NULL`). Standalone form fields (`formId IS NOT NULL`) are unaffected.
- **Response Fields**: Array of created `EventFormField` objects with nested `fieldDefinition` inside `{ data: [...] }`.
- **Errors**: `400 Bad Request`, `401 Unauthorized`, `403 Forbidden` (`Cannot attach field definitions owned by other clubs`), `404 Not Found`.
- **Source**: `apps/api/src/modules/events/events.router.ts:128-146`, `apps/api/src/modules/events/events.service.ts:808-869`

---

### `GET /v1/events/:id/registrations/:registrationId/responses` (registration responses)
- **Method**: `GET`
- **Path**: `/v1/events/:id/registrations/:registrationId/responses`
- **Auth**: `authenticate` + `canManageEvent`
- **Request Body**: None
- **Envelope**: `{ data: EventRegistrationResponse[] }` (status `200 OK`)
- **Scoping**: Strictly scoped to `eventRegistrationId = :registrationId` AND `formId IS NULL` (only registration responses, never standalone form responses).
- **Response Fields** (`data` array):
  - Array of `EventRegistrationResponse` items with `{ id, eventRegistrationId, fieldDefinitionId, formId: null, value, createdAt, updatedAt, fieldDefinition: { id, key, label, fieldType } }`.
- **Errors**: `401 Unauthorized`, `403 Forbidden`, `404 Not Found` (registration does not exist or does not belong to `:id`).
- **Source**: `apps/api/src/modules/registrations/registrations.router.ts:29-45`, `apps/api/src/modules/registrations/registrations.service.ts:148-175`

---

### `GET /v1/users/me/profile-values` (prefill profile values)
- **Method**: `GET`
- **Path**: `/v1/users/me/profile-values` (also dual-mounted at `/users/me/profile-values`)
- **Auth**: `authenticate` (Bearer JWT)
- **Request Body**: None
- **Envelope**: `{ data: StudentProfileValue[] }` (status `200 OK`)
- **Response Fields** (`data` array):

| Field | Type | Nullable | Notes |
|---|---|---|---|
| `id` | string (UUID) | No | Record ID |
| `userId` | string (UUID) | No | Owning student user ID |
| `fieldDefinitionId` | string (UUID) | No | Field definition ID |
| `value` | string | No | Saved profile response value |
| `createdAt` | string (ISO) | No | Timestamp |
| `updatedAt` | string (ISO) | No | Timestamp |
| `fieldDefinition` | object | No | Field metadata `{ id, key, label, fieldType, options, isReusable }` |

- **Errors**: `401 Unauthorized`.
- **Source**: `apps/api/src/modules/users/users.router.ts:21-28`, `apps/api/src/modules/users/users.service.ts:63-71`

---

## System Policy Contracts

### `GET /v1/policies/:slug` (public published policy)
- **Method**: `GET`
- **Path**: `/v1/policies/:slug`
- **Auth**: None (public unauthenticated read)
- **Request Body**: None
- **Envelope**: `{ data: PublishedPolicyResponse }` (status `200 OK`)
- **Cache Headers**: `Cache-Control: public, max-age=60, s-maxage=60, stale-while-revalidate=60`
- **Response Fields** (`data` object):

| Field | Type | Nullable | Notes |
|---|---|---|---|
| `slug` | string | No | Policy identifier slug (e.g. `'privacy-policy'`) |
| `title` | string | No | Policy display title |
| `content` | string | No | Authoritative published Markdown content (draft content strictly excluded) |
| `version` | number | No | Integer published version, increments only on publication |
| `publishedAt` | string (ISO) | Yes | Timestamp when current version was published |
| `updatedAt` | string (ISO) | No | Record update timestamp |

- **Errors**: `400 Bad Request` (invalid slug format), `404 Not Found` (policy missing or not yet published).
- **Source**: `apps/api/src/modules/policies/policies.router.ts`, `apps/api/src/modules/policies/policies.service.ts`

---

### `GET /v1/admin/policies/:slug` (admin policy editor state)
- **Method**: `GET`
- **Path**: `/v1/admin/policies/:slug`
- **Auth**: `authenticate` + `requireRole('PLATFORM_ADMIN')`
- **Request Body**: None
- **Envelope**: `{ data: AdminPolicyResponse }` (status `200 OK`)
- **Response Fields** (`data` object):

| Field | Type | Nullable | Notes |
|---|---|---|---|
| `slug` | string | No | Policy identifier slug |
| `title` | string | No | Policy display title |
| `publishedContent` | string | No | Currently published Markdown content |
| `draftContent` | string | No | Working draft Markdown content |
| `publishedVersion` | number | No | Published version number |
| `publishedAt` | string (ISO) | Yes | Publication timestamp |
| `concurrencyRevision` | number | No | Monotonically increasing revision counter (increments on every draft save and publish) |
| `publisher` | object | Yes | Publisher user info `{ id, fullName, email }` (null if publisher deleted or not yet published) |
| `updatedAt` | string (ISO) | No | Timestamp of last modification |

- **Errors**: `401 Unauthorized`, `403 Forbidden` (non-platform-admin), `404 Not Found`.
- **Source**: `apps/api/src/modules/policies/policies.router.ts`, `apps/api/src/modules/policies/policies.service.ts`

---

### `PUT /v1/admin/policies/:slug` (admin save draft or publish)
- **Method**: `PUT`
- **Path**: `/v1/admin/policies/:slug`
- **Auth**: `authenticate` + `requireRole('PLATFORM_ADMIN')`
- **Envelope**: `{ data: AdminPolicyResponse }` (status `200 OK`)
- **Request Body**:

```json
{
  "action": "SAVE_DRAFT",
  "content": "# Privacy Policy...",
  "base_concurrency_revision": 3
}
```

- **Validation Rules**:
  - `action`: Exactly `'SAVE_DRAFT'` or `'PUBLISH'`.
  - `content`: Non-empty Markdown string with UTF-8 byte length $\le 102,400$ bytes (100 KiB), validated via `Buffer.byteLength(val, 'utf8') <= 102400`.
  - `base_concurrency_revision`: Non-negative integer representing the server revision known to the client.
- **Execution Semantics**:
  - Executed inside PostgreSQL transaction using `withUserContext(adminId, ...)`.
  - **Optimistic Concurrency Check**: Validates `base_concurrency_revision === current policy.concurrency_revision`.
  - **Draft Save (`action === 'SAVE_DRAFT'`)**: Updates `draft_content`, increments `concurrency_revision`, preserves `published_content` and `published_version`. Records `POLICY_SAVED_DRAFT` in `audit_logs` (SHA-256 hash and metadata only, < 500 bytes).
  - **Publish (`action === 'PUBLISH'`)**: Atomically copies content to `published_content` and `draft_content`, increments `published_version` by 1, increments `concurrency_revision` by 1, updates `published_at` to now and `published_by_user_id` to current user. Records `POLICY_PUBLISHED` in `audit_logs` with complete previous and new Markdown snapshots.
- **Errors**:
  - `400 Bad Request`: Validation failure (empty content, invalid action, content exceeds 100 KiB UTF-8).
  - `401 Unauthorized`: Missing or invalid JWT.
  - `403 Forbidden`: Authenticated user is not `PLATFORM_ADMIN`.
  - `404 Not Found`: Policy slug not found.
  - `409 Conflict`: Revision mismatch (`base_concurrency_revision !== policy.concurrency_revision`). Returns:
    ```json
    {
      "error": {
        "code": "POLICY_CONFLICT",
        "message": "Policy has been modified by another administrator.",
        "details": {
          "server_revision": 4,
          "server_draft_content": "...",
          "server_published_content": "...",
          "server_updated_at": "2026-10-29T10:00:00.000Z",
          "last_updated_by": {
            "id": "uuid",
            "full_name": "Admin Name",
            "email": "admin@nstsdc.org"
          }
        }
      }
    }
    ```
- **Source**: `apps/api/src/modules/policies/policies.router.ts`, `apps/api/src/modules/policies/policies.service.ts`

---

## Outstanding follow-ups from this audit

1. All list/detail shape mismatches (Club, User, Registration) should be flagged to
   backend as a code-quality issue, but are NOT to be "fixed" from the frontend —
   build against the shapes as they actually are, documented above.

