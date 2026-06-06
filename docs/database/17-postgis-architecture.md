# PostGIS Architecture

## Role in Attendance
Used to validate physical proximity during check-ins.

## Configuration
* **Event Locations**: `events.location_geofence` is stored as a PostGIS `GEOGRAPHY(Point, 4326)` type.
* **Geofence Radius**: Configurable via `attendance_sessions.geofence_radius`, defaults to 50 meters. (Maximum allowed value enforced at RPC layer).

## GiST Indexes
A GiST index on `location_geofence` ensures immediate lookups for spatial queries.

## ST_DWithin
Validation query checks:
`ST_DWithin(events.location_geofence, ST_SetSRID(ST_MakePoint(user_lng, user_lat), 4326), attendance_sessions.geofence_radius)`
