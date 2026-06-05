# ADR-006: Use PostgreSQL Row Level Security

## Status
Accepted

## Context
Direct database access from clients (via Supabase) bypasses traditional backend middleware.

## Decision
Use PostgreSQL Row Level Security (RLS).

## Consequences
Positive: Authorization must be enforced at the database layer rather than frontend logic. Guaranteed secure data access.
Negative: Requires writing complex SQL policies.

## Alternatives Considered
* **Application-level middleware**: Rejected because clients talk directly to Supabase PostgREST APIs.

## Future Impact
Ensures security is enforced at the lowest possible level.
