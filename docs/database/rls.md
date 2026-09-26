# Row Level Security (RLS) & Authorization Architecture

This document defines the Row Level Security (RLS) policies, role resolution helpers, transaction context mechanics, and defense-in-depth authorization model for PostgreSQL.

---

## 1. Multi-Layered Authorization Model

In NST-Events, **RLS is NOT the sole authorization boundary**—it operates as a mandatory, database-level defense-in-depth boundary beneath backend Express middleware:

```mermaid
graph TD
    Client[HTTP Client: Dashboard / Mobile] --> JWT[Express JWT Auth & Role Middleware]
    JWT --> RBAC[Express Route Guards & Service Layer]
    RBAC --> TX[withUserContext: SELECT set_config('app.user_id', ...)]
    TX --> RLS[PostgreSQL Row Level Security Engine]
    RLS --> RPC[SECURITY DEFINER Procedures: mark_attendance, etc.]
    RLS --> Tables[(PostgreSQL Tables: FORCE RLS)]
```

1. **Layer 1: Express Authentication & RBAC Middleware:**
   - Validates the Google OAuth JWT signature and checks `user.security_version`.
   - Performs route-level authorization guards (e.g. `requireGlobalRole('PLATFORM_ADMIN')` or club-level ownership verification).
2. **Layer 2: Transaction-Local Context:**
   - On each service call, `@nst/database` executes `withUserContext(userId, tx => ...)`.
   - Injects the authenticated user ID: `SELECT set_config('app.user_id', ${userId}, true)`.
   - The `true` parameter makes the variable **transaction-local**, preventing cross-request pollution across pooled connections.
3. **Layer 3: PostgreSQL RLS (FORCE ROW LEVEL SECURITY):**
   - Applies even to table owners when queries run under role `nst_app`.
   - Evaluates SQL expressions against `current_user_id()`, `current_user_global_role()`, and `has_club_role(...)`.
4. **Layer 4: Atomic Stored Procedures (SECURITY DEFINER RPCs):**
   - Complex workflows (check-in, team formation, waitlist processing) bypass standard table policies and execute within validated procedural security contexts.

---

## 2. PostgreSQL Roles

| PostgreSQL Role | Type | Attributes | Intended Use |
| :--- | :--- | :--- | :--- |
| **`postgres`** | Superuser | `SUPERUSER`, `BYPASSRLS`, `CREATEDB`, `CREATEROLE` | Migrations (`prisma migrate deploy`), cluster administration, backup/restore. |
| **`nst_app`** | Application Runtime | `NOSUPERUSER`, `NOBYPASSRLS`, `NOCREATEDB`, `NOCREATEROLE` | Used by `apps/api` for all standard user operations. Subject to full RLS enforcement. |
| **`nst_worker`** | Worker Runtime | `NOSUPERUSER`, `NOBYPASSRLS`, `NOCREATEDB`, `NOCREATEROLE` | Used by `apps/worker` for background push notifications. Dedicated narrow grants. |

---

## 3. RLS Helper Functions

To optimize policy performance and prevent recursion, PostgreSQL helper functions are defined:

### 3.1 `current_user_id()`
```sql
CREATE OR REPLACE FUNCTION current_user_id() RETURNS uuid AS $$
  SELECT nullif(current_setting('app.user_id', true), '')::uuid;
$$ LANGUAGE sql STABLE;
```
Returns the currently authenticated user's UUID from the transaction-local variable `app.user_id`, or `NULL` if unauthenticated.

### 3.2 `current_user_global_role()`
```sql
CREATE OR REPLACE FUNCTION current_user_global_role() RETURNS text
LANGUAGE sql SECURITY DEFINER AS $$
  SELECT global_role::text FROM users WHERE id = current_user_id();
$$;
```
Reads the caller's global role (`STUDENT`, `FACULTY_MENTOR`, `FACULTY_ADMIN`, `PLATFORM_ADMIN`) via a `SECURITY DEFINER` lookup.

### 3.3 `has_club_role(p_club_id uuid, p_user_id uuid, p_roles text[])`
```sql
CREATE OR REPLACE FUNCTION has_club_role(p_club_id uuid, p_user_id uuid, p_roles text[])
RETURNS boolean LANGUAGE sql SECURITY DEFINER SET search_path = public AS $$
  SELECT EXISTS (
    SELECT 1 FROM club_memberships
    WHERE club_id = p_club_id
      AND user_id = p_user_id
      AND role::text = ANY(p_roles)
      AND deleted_at IS NULL
  );
$$;
```

### 3.4 Recursion Breaker: `can_see_user_as_organizer(target_user_id uuid)`
Introduced in `20260811074226_secure_users_rls`:
Because `users` policies referenced `event_registrations`, and `event_registrations` policies referenced `users`, PostgreSQL queries suffered infinite recursion crashes. `can_see_user_as_organizer` is a `SECURITY DEFINER` function that isolates the lookup, breaking the cycle.

---

## 4. Comprehensive RLS Table Policy Matrix

