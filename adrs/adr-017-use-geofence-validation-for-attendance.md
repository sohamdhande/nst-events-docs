# ADR-017: Use geofence validation for attendance

## Status
Accepted

## Context
Students might attempt to scan dynamic QR codes streamed remotely by friends.

## Decision
Use geofence validation for attendance.

## Consequences
Positive: Students must be physically near the event. Drastically reduces proxy attendance.
Negative: GPS drift indoors can cause false negatives.

## Alternatives Considered
* **QR only**: Rejected as it is vulnerable to real-time streaming.

## Future Impact
Requires robust location permission handling in the mobile app.
