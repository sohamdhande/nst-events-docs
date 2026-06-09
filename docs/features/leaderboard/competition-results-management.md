# Competition Results Management

## Purpose
Provides a secure UI surface for recording and verifying merit-based competition outcomes.

## Result Types
* `WINNER`
* `RUNNER_UP`
* `SECOND_RUNNER_UP`
* `TOP_10`
* `PARTICIPANT`

## Submission Flow
1. Event concludes and enters the `ARCHIVED` state.
2. Club Admin accesses the "Competition Results Entry" dashboard screen.
3. Admin selects attendees from the roster and assigns their Result Type.
4. Admin submits the batch.

## Review Flow
Faculty Mentors can access the "Competition Results Review" screen to audit the assigned winners and contest fraudulent entries.

## Leaderboard Impact & Recalculation
Submitting results triggers the `submit_competition_result` RPC. Points are reflected in the materialized leaderboard views on the next scheduled `pg_cron` refresh cycle (every 5 minutes). Platform Admin may also trigger a manual refresh via `POST /admin/leaderboard/recalculate`.

## Audit Requirements
Every assigned result creates a `SUBMIT_COMPETITION_RESULT` audit log. Students are completely barred from this UI.
