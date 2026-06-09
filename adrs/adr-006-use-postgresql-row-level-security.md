# ADR-006: Use PostgreSQL Row Level Security as Defense-in-Depth

## Status
Accepted

## Context
All client requests go through the Express backend. The Express RBAC middleware is the primary enforcement layer — it validates the user's JWT and checks their role before any database query executes. However, relying on a single authorization layer introduces risk: if the Express middleware is misconfigured, has a bug, or is bypassed, the database would be unprotected.

PostgreSQL Row Level Security (RLS) is used as a **secondary defense-in-depth layer** to ensure that even if the Express RBAC middleware is misconfigured or bypassed, the database enforces access control independently. The database operates as if clients cannot be trusted — every query is evaluated against RLS policies regardless of how it arrived.

RLS policies reference `current_setting('app.user_id')::uuid` (not `auth.uid()`, which was specific to Supabase Auth). The Express backend sets this session variable via a Prisma middleware before executing any query, ensuring the database always knows which user is performing an operation.

## Decision
Use PostgreSQL Row Level Security (RLS) as a secondary, defense-in-depth authorization layer alongside Express RBAC middleware.

## Consequences
Positive: Two independent enforcement layers. If Express middleware fails silently, RLS provides a safety net. Authorization is enforced at the database level regardless of the entry path.
Negative: Requires writing and maintaining complex SQL policies. RLS policies must be kept synchronized with Express middleware rules to avoid contradictions.

## Alternatives Considered
* **RLS as sole authorization layer**: Rejected. Relying solely on RLS without Express RBAC middleware would make business logic harder to test, debug, and maintain in isolation. Express middleware handles complex conditional logic (e.g., checking request payloads, rate limits, and multi-step validation) more cleanly than SQL policies can.
* **Express RBAC only, no RLS**: Rejected. Without RLS, a bug in Express middleware would expose the raw database without any safety net. Defense-in-depth is a core security principle for a platform managing student PII and attendance integrity.
* **Frontend-only gating**: Rejected as completely insecure.

## Future Impact
Ensures security is enforced at the lowest possible level. Even if the API layer evolves or is extended, the database remains independently secured.
