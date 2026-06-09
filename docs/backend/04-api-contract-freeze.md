# API Contract Freeze

> **Status**: FROZEN | **Version**: 1.0 | **Date**: 2026-06-09
> **Authority**: API Freeze V1, Security Freeze V1, RPC Catalog


### `PATCH /events/:id/sessions/:sessionId`
| Field | Value |
|
### `GET /events/:id/results`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated |
| **Response 200** | `{ "data": [{ "id": "string", "user_id": "string", "result_type": "string" }] }` |

---|---|
| **Auth** | Bearer JWT |
| **Roles** | `CLUB_ADMIN`, `CORE_MEMBER` |
| **Request** | `{ "title"?: "string", "start_time"?: "string", "end_time"?: "string", "open_at"?: "string", "close_at"?: "string", "geofence_radius"?: number }` |
| **Response 200** | `{ "id": "string", "title": "string", "open_at": "string", "close_at": "string" }` |

---

## Global Conventions

### Base URL
`https://api.nstsdc.org/v1`

### Authentication
All endpoints (except `GET /auth/google` and `GET /auth/google/callback`) require `Authorization: Bearer <access_jwt>`.

### Error Response Format (RFC 7807)
```json
{
  "type": "https://api.nstsdc.org/errors/forbidden",
  "title": "Forbidden",
  "status": 403,
  "detail": "You do not have CLUB_ADMIN role for this club.",
  "instance": "/v1/events/abc-123"
}
```

### Cursor Pagination
All list endpoints use cursor-based pagination:
```
GET /v1/events?cursor=<last_id>&limit=20&sort=start_time&order=desc
```
Response:
```json
{
  "data": [...],
  "pagination": {
    "next_cursor": "uuid-of-last-item",
    "has_more": true,
    "total_count": 142
  }
}
```
- `cursor`: UUID of the last item from the previous page (omit for first page)
- `limit`: Page size (default 20, max 100)
- `sort`: Sort field (endpoint-specific)
- `order`: `asc` or `desc`

### Filtering
Query parameters with `filter_` prefix:
```
GET /v1/events?filter_state=PUBLISHED&filter_event_type=HACKATHON&filter_club_id=uuid
```

### Common Status Codes
| Code | Meaning |
|---|---|
| 200 | Success |
| 201 | Created |
| 204 | No Content (successful delete/update) |
| 400 | Bad Request (validation error) |
| 401 | Unauthorized (missing/invalid JWT) |
| 403 | Forbidden (insufficient role) |
| 404 | Not Found |
| 409 | Conflict (duplicate registration, etc.) |
| 422 | Unprocessable Entity (business rule violation) |
| 429 | Too Many Requests (rate limited) |
| 500 | Internal Server Error |

---

## Auth Endpoints

### `GET /auth/google`
| Field | Value |
|---|---|
| **Auth** | None |
| **Roles** | Public |
| **Description** | Redirects to Google OAuth consent screen |
| **Response** | 302 Redirect to `accounts.google.com` |

### `GET /auth/google/callback`
| Field | Value |
|---|---|
| **Auth** | None (OAuth code in query) |
| **Roles** | Public |
| **Description** | Handles OAuth code exchange, verifies `id_token` (`aud`, `hd`), enforces domain restriction, upserts user on `google_sub`, issues tokens |
| **Request** | Query: `code`, `state` |
| **Response 200** | `{ access_token, expires_in, user: { id, email, full_name, global_role } }` + Set-Cookie (refresh token) |
| **Response 403** | Domain not allowed |

### `POST /auth/refresh`
| Field | Value |
|---|---|
| **Auth** | HttpOnly refresh cookie |
| **Roles** | Authenticated |
| **Description** | Rotates refresh token, issues new access JWT |
| **Response 200** | `{ access_token, expires_in }` + Set-Cookie (new refresh token) |
| **Response 401** | Token expired/revoked/invalid |

### `POST /auth/logout`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT + HttpOnly cookie |
| **Roles** | Authenticated |
| **Description** | Revokes refresh token, clears cookie |
| **Response 204** | `{}` |

