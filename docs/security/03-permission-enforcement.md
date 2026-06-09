# Permission Enforcement

## Three-Layer Permission Model

### Layer 1: Frontend Permissions (UX Only)
The UI hides buttons and routes based on the user's role (optimistic UX). This is strictly for user experience purposes and is **not** a security boundary. Any client-side gate can be bypassed by a determined attacker making raw HTTP requests.

### Layer 2: Express RBAC Middleware (Primary Security Boundary)
All API requests pass through Express RBAC middleware before any route handler executes. The middleware:
1. Verifies the JWT signature and expiry.
2. Extracts `user_id` from the token.
3. Resolves the user's effective role by querying `club_memberships` live.
4. Rejects the request with `403 Forbidden` if the role check fails.

This is the **primary enforcement layer**. If a user bypasses the frontend and sends a raw HTTP request to `DELETE /events/:id`, the Express middleware intercepts and rejects it before any database interaction occurs.

### Layer 3: PostgreSQL RLS (Secondary / Defense-in-Depth)
Even if the Express RBAC middleware is bypassed or misconfigured, PostgreSQL RLS policies independently enforce access at the row level. The database evaluates `current_setting('app.user_id')::uuid` on every qualifying query and rejects unauthorized reads and writes natively.

This layer exists as a safety net — it should never be the only thing preventing unauthorized access, but it guarantees that the database cannot be exploited even if the API layer is compromised.
