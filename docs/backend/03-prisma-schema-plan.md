# Prisma Schema Generation Plan

> **Status**: IMPLEMENTATION-READY | **Version**: 1.0 | **Date**: 2026-06-09
> **Authority**: Database Freeze V1, schema-architecture.md, table-catalog.md

---

## Prisma-Managed vs SQL-Managed Separation

Prisma manages table structures, enums, relations, and basic indexes. Everything else is SQL-managed via raw migration files.

| Layer | Managed By | Examples |
|---|---|---|
| **Tables, columns, enums, relations** | Prisma schema (`schema.prisma`) | All entity definitions |
| **Basic indexes, unique constraints** | Prisma schema (`@@index`, `@@unique`) | Foreign key indexes, composite uniques |
| **Temporal CHECK constraints** | SQL migration | `CHECK(start_time < end_time)`, `CHECK(open_at < close_at)` |
| **PostgreSQL extensions** | SQL migration | `CREATE EXTENSION postgis, pgmq, pg_cron` |
| **RLS policies** | SQL migration | All `CREATE POLICY` statements |
| **Helper functions** | SQL migration | `current_user_id()` |
| **Stored procedures (RPCs)** | SQL migration | `register_event()`, `mark_attendance()`, etc. |
| **Database triggers** | SQL migration | Audit triggers, soft-delete cascades, `updated_at` |
| **Views** | SQL migration | Soft-delete views with `security_invoker = true` |
| **Materialized views** | SQL migration | `club_leaderboard_mv`, `student_leaderboard_mv` |
| **Generated columns** | SQL migration | `search_vector tsvector GENERATED ALWAYS AS (...)` |
| **pg_cron jobs** | SQL migration | Leaderboard MV refresh, token cleanup |
| **pgmq queues** | SQL migration | `SELECT pgmq.create('notifications')` |
| **PostGIS column types** | SQL migration (override Prisma) | `GEOGRAPHY(Point, 4326)` via `ALTER COLUMN` |
| **Partial indexes** | SQL migration | `WHERE deleted_at IS NULL` indexes |

### Migration Workflow
1. Define tables and relations in `schema.prisma`
2. Run `npx prisma migrate dev --create-only` to generate base DDL
3. Append raw SQL to the generated migration file (or create separate SQL-only migrations) for extensions, RLS, triggers, views, RPCs, etc.
4. Run `npx prisma migrate dev` to apply

---

## Entity Definitions

### 1. `users`

| Attribute | Detail |
|---|---|
| **Purpose** | Stores authenticated user identities and global roles |
| **Primary Key** | `id UUID DEFAULT gen_random_uuid()` |
| **Key Columns** | `google_sub TEXT UNIQUE NOT NULL` (stable OIDC subject from Google `id_token`), `email TEXT UNIQUE NOT NULL`, `full_name TEXT NOT NULL`, `avatar_url TEXT NULL` (NULL in V1), `global_role global_role_enum NOT NULL DEFAULT 'STUDENT'` |
| **Timestamps** | `created_at TIMESTAMPTZ DEFAULT now()`, `updated_at TIMESTAMPTZ` (trigger-managed) |
| **Soft Delete** | `deleted_at TIMESTAMPTZ NULL` |
| **Constraints** | `CHECK (email LIKE '%@adypu.edu.in' OR email LIKE '%@newtonschool.co')` (SQL-managed) |
| **Indexes** | `email` UNIQUE B-Tree, `google_sub` UNIQUE B-Tree, `search_vector` GIN (SQL-managed) |
| **Relationships** | Has many: `club_memberships`, `event_registrations`, `attendance_records`, `notifications`, `refresh_tokens`, `audit_logs` |
| **Audit** | INSERT/UPDATE/DELETE triggers write to `audit_logs` |
| **RLS** | SELECT own row + public profile fields for all auth users; UPDATE own row only |

### 2. `refresh_tokens`

