# ADR-953: Leaderboard Scoring Model

## Status
Accepted

## Context
We want to gamify campus involvement to increase student engagement with clubs and events.

## Decision
We will maintain separate **Individual Leaderboard** and **Club Leaderboard**.
Points are awarded *only* for meaningful participation (e.g., confirmed physical attendance, winning an event, or organizing). No points will be awarded for mere registration to prevent spamming.

## Consequences
Positive: Encourages genuine participation rather than metric gaming.
Negative: Requires strict integration with the attendance verification system to award points.

## Alternatives Considered
* **Points for registration**: Rejected due to the high likelihood of students registering and not showing up.

## Future Impact
Will drive core engagement loops and may be tied to institutional rewards or recognition.