---

## User Endpoints

### `GET /users/me`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated |
| **Response 200** | `{ id, email, full_name, avatar_url, global_role, club_memberships: [{ club_id, club_name, role }] }` |

### `PATCH /users/me`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated |
| **Request** | `{ full_name?: string }` |
| **Response 200** | `{ "id": "string", "email": "string", "full_name": "string" }` |


### `POST /users/me/push-token`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated |
| **Request** | `{ "device_id": "string", "expo_token": "string", "platform": "string" }` |
| **Response 200** | `{ "success": true }` |

### `GET /users/:id/profile`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated |
| **Description** | Public profile view (limited fields) |
| **Response 200** | `{ id, full_name, avatar_url, club_memberships: [{ club_id, club_name, role }] }` |

---

## Club Endpoints

### `GET /clubs`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated |
| **Query** | `cursor` (string), `limit` (number), `sort` (name, created_at) |
| **Response 200** | `{ "data": [...], "pagination": { "next_cursor": "string", "has_more": boolean } }` |

### `GET /clubs/:id`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated |
| **Response 200** | `{ id, name, description, banner_url, status, members: [...], event_count }` |


### `PATCH /clubs/:id/status`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `PLATFORM_ADMIN` |
| **Request** | `{ "status": "ACTIVE" | "INACTIVE" | "DISSOLVED" }` |
| **Response 200** | `{ "id": "string", "status": "string" }` |

### `POST /clubs`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `PLATFORM_ADMIN` |
| **Request** | `{ name, description }` |
| **Response 201** | `{ "id": "string", "name": "string", "status": "string" }` |

### `POST /clubs/:id/members`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `CLUB_ADMIN`, `FACULTY_MENTOR`, `FACULTY_ADMIN`, `PLATFORM_ADMIN` |
| **Request** | `{ user_id, role: ClubRole }` |
| **Response 201** | `{ "id": "string", "user_id": "string", "role": "string" }` |
| **Response 409** | User already a member |

### `PATCH /clubs/:id/members/:userId`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `CLUB_ADMIN`, `FACULTY_MENTOR`, `FACULTY_ADMIN`, `PLATFORM_ADMIN` |
| **Request** | `{ role: ClubRole }` |
| **Response 200** | `{ "id": "string", "user_id": "string", "role": "string" }` |

### `DELETE /clubs/:id/members/:userId`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `CLUB_ADMIN`, `FACULTY_MENTOR`, `FACULTY_ADMIN`, `PLATFORM_ADMIN` |
| **Response 204** | `{}` |

---

## Event Endpoints

### `GET /events`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated |
| **Query** | `cursor` (string), `limit` (number), `sort` (start_time, created_at), `filter_state`, `filter_event_type`, `filter_club_id`, `filter_visibility`, `q` (full-text search) |
| **Response 200** | `{ "data": [...], "pagination": { "next_cursor": "string", "has_more": boolean } }` |

### `GET /events/:id`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated (RLS-filtered) |
| **Response 200** | Full event detail with clubs, sessions, registration stats |

### `POST /events`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `CLUB_ADMIN`, `CORE_MEMBER` |
| **Request** | `{ title, description, start_time, end_time, location_name, location_lat, location_lng, event_type, visibility, registration_type, metadata, attendance_type, max_capacity, club_ids: [{ club_id, is_primary }] }` |
| **Response 201** | `{ "id": "string", "title": "string", "state": "DRAFT" }` |

### `PATCH /events/:id`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `CLUB_ADMIN` (only in DRAFT state) |
| **Request** | `{ "title"?: "string", "description"?: "string", "start_time"?: "string", "end_time"?: "string", "location_name"?: "string", "location_lat"?: number, "location_lng"?: number, "event_type"?: "string", "visibility"?: "string", "registration_type"?: "string", "metadata"?: object, "attendance_type"?: "string", "max_capacity"?: number }` |
| **Response 200** | `{ "id": "string", "title": "string", "state": "string" }` |
| **Response 422** | Event not in DRAFT state |

