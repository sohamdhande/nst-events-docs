# Table Catalog

## `users`
* **Purpose**: Stores authenticated user identities and global roles.
* **Primary Key**: `id UUID` (Maps to auth.users)
* **Foreign Keys**: None
* **Indexes**: `email` (UNIQUE B-Tree)
* **RLS Notes**: Readable by self. Public fields readable by all.

## `clubs`
* **Purpose**: Represents campus organizations.
* **Primary Key**: `id UUID`
* **Foreign Keys**: None
* **Indexes**: `name` (UNIQUE B-Tree), `to_tsvector` (GIN)
* **RLS Notes**: Readable by all. Updatable by Platform Admin.

## `club_memberships`
* **Purpose**: Maps users to clubs and assigns contextual roles (Club Admin, Member, etc).
* **Primary Key**: `id UUID`
* **Foreign Keys**: `user_id`, `club_id`
* **Indexes**: Composite `(club_id, user_id)` (UNIQUE)
* **RLS Notes**: Readable by all. Updatable by Club Admin / Faculty.

## `events`
* **Purpose**: Core generic event data.
* **Primary Key**: `id UUID`
* **Foreign Keys**: `created_by` (users.id)
* **Indexes**: `start_time` (B-Tree), `metadata` (GIN), `location_geofence` (GiST)
* **RLS Notes**: Readable by all if PUBLISHED and PUBLIC. Updatable based on club_membership roles.
* **Additional Columns**: 
  * `attendance_type` (`attendance_type_enum NOT NULL DEFAULT 'SINGLE'`)
  * `is_locked` (`BOOLEAN NOT NULL DEFAULT false`)

## `attendance_sessions`
* **Purpose**: Granular time blocks requiring check-in.
* **Primary Key**: `id UUID`
* **Foreign Keys**: `event_id`
* **Indexes**: `event_id` (B-Tree)
* **RLS Notes**: Same read policies as parent event.
* **Additional Columns**:
  * `open_at` (`TIMESTAMPTZ NOT NULL`)
  * `close_at` (`TIMESTAMPTZ NOT NULL`)
  * `geofence_radius` (`FLOAT NOT NULL DEFAULT 50`)

## `attendance_records`
* **Purpose**: Verified proof of physical presence.
* **Primary Key**: `id UUID`
* **Foreign Keys**: `session_id`, `user_id`
* **Indexes**: Composite `(session_id, user_id)` (UNIQUE)
* **RLS Notes**: Insertable by self if conditions met. Updatable only by Admins.
* **Additional Columns**:
  * `audit_metadata` (`JSONB` - Stores device_os, gps_accuracy, mock_location_detected, app_version)

## `event_registrations`
* **Purpose**: Tracks intent to attend.
* **Primary Key**: `id UUID`
* **Foreign Keys**: `event_id`, `user_id`, `team_id`
* **Indexes**: Composite `(event_id, user_id)` (UNIQUE)
* **RLS Notes**: Insertable by self. Readable by event organizers.

## `event_results`
Stores competition outcomes securely.
* `id` (uuid, pk)
* `event_id` (uuid, fk -> events.id)
* `user_id` (uuid, fk -> users.id)
* `result_type` (competition_result enum)
* `created_by` (uuid, fk -> users.id)
* `created_at` (timestamptz)

**Note on `event_registrations`:**
The `event_registrations` table has been upgraded to include a `participation_role` (enum) field.

## `attendance_disputes`
* **Purpose**: Tracks student-submitted attendance correction requests.
* **Columns**:
  * `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  * `attendance_record_id UUID REFERENCES attendance_records(id)`
  * `session_id UUID REFERENCES attendance_sessions(id)`
  * `event_id UUID REFERENCES events(id)`
  * `user_id UUID REFERENCES profiles(id)`
  * `reason TEXT NOT NULL`
  * `evidence_urls TEXT[]`
  * `status dispute_status NOT NULL DEFAULT 'PENDING'`
  * `dispute_window_expires_at TIMESTAMPTZ NOT NULL`
  * `submitted_at TIMESTAMPTZ NOT NULL DEFAULT now()`
  * `reviewed_at TIMESTAMPTZ`
  * `reviewed_by UUID REFERENCES profiles(id)`
  * `review_notes TEXT`
  * `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`

## `leadership_handover_requests`
* **Purpose**: Manages stateful club leadership transfer workflow.
* **Columns**:
  * `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
  * `club_id UUID REFERENCES clubs(id)`
  * `initiated_by UUID REFERENCES profiles(id)`
  * `successor_id UUID REFERENCES profiles(id)`
  * `faculty_mentor_id UUID REFERENCES profiles(id)`
  * `status handover_status NOT NULL DEFAULT 'PENDING'`
  * `initiated_at TIMESTAMPTZ NOT NULL DEFAULT now()`
  * `reviewed_at TIMESTAMPTZ`
  * `review_notes TEXT`
  * `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`

## `club_leaderboard_mv` (Materialized View)
* **Purpose**: Derived club-level aggregate scores. Never written to directly.
* **Source**: Aggregated from attendance_records, event_results, registrations.
* **Columns**:
  * `club_id UUID`
  * `club_name TEXT`
  * `total_points INTEGER`
  * `event_count INTEGER`
  * `member_count INTEGER`
  * `last_refreshed_at TIMESTAMPTZ`
* **Refresh trigger**: On attendance mark, event result submission, or manual admin trigger.

## `student_leaderboard_mv` (Materialized View)
* **Purpose**: Derived per-student aggregate scores. Never written to directly.
* **Columns**:
  * `user_id UUID`
  * `display_name TEXT`
  * `total_points INTEGER`
  * `attendance_points INTEGER`
  * `contribution_points INTEGER`
  * `competition_points INTEGER`
  * `last_refreshed_at TIMESTAMPTZ`
