# PostGIS Architecture

## Role in Attendance
Used to validate physical proximity during check-ins.

## Configuration
* **Event Locations**: `events.location_geofence` is stored as a PostGIS `GEOGRAPHY(Point, 4326)` type.
* **Geofence Radius**: Configurable in metadata, defaults to 50 meters.

## GiST Indexes
A GiST index on `location_geofence` ensures immediate lookups for spatial queries.

## ST_DWithin
Validation query checks:
`ST_DWithin(event.location_geofence, ST_SetSRID(ST_MakePoint(user_lng, user_lat), 4326), 50)`
