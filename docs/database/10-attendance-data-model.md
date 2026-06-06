# Attendance Data Model

## Core Tables
1. **`attendance_sessions`**: The target for check-in.
2. **`attendance_records`**: The actual proof.
3. **`attendance_disputes`**: Tracks student-submitted attendance correction requests.

## Geofence Validation
* The mobile client submits its `latitude` and `longitude`.
* The database function utilizes PostGIS `ST_DWithin` to verify the point is within the `events.location_geofence` radius before inserting the attendance record.

## Dynamic QR Sessions
* Organizers generate a time-based TOTP token. The mobile app scans this token and submits it to a Supabase Edge Function which validates the token timing before performing the database insert.

## Multi-Day Flow (Workshop Example)
* **Event**: "React Bootcamp"
* **Sessions**: "Day 1 (Monday)", "Day 2 (Tuesday)", "Day 3 (Wednesday)"
* **Flow**: Students must scan a separate QR code for each session. The analytics dashboard aggregates this to show partial or full completion.

## Audit Logs
Every `attendance_records` row includes `audit_metadata` (`JSONB`).

**Fields:**
* `device_id` (string) -- unique device identifier
* `device_os` (string) -- android / ios
* `gps_accuracy` (float) -- accuracy in meters at time of scan
* `mock_location_detected` (boolean) -- true if mock location suspected
* `app_version` (string) -- for debugging
* `flagged` (boolean) -- true if fraud signal detected
* `flag_reason` (string) -- e.g. "device_collision"
## Operations Mode Offline Cache
For Operations Mode, a local offline cache schema is used (stored in encrypted on-device storage). This is not a DB table.

* **Local Cache Entry**:
  * `user_id` (string)
  * `session_id` (string)
  * `scanned_token` (string) - the TOTP payload captured at scan time, for server replay validation
  * `scan_timestamp` (ISO8601 string)
  * `device_id` (string)
  * `gps_lat` (number)
  * `gps_lng` (number)
  * `offline_seq` (integer) - ordering for multiple offline scans in one session

* **Conflict Rule**: If the server already has an attendance record for `(user_id, session_id)`, the server record wins. The cached entry is discarded silently.