| Attribute | Detail |
|---|---|
| **Purpose** | Server-side session storage for opaque refresh tokens. Enables revocation. |
| **Primary Key** | `id UUID DEFAULT gen_random_uuid()` |
| **Key Columns** | `user_id UUID NOT NULL FK→users(ON DELETE CASCADE)`, `token_hash TEXT UNIQUE NOT NULL` (SHA-256), `family_id UUID NOT NULL` (Links rotated tokens), `expires_at TIMESTAMPTZ NOT NULL`, `revoked_at TIMESTAMPTZ NULL`, `user_agent TEXT NULL`, `ip_address TEXT NULL` |
| **Timestamps** | `created_at TIMESTAMPTZ DEFAULT now()` |
| **Soft Delete** | No — hard delete on cleanup |
| **Indexes** | `token_hash` UNIQUE B-Tree, `user_id` B-Tree, `expires_at` B-Tree |
| **Relationships** | Belongs to: `users` (CASCADE) |
| **Audit** | Not audited (high-churn table) |
| **RLS** | Not client-accessible. Managed entirely by Express auth module. |
| **Cleanup** | `pg_cron` job purges rows where `expires_at < now()` OR `revoked_at IS NOT NULL AND revoked_at < now() - interval '7 days'` |

### 3. `clubs`

| Attribute | Detail |
|---|---|
| **Purpose** | Represents campus organizations |
| **Primary Key** | `id UUID DEFAULT gen_random_uuid()` |
| **Key Columns** | `name TEXT UNIQUE NOT NULL`, `description TEXT`, `banner_url TEXT NULL` (NULL in V1), `status club_status_enum NOT NULL DEFAULT 'ACTIVE'` |
| **Timestamps** | `created_at`, `updated_at` |
| **Soft Delete** | `deleted_at TIMESTAMPTZ NULL` |
| **Indexes** | `name` UNIQUE B-Tree, `search_vector` GIN (SQL) |
| **Relationships** | Has many: `club_memberships`, `event_clubs`, `announcements` |
| **Audit** | DELETE triggers audit log |
| **RLS** | SELECT: all auth users. UPDATE: Platform Admin. |
| **Cascade** | Soft-deleting a club cascades to `club_memberships`, `event_clubs` (trigger-managed) |
| **Status Transitions** | `ACTIVE` (default) → `INACTIVE` (Platform Admin) → `ACTIVE` (Platform Admin reversal) or `DISSOLVED` (Platform Admin, irreversible). A `DISSOLVED` club cannot transition to any other state. |

### 4. `club_memberships`

| Attribute | Detail |
|---|---|
| **Purpose** | Maps users to clubs with contextual RBAC roles |
| **Primary Key** | `id UUID` |
| **Key Columns** | `user_id UUID FK→users`, `club_id UUID FK→clubs`, `role club_role_enum NOT NULL`, `joined_at TIMESTAMPTZ DEFAULT now()` |
| **Timestamps** | `created_at`, `updated_at` |
| **Soft Delete** | `deleted_at TIMESTAMPTZ NULL` |
| **Constraints** | `UNIQUE(club_id, user_id) WHERE deleted_at IS NULL` (partial unique, SQL) |
| **Indexes** | Composite `(club_id, user_id)`, `user_id` B-Tree |
| **Relationships** | Belongs to: `users`, `clubs` |
| **Audit** | Role changes trigger audit log |
| **RLS** | SELECT: all auth. INSERT/UPDATE/DELETE: Club Admin, Faculty. |

### 5. `events`