### `DELETE /events/:id`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `CLUB_ADMIN` |
| **Response 204** | `{}` |

### `POST /events/:id/submit-for-approval`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `CLUB_ADMIN`, `CORE_MEMBER` |
| **Description** | DRAFT → PENDING_APPROVAL. Notifies Faculty Mentor. |
| **Response 200** | `{ state: "PENDING_APPROVAL" }` |
| **Response 422** | Not in DRAFT state |

### `POST /events/:id/approve`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `FACULTY_MENTOR`, `FACULTY_ADMIN`, `PLATFORM_ADMIN` |
| **Description** | PENDING_APPROVAL → PUBLISHED. Triggers push notifications. |
| **Response 200** | `{ state: "PUBLISHED" }` |

### `POST /events/:id/reject`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `FACULTY_MENTOR`, `FACULTY_ADMIN`, `PLATFORM_ADMIN` |
| **Request** | `{ rejection_reason: string }` |
| **Description** | PENDING_APPROVAL → DRAFT. Notifies Club Admin. |
| **Response 200** | `{ state: "DRAFT" }` |

### `POST /events/:id/lock`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `CLUB_ADMIN`, `FACULTY_MENTOR`, `FACULTY_ADMIN`, `PLATFORM_ADMIN` |
| **Response 200** | `{ is_locked: true }` |

### `POST /events/:id/unlock`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `CLUB_ADMIN`, `FACULTY_MENTOR`, `FACULTY_ADMIN`, `PLATFORM_ADMIN` |
| **Response 200** | `{ is_locked: false }` |

### `GET /events/:id/sessions`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated |
| **Response 200** | List of attendance sessions |

### `POST /events/:id/sessions`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `CLUB_ADMIN`, `CORE_MEMBER` |
| **Request** | `{ title, start_time, end_time, open_at, close_at, geofence_radius? }` |
| **Response 201** | `{ "id": "string", "title": "string" }` |

---


### `PATCH /events/:id/sessions/:sessionId`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `CLUB_ADMIN`, `CORE_MEMBER` |
| **Request** | `{ "title"?: "string", "start_time"?: "string", "end_time"?: "string", "open_at"?: "string", "close_at"?: "string", "geofence_radius"?: number }` |
| **Response 200** | `{ "id": "string", "title": "string", "open_at": "string", "close_at": "string" }` |

## Registration Endpoints

### `POST /events/:id/register`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated student |
| **Description** | Calls `register_event` RPC with `SELECT FOR UPDATE` |
| **Response 201** | `{ registration_id, status: "REGISTERED" | "WAITLISTED" }` |
| **Response 409** | Already registered |
| **Response 422** | Event locked, not PUBLISHED, or registration closed |

### `DELETE /events/:id/register`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Own registration |
| **Description** | Cancel registration. Triggers waitlist promotion. |
| **Response 204** | `{}` |

### `POST /events/:id/teams`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated (team-based events only) |
| **Request** | `{ team_name: string }` |
| **Response 201** | `{ team_id, name, leader_id }` |

### `POST /teams/:id/join`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated |
| **Response 200** | `{ "success": true }` |
| **Response 422** | Team full (per event metadata team_size_max) |

### `DELETE /teams/:id/leave`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Team member (self) |
| **Response 204** | `{}` |


### `PATCH /teams/:id`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Team Leader |
| **Request** | `{ "name"?: "string", "leader_id"?: "string" }` |
| **Response 200** | `{ "id": "string", "name": "string", "leader_id": "string" }` |

### `DELETE /teams/:id/members/:userId`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Team Leader |
| **Response 204** | `{}` |

### `GET /events/:id/registrations`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `CLUB_ADMIN`, `CORE_MEMBER`, Faculty, Platform Admin |
| **Query** | `cursor` (string), `limit` (number), `filter_status` |
| **Response 200** | Paginated registrations with user info |

### `GET /users/me/registrations`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated |
| **Response 200** | Own registrations with event info |

---

## Attendance Endpoints

