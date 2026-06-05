# ADR-001: Use Supabase as the primary backend platform

## Status
Accepted

## Context
The project has a 6-person team and a 2-month timeline. We need a robust backend with minimal infrastructure overhead.

## Decision
Use Supabase as the primary backend platform.

## Consequences
Positive: Supabase significantly reduces backend development effort while still providing PostgreSQL, authentication, storage, RLS, and Edge Functions.
Negative: Platform lock-in to Supabase-specific features.

## Alternatives Considered
* **Custom Node.js Backend**: Rejected due to timeline constraints.
* **Firebase**: Rejected because we need relational data modeling.

## Future Impact
Will enable rapid prototyping and scale nicely for the target 3000+ students.
