# ADR-950: Event Type Architecture

## Status
Accepted

## Context
Events have diverse metadata needs (e.g., Hackathons need team sizes, Workshops need prerequisite info).

## Decision
We will use a Generic Event Model. All events reside in a single `events` table containing standard columns plus an `event_type` column and a `metadata` JSONB column.

## Consequences
Positive: Drastically simplifies schema and queries for the global event feed. 
Negative: Harder to enforce strict SQL-level constraints on metadata fields.

## Alternatives Considered
* **Specialized Tables**: Rejected due to high schema complexity.

## Future Impact
Will dictate the core of our data layer. RLS policies and API validation layers must enforce schema constraints for the `metadata` payload.
