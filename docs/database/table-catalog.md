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

## `attendance_sessions`
* **Purpose**: Granular time blocks requiring check-in.
* **Primary Key**: `id UUID`
* **Foreign Keys**: `event_id`
* **Indexes**: `event_id` (B-Tree)
* **RLS Notes**: Same read policies as parent event.

## `attendance_records`
* **Purpose**: Verified proof of physical presence.
* **Primary Key**: `id UUID`
* **Foreign Keys**: `session_id`, `user_id`
* **Indexes**: Composite `(session_id, user_id)` (UNIQUE)
* **RLS Notes**: Insertable by self if conditions met. Updatable only by Admins.

## `event_registrations`
* **Purpose**: Tracks intent to attend.
* **Primary Key**: `id UUID`
* **Foreign Keys**: `event_id`, `user_id`, `team_id`
* **Indexes**: Composite `(event_id, user_id)` (UNIQUE)
* **RLS Notes**: Insertable by self. Readable by event organizers.\n