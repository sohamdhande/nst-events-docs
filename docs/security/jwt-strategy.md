# JWT Strategy

## Approach
NST-Events uses standard Supabase Auth JWTs.

## Live Permission Checks vs JWT Claims
We **DO NOT** embed complex roles (like `CLUB_ADMIN for Club X`, Global Roles, Club Roles, Faculty Roles) into the JWT custom claims. 
* **Why**: Roles change dynamically. If a Club Admin is demoted, their JWT might remain valid for 1 hour, allowing them to do damage.
* **Solution**: The JWT only contains the `auth.uid()`. Every query checks the `club_memberships` table live.

## Token Expiry & Refresh Strategy
* Access tokens expire in 1 hour.
* Refresh tokens are rotated automatically by the Supabase client SDKs.

## Stale Permission Risks & Mitigations
By relying on live RLS subqueries rather than JWT claims, the risk of stale permissions is completely eliminated. Role revocation takes effect instantly.\n