# Master Database Architecture

## Purpose
To define the complete PostgreSQL database architecture for NST-Events, serving as the single source of truth for all data modeling decisions before API and RLS design begins.

## Database Principles
1. **Relational First**: Strongly typed columns and foreign key constraints are heavily preferred over unstructured data.
2. **Secure by Default**: Row Level Security (RLS) handles all authorization at the lowest layer.
3. **Auditability**: Destructive actions are either soft-deleted or fully tracked in immutable audit logs.

## Architecture Goals
* Support highly concurrent registration spikes (500+ users in minutes).
* Prevent proxy attendance through geofencing and dynamic QR codes.
* Accommodate multi-club events and complex overlapping faculty/student roles.

## Core Design Decisions
* We use a unified generic `events` table with structured JSONB metadata.
* Authorization is heavily tied to the `club_memberships` table mapping users to roles within specific contexts.
* Real-time tracking is handled through Supabase subscriptions.
* Intensive async tasks (notifications) use `pgmq`.

## Schema Overview: Major Domains
* **Users & Roles**: `users`, `club_memberships`
* **Clubs**: `clubs`
* **Events**: `events`, `event_clubs`, `event_approvals`
* **Registrations\nEvent Results (Competition Outcomes)**: `event_registrations`, `teams`, `team_members`
* **Attendance**: `attendance_sessions`, `attendance_records`
* **Notifications**: `notifications`, `notification_preferences`
* **Analytics & Gamification**: `club_leaderboard_mv` and `student_leaderboard_mv` as materialized views
* **Audit Logs**: `audit_logs`

## Database Conventions
* **UUID Strategy**: All primary keys are `UUIDv4`.
* **Naming Conventions**: `snake_case` for all tables, columns, and enums. Plural table names (`users`).
* **Timestamp Standards**: All tables include `created_at` and `updated_at` (managed via trigger) with `TIMESTAMPTZ`.
* **Soft Delete Rules**: Critical operational tables use a `deleted_at` column instead of hard deletes.

## Future Expansion Strategy
* Schema is designed to handle multiple campuses by eventually introducing a `tenant_id` column, though V1 assumes a single campus.