| Attribute | Detail |
|---|---|
| **Purpose** | Core generic event model |
| **Primary Key** | `id UUID` |
| **Key Columns** | `title TEXT NOT NULL`, `description TEXT`, `start_time TIMESTAMPTZ NOT NULL`, `end_time TIMESTAMPTZ NOT NULL`, `location_name TEXT`, `location_geofence GEOGRAPHY(Point,4326)` (SQL-managed type), `event_type event_type_enum NOT NULL`, `state event_state_enum NOT NULL DEFAULT 'DRAFT'`, `visibility event_visibility_enum NOT NULL DEFAULT 'PUBLIC'`, `registration_type registration_type_enum NOT NULL DEFAULT 'INDIVIDUAL'`, `metadata JSONB DEFAULT '{}'`, `attendance_type attendance_type_enum NOT NULL DEFAULT 'SINGLE'`, `is_locked BOOLEAN NOT NULL DEFAULT false`, `max_capacity INTEGER NULL`, `registration_count INTEGER NOT NULL DEFAULT 0`, `created_by UUID FK→users NOT NULL` |
| **Timestamps** | `created_at`, `updated_at` |
| **Soft Delete** | `deleted_at TIMESTAMPTZ NULL` |
| **Indexes** | `start_time` B-Tree, `metadata` GIN, `location_geofence` GiST (SQL), `state` B-Tree, `search_vector` GIN (SQL) |
| **Relationships** | Has many: `event_clubs`, `attendance_sessions`, `event_registrations`, `teams`. Belongs to: `users` (created_by) |
| **Audit** | State transitions trigger audit log |
| **RLS** | SELECT: all if PUBLISHED+PUBLIC, organizers if DRAFT/PENDING. INSERT: Club Admin, Core Member. UPDATE: Club Admin (DRAFT), Faculty (state transitions). DELETE: soft-delete by Club Admin. |
| **Cascade** | Soft-delete cascades to `event_registrations`, `attendance_sessions`, `teams` (trigger) |

### 6. `event_clubs`

| Attribute | Detail |
|---|---|
| **Purpose** | M:N mapping for multi-club collaborative events |
| **Key Columns** | `event_id UUID FK→events`, `club_id UUID FK→clubs`, `is_primary BOOLEAN NOT NULL DEFAULT false` |
| **Constraints** | `PRIMARY KEY (event_id, club_id)` |
| **Relationships** | Belongs to: `events`, `clubs` |
| **RLS** | SELECT: all auth. INSERT/UPDATE/DELETE: Club Admin of the club. |

### 7. `attendance_sessions`

| Attribute | Detail |
|---|---|
| **Purpose** | Granular time blocks requiring check-in (supports multi-day events) |
| **Primary Key** | `id UUID` |
| **Key Columns** | `event_id UUID FK→events NOT NULL`, `title TEXT NOT NULL`, `start_time TIMESTAMPTZ NOT NULL`, `end_time TIMESTAMPTZ NOT NULL`, `open_at TIMESTAMPTZ NOT NULL`, `close_at TIMESTAMPTZ NOT NULL`, `geofence_radius FLOAT NOT NULL DEFAULT 50` |
| **Timestamps** | `created_at`, `updated_at` |
| **Soft Delete** | `deleted_at TIMESTAMPTZ NULL` |
| **Indexes** | `event_id` B-Tree |
| **Relationships** | Belongs to: `events`. Has many: `attendance_records` |
| **RLS** | Same read policies as parent event. INSERT/UPDATE: organizers. |

### 8. `attendance_records`

| Attribute | Detail |
|---|---|
| **Purpose** | Verified proof of physical presence |
| **Primary Key** | `id UUID` |
| **Key Columns** | `session_id UUID FK→attendance_sessions NOT NULL`, `user_id UUID FK→users NOT NULL`, `marked_by UUID FK→users`, `marked_at TIMESTAMPTZ NOT NULL DEFAULT now()`, `method attendance_method_enum NOT NULL`, `status attendance_status_enum NOT NULL`, `audit_metadata JSONB NOT NULL DEFAULT '{}'` |
| **Constraints** | `UNIQUE(session_id, user_id)` |
| **Indexes** | Composite `(session_id, user_id)` UNIQUE |
| **Relationships** | Belongs to: `attendance_sessions`, `users` |
| **Audit** | All writes trigger audit log. `audit_metadata` stores: `device_id`, `device_os`, `gps_accuracy`, `mock_location_detected`, `app_version`, `flagged`, `flag_reason` |
| **RLS** | SELECT: self + organizers. INSERT: via RPC only. UPDATE/DELETE: Platform Admin only. |

