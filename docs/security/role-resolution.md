# Role Resolution

## Mechanism
When a query is executed, the database resolves the user's effective permissions based on the context of the row being accessed.

If a user tries to `UPDATE events SET title = 'X' WHERE id = 'Y'`:
1. The RLS policy looks up the `club_id` associated with event `Y`.
2. The policy checks if `auth.uid()` has a row in `club_memberships` for `club_id` with `role IN ('CLUB_ADMIN', 'CORE_MEMBER')`.
3. If True, the update proceeds.

## Overlapping Roles
How overlapping roles work: A user can be a `MEMBER` of Club A and a `CLUB_ADMIN` of Club B. Their permissions are strictly scoped to the `club_id` in question. Effective permissions are calculated on a per-row basis.\n