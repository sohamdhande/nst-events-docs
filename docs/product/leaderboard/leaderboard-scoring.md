# Leaderboard Scoring Rules (DECOMMISSIONED)

> **Status: Removed**. Leaderboards and leaderboard scoring have been decommissioned and removed from NST Events.

### Roles
* `ATTENDEE`: +10
* `VOLUNTEER`: +20
* `ORGANIZER`: +30
* `SPEAKER`: +40
* `MENTOR`: +40
* Session Attendance: +5

### Competition Results
* `PARTICIPANT`: +15
* `TOP_10`: +30
* `SECOND_RUNNER_UP`: +40
* `RUNNER_UP`: +50
* `WINNER`: +75

## Accumulation & Architecture
**Architecture Standard**: The `leaderboard_scores` table remains an immutable append-only ledger. Insertion RPCs (e.g. attendance marking, result submission) never mutate historical ledger rows. 

**Role Precedence**: Points stack based on the highest role achieved per event. Users cannot receive both ATTENDEE and SPEAKER points for the exact same session. This deduplication and "highest role" resolution is strictly computed downstream during the SQL Materialized View aggregation phase, ensuring the base ledger retains a full unadulterated history of all earned points.