### 9. `event_registrations`

| Attribute | Detail |
|---|---|
| **Purpose** | Tracks intent to attend (individual or team). Replaces `team_members` entirely. |
| **Primary Key** | `id UUID` |
| **Key Columns** | `event_id UUID FK→events NOT NULL`, `user_id UUID FK→users NOT NULL`, `team_id UUID NULL`, `registration_status registration_status_enum NOT NULL DEFAULT 'REGISTERED'`, `participation_role participation_role_enum NOT NULL DEFAULT 'ATTENDEE'`, `registered_at TIMESTAMPTZ DEFAULT now()` |
| **Soft Delete** | `deleted_at TIMESTAMPTZ NULL` |
| **Constraints** | `UNIQUE(event_id, user_id) WHERE deleted_at IS NULL` (partial, SQL). `FOREIGN KEY (team_id, event_id) REFERENCES teams(id, event_id)` (Composite FK to prevent cross-event team registration). |
| **Indexes** | Composite `(event_id, user_id)`, `user_id` B-Tree, `(team_id, event_id)` B-Tree |
| **Relationships** | Belongs to: `events`, `users`, `teams` (nullable) |
| **RLS** | SELECT: self + organizers. INSERT: self (via RPC). DELETE: self. UPDATE(participation_role): Club Admin, Faculty only. |

### 10. `teams`

| Attribute | Detail |
|---|---|
| **Purpose** | Team-based hackathon/competition grouping |
| **Primary Key** | `id UUID` |
| **Key Columns** | `event_id UUID FK→events NOT NULL`, `name TEXT NOT NULL`, `leader_id UUID FK→users NOT NULL` |
| **Timestamps** | `created_at` |
| **Soft Delete** | `deleted_at TIMESTAMPTZ NULL` |
| **Constraints** | `UNIQUE(id, event_id)` (Allows composite FK from registrations), `UNIQUE(event_id, name)` (Team names are unique within an event) |
| **Relationships** | Belongs to: `events`, `users` (leader). Has many: `event_registrations` |
| **RLS** | SELECT: all auth. INSERT: authenticated. UPDATE: team leader. DELETE: team leader. |



### 12. `event_results`

| Attribute | Detail |
|---|---|
| **Purpose** | Competition outcomes for leaderboard scoring |
| **Primary Key** | `id UUID` |
| **Key Columns** | `event_id UUID FK→events`, `user_id UUID FK→users`, `result_type competition_result_enum NOT NULL`, `created_by UUID FK→users NOT NULL` |
| **Constraints** | `UNIQUE(event_id, user_id)` — User appears at most once per event result set. |
| **Timestamps** | `created_at` |
| **RLS** | SELECT: all auth. INSERT/UPDATE: Club Admin, Faculty. DELETE: Platform Admin. |

### 13. `notifications`

| Attribute | Detail |
|---|---|
| **Purpose** | Persistent in-app notification inbox |
| **Primary Key** | `id UUID` |
| **Key Columns** | `user_id UUID FK→users NOT NULL`, `title TEXT NOT NULL`, `body TEXT NOT NULL`, `type TEXT NOT NULL`, `read_at TIMESTAMPTZ NULL`, `metadata JSONB DEFAULT '{}'`, `delivered_at TIMESTAMPTZ NULL`, `delivery_failed_at TIMESTAMPTZ NULL`, `delivery_error TEXT NULL` |
| **Timestamps** | `created_at` |
| **Indexes** | `user_id` B-Tree, `(user_id, created_at DESC)` composite (Optimizes inbox query), `(user_id, read_at)` composite |
| **Relationships** | Belongs to: `users` |
| **RLS** | SELECT: self. UPDATE(read_at): self. INSERT: system/worker only. |

### 14. `notification_preferences`

| Attribute | Detail |
|---|---|
| **Purpose** | Per-user notification opt-out settings |
| **Primary Key** | `id UUID` |
| **Key Columns** | `user_id UUID FK→users UNIQUE NOT NULL`, `push_enabled BOOLEAN DEFAULT true`, `event_reminders BOOLEAN DEFAULT true`, `club_announcements BOOLEAN DEFAULT true`, `attendance_alerts BOOLEAN DEFAULT true` |
| **RLS** | SELECT/UPDATE: self only. |

