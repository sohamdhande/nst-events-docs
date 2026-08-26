# Table Policy Matrix

Comprehensive breakdown of RLS policies per table. These policies are the **secondary** defense-in-depth layer. Express RBAC middleware is the primary enforcement layer for all operations.

Note: RLS policies use `current_user_id()` — a PostgreSQL helper function defined as `nullif(current_setting('app.user_id', true), '')::uuid`. This value is set per-transaction by the `withUserContext` Prisma wrapper in the Express backend (see `docs/security/01-rls-architecture.md`).

| Table | Read (SELECT) | Insert | Update | Delete | Notes |
|---|---|---|---|---|---|
| `users` | All Auth Users (Public fields) | Express OAuth callback creates user record | Self | None | `global_role` column is the source of truth for platform-wide roles. **Critical Note**: RLS operates at the row level, meaning it cannot block a user from updating their own `global_role` via a bypassed Express route. Therefore, a `BEFORE UPDATE` trigger on `users` is MANDATORY to enforce that only `PLATFORM_ADMIN` can modify `global_role`. |
| `clubs` | All Auth Users | Platform Admin | Platform Admin | None | Soft delete only |
| `club_memberships` | All Auth Users | Admin/Faculty | Admin/Faculty | Admin/Faculty | Club-scoped roles (MEMBER, CORE_MEMBER, CLUB_ADMIN, FACULTY_MENTOR) |
| `events` | All (If Published) | Club Admin, Core Member, Faculty Mentor, Faculty Admin, Platform Admin | Admin / Faculty | Club Admin | Drafts visible to organizers. **Approval workflow is represented by `events.state` transitions (DRAFT→PENDING_APPROVAL→PUBLISHED), not a separate table. Every state change is captured in `audit_logs`.** |
| `event_clubs` | All Auth Users | Club Admin | Club Admin | Club Admin | |
| `push_tokens` | Self (Express auth only) | Express (on login/token registration) | Express | Express (on DeviceNotRegistered) | Never client-readable directly |
| `teams` | All Auth Users | Authenticated | Team Leader | Team Leader | |
| `event_registrations`| Self & Organizers | Self (via Express RPC handler) | None | Self | RPC handles capacity |
| `attendance_sessions`| All Auth Users | Organizers | Organizers | None | |
| `attendance_records`| Self & Organizers | Express RPC only | Platform Admin | Platform Admin | Faculty cannot update |
| `notifications` | Self | System Queue (via nst-worker) | Self (mark read) | Self | |
| `notification_preferences` | Self | Self | Self | None | Students can only read and update their own preferences |
| `audit_logs` | Platform Admin | Triggers | None | None | Append only |
| `refresh_tokens` | None (Express only) | Express auth module | Express auth module | Express auth module | Not accessible to any client role |
| `attendance_disputes` | Self & Organizers | Self | Club Admin, Faculty | None | |
| `leadership_handover_requests`| Participants + Faculty + Platform Admin | Initiated_by | Faculty Mentor, Platform Admin | None | |
| `notification_jobs` | None (Client/Express) | All Auth Users (via RPC/Express) | None (Client/Express) | None (Client/Express) | **Service-Only Table.** Clients can `INSERT` (enqueue) jobs. Only the `nst-worker` service role can `SELECT`, `UPDATE`, or `DELETE`. SECURITY DEFINER RPCs bypass RLS to enqueue jobs. |

> **Storage objects**: General file uploads are deferred to post-V1 (ADR-008), though Club Branding (Banners) is an approved pilot. No storage object policies are required in the DB layer for V1 since Express handles storage authorization.

## `event_results`
* **SELECT**: Public (all authenticated users can view winners).
* **INSERT/UPDATE**: `Club Admin`, `Faculty Mentor`, `Faculty Admin`, `Platform Admin`. (Students explicitly blocked).
* **DELETE**: `Platform Admin` only.

## `event_registrations` (Update for `participation_role`)
* **UPDATE (`participation_role`)**: Restricted to `Club Admin`, `Faculty Mentor`, `Faculty Admin`, `Platform Admin`. Students cannot modify their own roles.

