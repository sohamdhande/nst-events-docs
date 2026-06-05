# Table Policy Matrix

Comprehensive breakdown of RLS policies per table.

| Table | Read (SELECT) | Insert | Update | Delete | Notes |
|---|---|---|---|---|---|
| `users` | All Auth Users (Public fields) | Supabase Auth Trigger | Self | None | |
| `user_roles` | All Auth Users | Admin/Faculty | Admin/Faculty | Admin/Faculty | Maps global overrides |
| `clubs` | All Auth Users | Platform Admin | Platform Admin | None | Soft delete only |
| `club_members` | All Auth Users | Admin/Faculty | Admin/Faculty | Admin/Faculty | |
| `faculty_clubs`| Faculty & Admin | Platform Admin | Platform Admin | Platform Admin | |
| `events` | All (If Published) | Club Admin / Core | Admin / Faculty | Club Admin | Drafts visible to organizers |
| `event_clubs` | All Auth Users | Club Admin | Club Admin | Club Admin | |
| `event_approvals`| Organizers & Faculty| Faculty | Faculty | None | Immutable log |
| `teams` | All Auth Users | Authenticated | Team Leader | Team Leader | |
| `team_members` | All Auth Users | Team Leader | None | Team Leader / Self | |
| `event_registrations`| Self & Organizers | Self | None | Self | RPC handles capacity |
| `attendance_sessions`| All Auth Users | Organizers | Organizers | None | |
| `attendance_records`| Self & Organizers | RPC Only (Self) | Platform Admin | Platform Admin | Faculty cannot update |
| `notifications` | Self | System Queue | Self (mark read) | Self | |
| `leaderboard_points`| All Auth Users | Edge Functions | None | None | Append only |
| `audit_logs` | Platform Admin | Triggers | None | None | Append only |
| `storage.objects`| Public for Media | Owner / Admin | Owner / Admin | Owner / Admin | Secure docs locked down |\n