### 15. `push_tokens`

| Attribute | Detail |
|---|---|
| **Purpose** | Stores Expo push tokens per device so `nst-worker` can deliver push notifications. One user may have multiple active tokens (phone + tablet). |
| **Primary Key** | `id UUID DEFAULT gen_random_uuid()` |
| **Key Columns** | `user_id UUID FK→users NOT NULL`, `device_id TEXT NOT NULL` (client-generated device identifier, same value used in `attendance_records.audit_metadata`), `expo_token TEXT NOT NULL` (Expo push token of the form `ExponentPushToken[...]`), `platform TEXT NOT NULL` (values: `'ios'`, `'android'`), `last_seen_at TIMESTAMPTZ NOT NULL DEFAULT now()` (updated on each authenticated request to detect stale tokens) |
| **Timestamps** | `created_at TIMESTAMPTZ DEFAULT now()`, `updated_at TIMESTAMPTZ` (trigger-managed) |
| **Constraints** | `UNIQUE(device_id)` — One device = one active push token. On token refresh, UPSERT (`ON CONFLICT(device_id) DO UPDATE`) the existing row to assign it to the current user. |
| **Indexes** | `user_id` B-Tree, `device_id` UNIQUE B-Tree |
| **Relationships** | Belongs to: `users` (ON DELETE CASCADE) |
| **Soft Delete** | No — hard delete when `DeviceNotRegistered` returned by Expo. |
| **RLS** | Not client-accessible. Managed entirely by Express (upsert) and nst-worker (delete). No `SELECT` policy granted to any client role. |
| **Lifecycle** | **Upsert on login**: OAuth callback upserts using `(device_id)`, setting `user_id`, `expo_token` and `last_seen_at`. **Cleanup on DeviceNotRegistered**: nst-worker hard-deletes the row immediately after Expo returns this error code — no retry. **Stale cleanup**: `pg_cron` job hard-deletes rows where `last_seen_at < now() - interval '90 days'`. |
| **Audit** | Not audited (high-churn operational table). |

### 16. `announcements`


| Attribute | Detail |
|---|---|
| **Purpose** | Global or club-specific broadcasts |
| **Primary Key** | `id UUID` |
| **Key Columns** | `club_id UUID FK→clubs NULL`, `title TEXT NOT NULL`, `content TEXT NOT NULL`, `created_by UUID FK→users NOT NULL` |
| **Timestamps** | `created_at` |
| **RLS** | SELECT: all auth. INSERT: Club Admin (club-scoped), Platform Admin (global). |

### 16. `attendance_disputes`

| Attribute | Detail |
|---|---|
| **Purpose** | Student-submitted attendance correction requests |
| **Primary Key** | `id UUID` |
| **Key Columns** | `attendance_record_id UUID FK→attendance_records NULL`, `session_id UUID FK→attendance_sessions NOT NULL`, `event_id UUID FK→events NOT NULL`, `user_id UUID FK→users NOT NULL`, `reason TEXT NOT NULL`, `evidence_urls TEXT[] NULL`, `status dispute_status_enum NOT NULL DEFAULT 'PENDING'`, `dispute_window_expires_at TIMESTAMPTZ NOT NULL`, `submitted_at TIMESTAMPTZ DEFAULT now()`, `reviewed_at TIMESTAMPTZ NULL`, `reviewed_by UUID FK→users NULL`, `review_notes TEXT NULL` |
| **Constraints** | `UNIQUE(session_id, user_id)` — One dispute per student per session. |
| **Timestamps** | `created_at` |
| **RLS** | SELECT: self + organizers. INSERT: self. UPDATE: Club Admin, Faculty. |

### 17. `leadership_handover_requests`

