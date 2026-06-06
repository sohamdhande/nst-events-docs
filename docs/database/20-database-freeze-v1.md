# Database Freeze V1

## Status
FROZEN

## Version
1.0

## Date
2026-06-05

## Accepted Risks
* Monolithic database structure (All domains in one database).
* High reliance on PostGIS for core flow.
* Deno Edge Functions must handle queue workers effectively.

## Known Tradeoffs
* Soft deletes using views add slight overhead to basic `SELECT` queries.
* Full Text Search ranking is basic compared to Elasticsearch.

## Future Improvements (Not In Scope For V1)
* Materialized views postponed (Real-time queries are sufficient for now).
* Advanced analytics postponed (Using basic aggregate queries).
* No cryptographic device signing (Relying on basic heuristics + Geofencing + Dynamic QR).
* Database partitioning for `attendance_records` postponed until year 2.

## Resolved Issues (Phase 1-4)
* Table naming contradictions (`attendance_sessions`, `club_memberships`) have been fixed.
* Leaderboard is now officially modeled as a materialized view (`club_leaderboard_mv` and `student_leaderboard_mv`).
* Added missing tables: `attendance_disputes`, `leadership_handover_requests`, `club_leaderboard_mv`, `student_leaderboard_mv`.
* Added missing columns: `audit_metadata` (to `attendance_records`), `attendance_type`, `is_locked` (to `events`), and `open_at`, `close_at`, `geofence_radius` (to `attendance_sessions`).
