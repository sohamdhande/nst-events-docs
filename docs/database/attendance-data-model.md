# Attendance Data Model

## Core Tables
1. **`attendance_sessions`**: The target for check-in.
2. **`attendance_records`**: The actual proof.

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
Every `attendance_records` row includes `audit_metadata` (e.g., `device_os`, `gps_accuracy`) to resolve grading disputes.\n