| Attribute | Detail |
|---|---|
| **Purpose** | Stateful club leadership transfer workflow |
| **Primary Key** | `id UUID` |
| **Key Columns** | `club_id UUID FK→clubs NOT NULL`, `initiated_by UUID FK→users NOT NULL`, `successor_id UUID FK→users NOT NULL`, `faculty_mentor_id UUID FK→users NULL`, `status handover_status_enum NOT NULL DEFAULT 'PENDING'`, `initiated_at TIMESTAMPTZ DEFAULT now()`, `reviewed_at TIMESTAMPTZ NULL`, `review_notes TEXT NULL` |
| **Timestamps** | `created_at` |
| **Indexes** | `WHERE status = 'PENDING'` (Partial UNIQUE index on `club_id` to ensure only one pending request per club). |
| **RLS** | SELECT: participants + Faculty + Platform Admin. UPDATE: Faculty Mentor (or Faculty Admin fallback), Platform Admin. |

### 18. `leaderboard_scores`

| Attribute | Detail |
|---|---|
| **Purpose** | Immutable ledger for gamification points |
| **Primary Key** | `id UUID` |
| **Key Columns** | `user_id UUID FK→users NOT NULL`, `club_id UUID FK→clubs NULL`, `points INTEGER NOT NULL`, `reason TEXT NOT NULL`, `source_id UUID NOT NULL`, `created_at TIMESTAMPTZ DEFAULT now()` |
| **Indexes** | `user_id` B-Tree, `club_id` B-Tree, `created_at` BRIN |
| **RLS** | SELECT: all auth. INSERT: Express backend only (via RPCs after attendance/result validation). |

### 19. `audit_logs`

| Attribute | Detail |
|---|---|
| **Purpose** | Immutable ledger of destructive or high-impact actions |
| **Primary Key** | `id BIGSERIAL` (not UUID — strict chronological ordering + performance) |
| **Key Columns** | `actor_id UUID FK→users NULL` (populated by `current_user_id()`), `action TEXT NOT NULL`, `entity_type TEXT NOT NULL`, `entity_id UUID`, `previous_state JSONB`, `new_state JSONB`, `ip_address TEXT NULL` |
| **Timestamps** | `created_at TIMESTAMPTZ DEFAULT now()` |
| **Indexes** | `entity_id` B-Tree, `actor_id` B-Tree, `created_at` BRIN |
| **RLS** | SELECT: Platform Admin, Faculty Admin only. INSERT: triggers only (SECURITY DEFINER). UPDATE/DELETE: denied to all. |
| **Retention** | Indefinite in V1 |

---

## Enums (Prisma-Managed)

> Canonical enum definitions are maintained in `docs/database/04-enums.md`. This Prisma block must match that file exactly.

```prisma
enum GlobalRole {
  STUDENT
  FACULTY_ADMIN
  PLATFORM_ADMIN
}

enum ClubRole {
  MEMBER
  CORE_MEMBER
  CLUB_ADMIN
  FACULTY_MENTOR
}

enum ClubStatus {
  ACTIVE
  INACTIVE
  DISSOLVED
}

enum EventType {
  WORKSHOP
  SEMINAR
  COMPETITION
  MEETUP
  HACKATHON
  OTHER
}

enum EventState {
  DRAFT
  PENDING_APPROVAL
  PUBLISHED
  ARCHIVED
}

enum EventVisibility {
  PUBLIC
  PRIVATE
}

enum RegistrationType {
  INDIVIDUAL
  TEAM
}

enum RegistrationStatus {
  REGISTERED
  WAITLISTED
  CANCELLED
}

enum AttendanceStatus {
  PRESENT
  ABSENT
  // Written exclusively by resolve_attendance_dispute RPC when resolution = 'APPROVED'.
  // Semantics: organizer-confirmed presence after a successful dispute review.
  // Distinct from PRESENT (self-scanned) and ABSENT (no record).
  EXCUSED
}

enum AttendanceMethod {
  QR
  MANUAL
  SYSTEM
}

enum AttendanceType {
  SINGLE
  MULTI_SESSION
}

enum ParticipationRole {
  ATTENDEE
  VOLUNTEER
  ORGANIZER
  SPEAKER
  MENTOR
}

enum CompetitionResult {
  WINNER
  RUNNER_UP
  SECOND_RUNNER_UP
  TOP_10
  PARTICIPANT
}

enum DisputeStatus {
  PENDING
  APPROVED
  REJECTED
}

enum HandoverStatus {
  PENDING
  APPROVED
  REJECTED
}
```

