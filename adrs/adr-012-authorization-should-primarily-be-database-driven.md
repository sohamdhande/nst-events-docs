# ADR-012: Authorization Uses Layered RBAC — Express First, RLS Second

## Status
Accepted

## Context
The platform serves multiple client applications (Expo mobile app, Next.js dashboard) with complex, context-sensitive permission rules. Authorization cannot be left to the frontend, as any client-side permission check can be bypassed by a motivated attacker making raw HTTP requests.

## Decision
Authorization is enforced at two independent layers:

1. **Express RBAC Middleware (Primary)**: All API requests are validated by Express middleware before any handler executes. The middleware verifies the JWT, extracts the user identity, resolves their effective role by querying `club_memberships`, and rejects the request with `403 Forbidden` if the role check fails.

2. **PostgreSQL Row Level Security (Secondary / Defense-in-Depth)**: RLS policies independently enforce data access at the database query level. Even if Express middleware is bypassed or misconfigured, the database will reject unauthorized reads and writes. The Express backend sets `current_setting('app.user_id')` via Prisma middleware so RLS policies can evaluate per-row access correctly.

This layered model means authorization is deeply consistent: if both layers agree access is granted, the operation proceeds. If either layer disagrees, the request fails.

## Consequences
Positive: Two independent enforcement layers provide strong security guarantees. Consistent access control across all client types. Express middleware is unit-testable in isolation.
Negative: Authorization logic must be maintained in two places (Express middleware and SQL policies) and kept synchronized.

## Alternatives Considered
* **Frontend gating only**: Rejected as completely insecure.
* **RLS only (no Express middleware)**: Rejected because RLS policies cannot express complex request-level logic (payload validation, rate limiting, multi-step business rules). Single layer also creates a single point of failure.

## Future Impact
Ensures long-term data integrity. Any new API endpoint must implement both an Express RBAC check and a corresponding RLS policy before it is considered secure.

