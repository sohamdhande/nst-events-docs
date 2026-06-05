# Multi-Role Handling

## Scenarios
* **Student + Club Admin**: Has student rights globally, but elevated rights ONLY for specific `club_id`s.
* **Student + Faculty Mentor**: Handled cleanly because permissions are club-scoped.
* **Student + Faculty Admin**: Faculty Admins have cross-club visibility.
* **Student + Platform Admin**: Platform Admin overrides all club-level checks in RLS policies via an `OR global_role = 'PLATFORM_ADMIN'` condition.

## Permission Resolution Rules
Permissions are additive. If a user holds multiple roles for a club, the highest privilege applies.

## Conflict Handling
In cases of conflict (e.g., a user is both trying to attend an event and approve it), the system enforces separation of duties where possible. For instance, Faculty cannot mark attendance, so the student role applies for check-in, but the faculty role applies for approval.