| Table | RLS Enabled | FORCE RLS | Important Policies | Backend Enforcement | Notes |
| :--- | :---: | :---: | :--- | :--- | :--- |
| **`users`** | **YES** | **YES** | Users SELECT self. Organizers SELECT enrolled attendees via `can_see_user_as_organizer`. Platform admins SELECT all. Updates restricted to self or admin. | API OAuth login, profile routes | Non-sensitive public profile lookup handled via `public_profiles` view. |
| **`refresh_tokens`** | **YES** | **YES** | `user_id = current_user_id()` for SELECT, INSERT, UPDATE. | Auth service refresh handler | Ensures tokens cannot be read across user boundaries. |
| **`clubs`** | **YES** | **YES** | SELECT permitted for active clubs (`deleted_at IS NULL`). Mutations restricted to `PLATFORM_ADMIN` / `FACULTY_ADMIN`. | Clubs controller guards | Public read for active campus directory. |
| **`club_memberships`** | **YES** | **YES** | Members & club admins SELECT roster. Platform/faculty admins SELECT all. INSERT/UPDATE/DELETE restricted to club admins & platform admins. | Club admin permissions | Audited automatically via trigger. |
| **`events`** | **YES** | **YES** | Anyone SELECTs `PUBLISHED`. Club organizers SELECT `DRAFT` / `PENDING_APPROVAL`. INSERT/UPDATE restricted to authorized club roles. | Event authorization middleware | Public events are openly readable; drafts strictly scoped. |
| **`event_clubs`** | **YES** | **YES** | SELECT permitted to authenticated users. Mutations restricted to host club admins and platform admins. | Event creation/edit service | Handles multi-club collaboration. |
| **`teams`** | **YES** | **YES** | Team members and event organizers SELECT. Mutations restricted to team leader. | Team service guards | Normalized name uniqueness enforced at table level. |
| **`event_registrations`** | **YES** | **YES** | Users SELECT own registrations. Organizers SELECT registrations for their events. Platform admins SELECT all. | Registration controller | Prevents students seeing peers' registration details. |
| **`attendance_sessions`** | **YES** | **YES** | Attendees SELECT during active scan window. Organizers & admins SELECT all sessions for their events. | Attendance controller | Protects `qr_secret` from unauthorized exposure. |
| **`attendance_records`** | **YES** | **YES** | Students SELECT own records. Organizers SELECT attendees for their events. Manual marks restricted to admins. | Attendance verification guards | Most check-ins occur via `mark_attendance` RPC. |
| **`attendance_disputes`** | **YES** | **YES** | Students SELECT own disputes. Club admins and faculty mentors SELECT/review disputes for their events. | Dispute management routes | Resolution enforced via `resolve_attendance_dispute` RPC. |
| **`event_results`** | **YES** | **YES** | Public SELECT. INSERT/UPDATE restricted to event organizers and platform admins. | Competition result service | Published competition rankings. |
| **`notifications`** | **YES** | **YES** | `user_id = current_user_id()` for users. `nst_worker` has ALL policy for background delivery tracking. | In-app notification routes | Dual-role access: user read, worker update. |
| **`notification_preferences`** | **YES** | **YES** | `user_id = current_user_id()` for users. `nst_worker` has SELECT grant. | User settings routes | Worker reads preferences before sending push. |
| **`push_tokens`** | **YES** | **YES** | `user_id = current_user_id()` for users. `nst_worker` has SELECT grant. | Device registration routes | Worker reads tokens during batch processing. |
| **`notification_jobs`** | **YES** | **YES** | `nst_app` can INSERT jobs. `nst_worker` can SELECT, UPDATE, and DELETE. Users have NO access. | Internal producer/worker | Dedicated worker queue table. |
| **`announcements`** | **YES** | **YES** | Public SELECT. Club admins create for their clubs. Platform admins create global. | Announcement routes | Global vs club announcements. |
| **`leadership_handover_requests`** | **YES** | **YES** | Initiator, successor, faculty mentor, and platform admins SELECT and review. | Governance workflow routes | State machine transitions. |
| **`leaderboard_scores`** | **YES** | **YES** | Public SELECT. INSERT restricted to attendance/competition RPCs and admins. | Leaderboard service | Immutable points ledger. |
| **`audit_logs`** | **YES** | **YES** | `nst_app` has INSERT grant. SELECT restricted to platform admins and user's own actions. | Admin audit views | Immutable forensic log. |
| **`authorized_students`** | **YES** | **YES** | SELECT permitted to authenticated users. Mutations restricted to `PLATFORM_ADMIN`. | Admin directory routes | Whitelist table. |
| **`user_academic_profiles`** | **YES** | **YES** | Authenticated users SELECT all (enables team leaders to check invitee eligibility). Mutations restricted to admins. | Academic profile routes | Batch association for eligibility. |
| **`academic_programs`** | **NO** | **NO** | Reference catalog. `GRANT SELECT ON ALL TABLES IN SCHEMA public TO nst_app`. | Academic routes | Public reference catalog. |
| **`academic_batches`** | **NO** | **NO** | Reference catalog. `GRANT SELECT ON ALL TABLES IN SCHEMA public TO nst_app`. | Academic routes | Public reference catalog. |
| **`event_audience_batches`** | **NO** | **NO** | Reference mapping. `GRANT SELECT ON ALL TABLES IN SCHEMA public TO nst_app`. | Event creation routes | Event eligibility mapping. |

---

## 5. Security Definier Functions & Bypasses

Certain stored procedures run with `SECURITY DEFINER` privileges to perform atomic cross-table transactions that would otherwise trigger conflicting RLS constraints:

1. **`mark_attendance`:**
   - Validates scan, creates attendance record, locks the device, and inserts into `leaderboard_scores`.
   - Runs with fixed search path: `SET search_path TO 'public', 'pg_catalog'`.
   - Explicitly verifies `v_user_id := current_user_id(); IF v_user_id IS NULL THEN RAISE EXCEPTION 'UNAUTHORIZED';`.
2. **`sync_offline_attendance`:**
   - Processes batched offline check-ins.
   - Enforces `v_user_id := current_user_id();` ensuring an actor can only submit check-ins for themselves.
3. **`register_event` & `process_waitlist`:**
   - Modifies registration count on `events`, updates team waitlists, and inserts into `event_registrations`.
4. **`resolve_attendance_dispute`:**
   - Verifies caller has `CLUB_ADMIN` or `FACULTY_MENTOR` status on the event before updating the dispute and modifying `attendance_records` to `EXCUSED`.