---

## Infrastructure Strategies

### PostGIS Strategy
- **Column**: `events.location_geofence` typed as `GEOGRAPHY(Point, 4326)` via SQL migration (Prisma models it as `Unsupported("geography(Point,4326)")`)
- **Index**: GiST index on `location_geofence` (SQL migration)
- **Validation**: `ST_DWithin(events.location_geofence, ST_SetSRID(ST_MakePoint(lng, lat), 4326), geofence_radius)` in `mark_attendance` RPC
- **Prisma Access**: Read via `$queryRaw`; write via `$executeRaw` with parameterized ST functions

### PGMQ Strategy
- **Queue creation**: `SELECT pgmq.create('notifications')` in SQL migration
- **Enqueue**: RPCs enqueue within same transaction as the triggering action (e.g., `approve_event` enqueues notification)
- **Dequeue**: `nst-worker` polls via `SELECT pgmq.read('notifications', 30, 100)` (30s visibility timeout, batch of 100)
- **Delete**: `SELECT pgmq.delete('notifications', msg_id)` after successful delivery
- **DLQ**: After 3 failures, `SELECT pgmq.archive('notifications', msg_id)` moves to DLQ
- **Autovacuum**: Aggressive settings on pgmq schema tables to prevent bloat

### Materialized View Strategy
- **Views**: `club_leaderboard_mv`, `student_leaderboard_mv` — defined in SQL migration
- **Source**: Aggregated from `attendance_records`, `event_results`, `event_registrations`, `leaderboard_scores`
- **Refresh**: `pg_cron` job every 5 minutes: `REFRESH MATERIALIZED VIEW CONCURRENTLY`
- **Manual refresh**: `POST /admin/leaderboard/recalculate` (Platform Admin)
- **Never**: Row-level triggers for refresh (causes lock contention)
- **Prisma Access**: Read via `$queryRaw` (Prisma cannot model materialized views natively)

### Audit Trigger Strategy
- **Trigger type**: `AFTER INSERT OR UPDATE OR DELETE` on high-value tables (`events`, `club_memberships`, `attendance_records`, `event_registrations`)
- **Actor resolution**: `current_user_id()` reads `app.user_id` set by `withUserContext`
- **Security**: Triggers defined as `SECURITY DEFINER` to bypass RLS on `audit_logs` INSERT
- **Tracked actions**: Event state transitions, club creation/deletion, role assignments, attendance overrides, leaderboard adjustments
- **NULL actor**: Indicates a bug — all mutating queries must run inside `withUserContext`

### Partitioning Strategy
- **V1**: No partitioning. Single-partition tables are sufficient for 3000 students.
- **Future (Year 2+)**: Range partition `attendance_records` by `created_at` (monthly partitions). Range partition `audit_logs` by `created_at` (yearly partitions). Prisma supports partitioned tables transparently.

### Soft Delete Rules
- **Tables with soft delete**: `users`, `clubs`, `club_memberships`, `events`, `attendance_sessions`, `event_registrations`, `teams`
- **Tables without soft delete**: `refresh_tokens` (hard delete), `audit_logs` (immutable), `leaderboard_scores` (immutable), `notifications` (user-deletable), `event_results`, `attendance_records`
- **Views**: `active_events`, `active_clubs`, `active_memberships` — `WHERE deleted_at IS NULL`, `security_invoker = true`
- **Cascade**: PostgreSQL triggers (not application layer). Event → registrations, sessions, teams. Club → memberships, event_clubs.
- **Partial unique indexes**: All unique constraints on soft-deletable tables use `WHERE deleted_at IS NULL`