### `POST /attendance/generate-qr`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `CLUB_ADMIN`, `CORE_MEMBER` (event organizer) |
| **Request** | `{ session_id }` |
| **Response 200** | `{ qr_payload, expires_at }` |
| **Rate Limit** | 10/min per user |

### `POST /attendance/mark`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated (must be registered) |
| **Request** | `{ session_id, totp_token, latitude, longitude, device_id, device_os, gps_accuracy, mock_location_detected, app_version }` |
| **Description** | Validates TOTP, geofence, device collision. Writes attendance via RPC. Idempotent: Returns existing record with 200 OK if already marked. |
| **Response 201** | `{ attendance_id, status: "PRESENT", points_awarded, flagged }` |
| **Response 422** | TOTP expired, outside geofence, event locked, not registered |
| **Rate Limit** | 5/min per user |

### `POST /attendance/sync-offline`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `CLUB_ADMIN`, `CORE_MEMBER` (organizer only) |
| **Request** | `{ records: [{ user_id, session_id, scanned_token, scan_timestamp, device_id, gps_lat, gps_lng, offline_seq }] }` |
| **Description** | Batch offline attendance submission. Geofence validated. TOTP NOT re-validated for offline. Entire batch audited. |
| **Response 200** | `{ "processed": number, "skipped": number, "errors": [{ "offline_seq": number, "error_code": "string", "message": "string" }] }` |

### `GET /events/:id/attendance`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `CLUB_ADMIN`, `CORE_MEMBER`, Faculty, Platform Admin |
| **Query** | `cursor` (string), `limit` (number), `filter_session_id`, `filter_status`, `filter_flagged` |
| **Response 200** | Paginated attendance records with user info |

### `GET /users/me/attendance`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated |
| **Response 200** | Own attendance history |

### `POST /attendance/disputes`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated (own attendance only) |
| **Request** | `{ session_id, reason, evidence_urls?: string[] }` |
| **Response 201** | `{ dispute_id, status: "PENDING", dispute_window_expires_at }` |
| **Response 422** | Dispute window expired (>24h after event end) |

### `GET /attendance/disputes`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `CLUB_ADMIN`, Faculty, Platform Admin |
| **Query** | `cursor` (string), `limit` (number), `filter_status`, `filter_event_id` |
| **Response 200** | Paginated disputes |

### `PATCH /attendance/disputes/:id`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `CLUB_ADMIN`, `FACULTY_MENTOR`, `FACULTY_ADMIN`, `PLATFORM_ADMIN` |
| **Request** | `{ resolution: "APPROVED" | "REJECTED", review_notes?: string }` |
| **Response 200** | `{ "id": "string", "status": "string" }` |

---

## Leaderboard Endpoints

### `GET /leaderboard/students`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated |
| **Query** | `cursor` (string), `limit` (number), `filter_club_id` |
| **Response 200** | Paginated student leaderboard (from `student_leaderboard_mv`) |

### `GET /leaderboard/clubs`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated |
| **Query** | `cursor` (string), `limit` (number) |
| **Response 200** | Paginated club leaderboard (from `club_leaderboard_mv`) |

### `POST /admin/leaderboard/recalculate`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `PLATFORM_ADMIN` |
| **Description** | Triggers `REFRESH MATERIALIZED VIEW CONCURRENTLY` for both MVs |
| **Response 200** | `{ refreshed_at }` |

---

## Notification Endpoints

### `GET /notifications`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated |
| **Query** | `cursor` (string), `limit` (number), `filter_read` (true/false) |
| **Response 200** | Paginated personal notifications |

### `PATCH /notifications/:id/read`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated (own notifications) |
| **Response 200** | `{ read_at }` |

### `PATCH /notifications/read-all`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated |
| **Response 204** | `{}` |

### `GET /notifications/preferences`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated |
| **Response 200** | `{ push_enabled, event_reminders, club_announcements, attendance_alerts }` |

### `PATCH /notifications/preferences`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated |
| **Request** | `{ "push_enabled"?: boolean, "event_reminders"?: boolean, "club_announcements"?: boolean, "attendance_alerts"?: boolean }` |
| **Response 200** | `{ "push_enabled": boolean }` |

