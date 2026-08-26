# CLUB DETAIL / CLUB OPERATIONS CONTRACT AUDIT

## 1. Existing Club Endpoints

- `GET /clubs/search`: Search clubs by query.
- `GET /clubs`: List clubs with pagination and sorting.
- `GET /clubs/:id`: Retrieve details for a specific club.
- `POST /clubs`: Create a new club (PLATFORM_ADMIN only).
- `PATCH /clubs/:id`: Update club details (name, description, banner_url).
- `PATCH /clubs/:id/status`: Update club status (PLATFORM_ADMIN only).
- `POST /clubs/:id/members`: Add a member to a club.
- `PATCH /clubs/:id/members/:userId`: Update a member's role.
- `DELETE /clubs/:id/members/:userId`: Remove a member.

## 2. Club Detail Fields (`GET /clubs/:id`)

| Field            | Available | Authoritative | Notes                               |
| ---------------- | --------- | ------------- | ----------------------------------- |
| `id`           | Yes       | Yes           |                                     |
| `name`         | Yes       | Yes           |                                     |
| `description`  | Yes       | Yes           |                                     |
| `banner_url`   | Yes       | Yes           |                                     |
| `status`       | Yes       | Yes           |                                     |
| `event_count`  | Yes       | Yes           | Aggregate from`_count.eventClubs` |
| `members`      | Yes       | Yes           | Inline array. Not paginated.        |
| `created_at`   | No        | -             | BACKEND DATA GAP                    |
| `updated_at`   | No        | -             | BACKEND DATA GAP                    |
| `member_count` | Yes       | Yes           | Derived from`members.length`      |

## 3. Member Contract

- `GET /clubs/:id` returns members inline as: `{ user_id, role, full_name, avatar_url }`.
- There is **no pagination** on the members response in this endpoint.
- Total member count is- [x] Persistent Club Header

- [X] Tabbed Club Workspace
- [X] Overview
- [X] Members Workspace
- [X] Events Navigation
- [X] Administration Workspace
- [X] Query-parameter tab state queried via `GET /v1/events?filter_club_id=:clubId`.

- The endpoint is paginated (`limit`, `cursor`).
- Sorting is available by `start_time` or `created_at`.
- There is **no server-side time filtering** (e.g., `upcoming` or `past` flags, or date bounds).

## 5. Event Counts

- **Total Event Count**: AVAILABLE (`event_count` from `GET /clubs/:id`).
- **Upcoming Event Count**: BACKEND DATA GAP.
- **Past Event Count**: BACKEND DATA GAP.
- **Active Event Count**: BACKEND DATA GAP.

## 6. Member Counts

- **Total Member Count**: AVAILABLE (derived from `members.length`).

## 7. Club Roles

- `GLOBAL_ROLE` vs `CLUB_ROLE`: These are strictly separate. A user can have `global_role = STUDENT` and `club_role = CLUB_ADMIN`.
- Allowed Club Roles: `CLUB_ADMIN`, `MEMBER`, `FACULTY_MENTOR`, `CORE_MEMBER`.
- Multiple users can hold the `CLUB_ADMIN` role simultaneously.
- Role changes trigger a `ROLE_CHANGED` notification.

## 8. Permissions

| Action             | PLATFORM_ADMIN | FACULTY_ADMIN | FACULTY_MENTOR | CLUB_ADMIN | CORE_MEMBER | STUDENT |
| ------------------ | -------------- | ------------- | -------------- | ---------- | ----------- | ------- |
| View Club          | Yes            | Yes           | Yes            | Yes        | Yes         | Yes     |
| Edit Club          | Yes            | No            | No             | Yes        | No          | No      |
| Change Status      | Yes            | No            | No             | No         | No          | No      |
| Add Member         | Yes            | No            | No             | Yes        | No          | No      |
| Remove Member      | Yes            | No            | No             | Yes        | No          | No      |
| Change Member Role | Yes            | No            | Yes            | Yes        | No          | No      |
| Create Event       | Yes            | No            | No             | Yes        | Yes         | No      |

## 9. BOLA (Broken Object Level Authorization)

- BOLA is strictly protected by the `requireClubRole` middleware.
- A `CLUB_ADMIN` of Club A will receive a 403 Forbidden (or 404 if data isn't exposed) when attempting to modify Club B.

## 10. Club Actions

- Edit Club Details (`PATCH /clubs/:id`)
- Change Status (`PATCH /clubs/:id/status` - Platform Admin only)

## 11. Event Actions

- The Club Detail page should link directly to canonical event operations:
  - View Event (`/events/:eventId`)
  - Manage Event (`/events/:eventId/edit`)
  - Manage Registrations / Teams (`/events/:eventId/registrations`)

## 12. Attention Signals

- Are there Club-level aggregates for attention signals (e.g., pending approvals, events near capacity)?
- BACKEND DATA GAP — CLUB OPERATIONAL ATTENTION SIGNALS.

## 13. Audit Data

- Are there Club-scoped audit logs available to Club Admins?
- BACKEND DATA GAP — CLUB-SCOPED AUDIT ACTIVITY.

## 14. Routing Recommendation

- Recommended target route: `/clubs/:clubId`
- Given the current lack of paginated members or extensive nested data, a single detail page is sufficient. Avoid nested tabs (e.g., `/clubs/:id/members`) unless backend pagination is introduced for members in the future.

## 15. Data Gaps

- BACKEND DATA GAP — COMPLETE CLUB UPCOMING EVENT QUERY
- BACKEND DATA GAP — COMPLETE PAST EVENTS QUERY
- BACKEND DATA GAP — CLUB OPERATIONAL ATTENTION SIGNALS
- BACKEND DATA GAP — CLUB-SCOPED AUDIT ACTIVITY

## 16. Implementation Readiness

- **CLUB DETAIL ROUTE — NOT CURRENTLY IMPLEMENTED**.
- The Club header and Member management sections are ready to be implemented based on `GET /clubs/:id`.
- The Event sections (Upcoming/Past) are BLOCKED by missing server-side date filtering. Do not implement fake client-side filtering over incomplete paginated data.
