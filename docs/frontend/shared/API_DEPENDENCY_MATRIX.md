# API Dependency Matrix

| SCREEN | PLATFORM | API | METHOD | PATH | PURPOSE | CONSUMED FIELDS | CACHE KEY | MUTATION | ERRORS | STATUS |
|---|---|---|---|---|---|---|---|---|---|---|
| Event Detail | Web/Mobile | `/v1/events/:id` | GET | `/v1/events/:id` | Fetch Event Details | `title, description, startTime, endTime, locationName` | `['event', id]` | No | 404, 500 | PLANNED |
| Event Detail | Web/Mobile | `/v1/events/:id/my-registration` | GET | `/v1/events/:id/my-registration` | Participant status | `status, teamId` | `['event', id, 'registration']` | No | 401 | PLANNED |
| Registration | Mobile | `/v1/events/:id/register` | POST | `/v1/events/:id/register` | Register for event | N/A (Status 201) | N/A | Yes | 400, 401, 409 | PLANNED |
| Registration | Mobile | `/v1/events/:id/teams` | POST | `/v1/events/:id/teams` | Create Team | `team_name` | N/A | Yes | 400, 401 | PLANNED |
| Teams | Web/Mobile | `/v1/events/:id/teams` | GET | `/v1/events/:id/teams` | List Teams | `id, name, leader_id` | `['event', id, 'teams']` | No | 401, 500 | PLANNED |
| Home Feed | Mobile | `/v1/events` | GET | `/v1/events` | Compose Feed Data | `title, state, isLocked, startTime` | `['events', 'list']` | No | 401, 500 | PLANNED |
| Home Feed | Mobile | `/v1/notifications` | GET | `/v1/notifications` | Unread Badge | `length` | `['notifications', 'list']` | No | 401, 500 | PLANNED |
| Waitlist (Organizer) | Web | `/v1/events/:id/registrations` | GET | `/v1/events/:id/registrations?filter_status=WAITLISTED` | Organizer Waitlist View | `id, userId, registrationStatus, registeredAt` | `['registrations', id, 'WAITLISTED']` | No | 401, 403 | PLANNED |
| Club Detail | Web/Mobile | `/v1/clubs/:id` | GET | `/v1/clubs/:id` | Fetch Club | `id, name, description, banner_url, status` | `['club', id]` | No | 404, 500 | PLANNED |
| Attendance | Web/Mobile | `/v1/events/:id/attendance` | GET | `/v1/events/:id/attendance` | Fetch Sessions | `id, eventId, title, startTime, endTime, openAt, closeAt, geofenceRadius` | `['event', id, 'sessions']` | No | 401, 404 | PLANNED |
| Attendance Export | Web | `/v1/events/:id/attendance/export` | GET | `/v1/events/:id/attendance/export` | Download CSV | N/A (CSV blob) | N/A | No | 401, 500 | PLANNED |
| Leaderboard | Web/Mobile | `/v1/leaderboard/students` | GET | `/v1/leaderboard/students` | Fetch Leaderboard | `user_id, display_name, total_points, attendance_points (STUBBED 0), contribution_points (STUBBED 0), competition_points (STUBBED 0)` | `['leaderboard', 'students']` | No | 401, 500 | PLANNED |
| SSE | Web/Mobile | `/v1/events/:id/live` | GET | `/v1/events/:id/live` | Live Updates | `type`, `payload.count`, `payload.user_id`, `payload.status` (Limited to `registration_count`, `waitlist_update`, and `heartbeat` only) | N/A | No | 401 | PLANNED |
| Dashboard | Web | `/v1/dashboard/summary` | GET | `/v1/dashboard/summary` | Fetch Summary | `upcoming_events, pending_approvals, my_clubs` | `['dashboard', 'summary']` | No | 401, 500 | PLANNED |
| Clubs List | Web | `/v1/clubs` | GET | `/v1/clubs` | Fetch Clubs | `id, name, description, banner_url, status, event_count, member_count` | `['clubs', 'list']` | No | 401, 500 | PLANNED |
| Clubs Search | Web | `/v1/clubs/search` | GET | `/v1/clubs/search` | Search Clubs | `id, name, description, banner_url, status, event_count, member_count` | `['clubs', 'search']` | No | 401, 500 | PLANNED |
| Admin Audit Logs | Web | `/v1/admin/audit-logs` | GET | `/v1/admin/audit-logs` | View Platform Logs | `id, action, entity_type, entity_id, actor_id, created_at` | `['admin', 'audit-logs']` | No | 401, 403, 500 | PLANNED |
| Admin Users | Web | `/v1/admin/users` | GET | `/v1/admin/users?q=&cursor=&limit=` | View Users List | `data (id, email, fullName, globalRole), pagination.next_cursor` | `['admin', 'users']` | No | 401, 403, 500 | PLANNED |
| Admin Role Update | Web | `/v1/admin/users/:userId/role` | POST | `/v1/admin/users/:userId/role` | Escalate Privileges| `global_role` | N/A | Yes | 401, 403, 500 | PLANNED |