---

## Admin Endpoints

### `GET /admin/audit-logs`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `PLATFORM_ADMIN`, `FACULTY_ADMIN` |
| **Query** | `cursor` (string), `limit` (number), `filter_entity_type`, `filter_entity_id`, `filter_actor_id`, `filter_action` |
| **Response 200** | Paginated audit logs |

### `POST /admin/users/:id/revoke-sessions`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `PLATFORM_ADMIN` |
| **Description** | Deletes all refresh_tokens for user (force logout) |
| **Response 204** | `{}` |

### `PATCH /admin/users/:id/role`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `PLATFORM_ADMIN` |
| **Request** | `{ global_role: GlobalRole }` |
| **Response 200** | `{ "id": "string", "global_role": "string" }` |

### `POST /admin/clubs/:id/force-transfer`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `PLATFORM_ADMIN` |
| **Request** | `{ new_admin_id }` |
| **Response 200** | `{ "success": true }` |

### `POST /admin/points/adjust`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `PLATFORM_ADMIN` |
| **Request** | `{ user_id, points, reason }` |
| **Response 201** | `{ "id": "string", "points": number }` |

---

## Leadership Transfer Endpoints

### `POST /clubs/:id/leadership/transfer`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `CLUB_ADMIN` (current admin of the club) |
| **Request** | `{ successor_id }` |
| **Response 201** | `{ handover_request_id, status: "PENDING" }` |

### `POST /leadership/:id/approve`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `FACULTY_MENTOR` (of club), `FACULTY_ADMIN` (fallback), `PLATFORM_ADMIN` |
| **Response 200** | `{ "success": true }` |

### `POST /leadership/:id/reject`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `FACULTY_MENTOR`, `FACULTY_ADMIN` (fallback), `PLATFORM_ADMIN` |
| **Request** | `{ review_notes }` |
| **Response 200** | `{ "success": true }` |

---

## Participation & Results Endpoints

### `PATCH /events/:id/registrations/:regId/role`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `CLUB_ADMIN`, `FACULTY_MENTOR`, `FACULTY_ADMIN`, `PLATFORM_ADMIN` |
| **Request** | `{ participation_role: ParticipationRole }` |
| **Response 200** | `{ "id": "string", "participation_role": "string" }` |

### `POST /events/:id/results`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | `CLUB_ADMIN`, `FACULTY_MENTOR`, `FACULTY_ADMIN`, `PLATFORM_ADMIN` |
| **Request** | `{ user_id, result_type: CompetitionResult }` |
| **Response 201** | `{ "id": "string", "result_type": "string" }` |

---


### `GET /events/:id/results`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated |
| **Response 200** | `{ "data": [{ "id": "string", "user_id": "string", "result_type": "string" }] }` |

## Search Endpoint

### `GET /search`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT |
| **Roles** | Any authenticated |
| **Query** | `q` (search term), `type` (events, clubs, users — users restricted to admin) |
| **Response 200** | `{ "data": [{ "id": "string", "_type": "EVENT" | "CLUB" | "USER", "name": "string" }], "pagination": { "next_cursor": "string", "has_more": boolean } }` |

---

## SSE Endpoints

### `GET /events/:id/live`
| Field | Value |
|---|---|
| **Auth** | Bearer JWT (query param `token` for SSE) |
| **Roles** | Any authenticated |
| **Description** | Server-Sent Events stream |
| **Events Emitted** | `attendance_count` (`{ count: number }`), `registration_count` (`{ count: number }`), `waitlist_update` (`{ user_id: string, status: string }`), `session_opened` (`{ session_id: string }`), `session_closed` (`{ session_id: string }`), `lock_status` (`{ is_locked: boolean }`), `heartbeat` (`{ timestamp: string }`) |
| **Headers** | `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `Connection: keep-alive` |
| **Reconnection** | Client sends `Last-Event-ID` header; server resumes from that point |
| **Rate Limit** | Max 3 concurrent SSE connections per user |
