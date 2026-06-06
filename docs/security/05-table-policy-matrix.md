# Table Policy Matrix

Comprehensive breakdown of RLS policies per table.

| Table | Read (SELECT) | Insert | Update | Delete | Notes |
|---|---|---|---|---|---|
| `users` | All Auth Users (Public fields) | Supabase Auth Trigger | Self | None | |
| `user_roles` | All Auth Users | Admin/Faculty | Admin/Faculty | Admin/Faculty | Maps global overrides |
| `clubs` | All Auth Users | Platform Admin | Platform Admin | None | Soft delete only |
| `club_memberships` | All Auth Users | Admin/Faculty | Admin/Faculty | Admin/Faculty | |
| `faculty_clubs`| Faculty & Admin | Platform Admin | Platform Admin | Platform Admin | |
| `events` | All (If Published) | Club Admin, Core Member, Faculty Mentor, Faculty Admin, Platform Admin | Admin / Faculty | Club Admin | Drafts visible to organizers |
| `event_clubs` | All Auth Users | Club Admin | Club Admin | Club Admin | |
| `event_approvals`| Organizers & Faculty| Faculty | Faculty | None | Immutable log |
| `teams` | All Auth Users | Authenticated | Team Leader | Team Leader | |
| `team_members` | All Auth Users | Team Leader | None | Team Leader / Self | |
| `event_registrations`| Self & Organizers | Self | None | Self | RPC handles capacity |
| `attendance_sessions`| All Auth Users | Organizers | Organizers | None | |
| `attendance_records`| Self & Organizers | RPC Only (Self) | Platform Admin | Platform Admin | Faculty cannot update |
| `notifications` | Self | System Queue | Self (mark read) | Self | |
| `notification_preferences` | Self | Self | Self | None | Students can only read and update their own preferences |
| `audit_logs` | Platform Admin | Triggers | None | None | Append only |
| `storage.objects`| Public for Media | Owner / Admin | Owner / Admin | Owner / Admin | Secure docs locked down |

## `event_results`
* **SELECT**: Public (all authenticated users can view winners).
* **INSERT/UPDATE**: `Club Admin`, `Faculty Mentor`, `Faculty Admin`, `Platform Admin`. (Students explicitly blocked).
* **DELETE**: `Platform Admin` only.

## `event_registrations` (Update for `participation_role`)
* **UPDATE (`participation_role`)**: Restricted to `Club Admin`, `Faculty Mentor`, `Faculty Admin`, `Platform Admin`. Students cannot modify their own roles.
