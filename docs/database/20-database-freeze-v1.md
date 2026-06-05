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
