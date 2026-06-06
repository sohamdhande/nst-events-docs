# Leaderboard Data Model

## Derived Architecture
The Leaderboard is purely a **Derived View**. Points must never be directly editable. 

## Merit-Based Point Model
Points are automatically calculated via SQL materialized views based on the following constants:

### Attendance Roles (from `event_registrations.participation_role`)
* `ATTENDEE`: +10
* `VOLUNTEER`: +20
* `ORGANIZER`: +30
* `SPEAKER`: +40
* `MENTOR`: +40
* Session Attendance: +5

### Competition Results (from `event_results.result_type`)
* `PARTICIPANT`: +15
* `TOP_10`: +30
* `SECOND_RUNNER_UP`: +40
* `RUNNER_UP`: +50
* `WINNER`: +75

## Abuse Prevention
Manual point insertion is strictly prohibited. Points are derived exclusively from verified Attendance Records, Participation Roles, and Competition Results. 
Only the Platform Admin may perform an `ADJUST_POINTS` RPC for disciplinary reasons, which enforces strict audit logging.
