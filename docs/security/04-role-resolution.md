# Role Resolution

## Mechanism
Role resolution operates at two layers: **Express RBAC middleware** (primary) and **PostgreSQL RLS** (secondary).

**In Express middleware**, when a request arrives:
1. The JWT is verified and `user_id` is extracted.
2. The middleware queries `club_memberships` to resolve the user's effective role for the relevant context.
3. If the role check fails, the request is rejected with `403 Forbidden` before any database query runs.

**In PostgreSQL RLS** (secondary check), when a query executes:
1. The database reads `current_setting('app.user_id')::uuid` — the session variable set by Prisma middleware.
2. The RLS policy looks up the `club_id` associated with the row being accessed.
3. The policy checks if the `user_id` has a row in `club_memberships` for that `club_id` with the required role.
4. If the check fails, the row is filtered out (SELECT) or the write is rejected (INSERT/UPDATE/DELETE).

**Example policy logic (PostgreSQL):**
```sql
-- current_user_id() is a stable SQL helper defined in the migrations:
-- SELECT nullif(current_setting('app.user_id', true), '')::uuid
--
-- User can UPDATE an event only if they are CLUB_ADMIN of an associated club
EXISTS (
  SELECT 1 FROM club_memberships cm
  JOIN event_clubs ec ON ec.club_id = cm.club_id
  WHERE cm.user_id = current_user_id()
    AND ec.event_id = events.id
    AND cm.role = 'CLUB_ADMIN'
)
```

## Overlapping Roles
A user can be a `MEMBER` of Club A and a `CLUB_ADMIN` of Club B. Their permissions are strictly scoped to the `club_id` in question. Effective permissions are calculated on a per-row basis in both Express middleware and RLS policies.

