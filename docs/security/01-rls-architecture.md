# RLS Architecture

## RLS Role in the New Architecture
PostgreSQL Row Level Security (RLS) is the **secondary, defense-in-depth** authorization layer. The **primary** authorization layer is Express RBAC middleware, which validates the user's JWT and role before any database query executes.

RLS is written as a **guardrail**, not a business logic engine. It answers one question: "should this database role be allowed to read or write this row if the application sends this query?" Express has already answered the higher-level question "should this user be allowed to perform this operation?" before the query reaches the database.

RLS ensures that even if Express middleware is misconfigured, has a bug, or is bypassed, the database will independently enforce access control at the row level.

## How RLS Knows the Current User — Critical Implementation Detail

Prisma uses a **connection pool** (PgBouncer or the built-in Prisma pool). Connection-level `SET` statements would leak across requests sharing the same connection. Therefore, the session variable **must** be set inside a transaction boundary using `SET LOCAL`, which is scoped to the current transaction only.

### Implementation Pattern (TypeScript)

Every protected database operation uses a utility wrapper:

```typescript
// lib/db.ts
import { prisma } from './prisma';

export async function withUserContext<T>(
  userId: string,
  fn: (tx: typeof prisma) => Promise<T>
): Promise<T> {
  return prisma.$transaction(async (tx) => {
    await tx.$executeRaw`SELECT set_config('app.user_id', ${userId}, true)`;
    return fn(tx);
  });
}
```

Usage in a route handler:
```typescript
// routes/events.ts
router.get('/events', authenticate, async (req, res) => {
  const events = await withUserContext(req.user.id, (tx) =>
    tx.event.findMany({ where: { state: 'PUBLISHED' } })
  );
  res.json(events);
});
```

### Why `set_config(..., true)` not `SET LOCAL`
`SELECT set_config('app.user_id', '<uuid>', true)` with the third argument `true` is equivalent to `SET LOCAL` — it is transaction-scoped. This is the preferred form when using Prisma's `$executeRaw` because it avoids raw SQL string interpolation for the setting name.

### What Happens Without the Transaction
If a query runs outside the `withUserContext` wrapper, `current_setting('app.user_id')` returns an empty string. RLS policies must handle this gracefully using `nullif(current_setting('app.user_id', true), '')::uuid` to avoid runtime errors.

### RLS Policy Reference
```sql
-- Safe pattern used in all RLS policies
CREATE OR REPLACE FUNCTION current_user_id() RETURNS uuid AS $$
  SELECT nullif(current_setting('app.user_id', true), '')::uuid;
$$ LANGUAGE sql STABLE;
```
Policies use `current_user_id()` for clean, consistent identity resolution.

## Trust Model
We operate on a **Zero Trust** internal model. An authenticated JWT proves *identity*. *Authorization* is derived from relational data (`club_memberships`) evaluated at query time — not stored in JWT claims.

## RLS Scope: What RLS Should and Should Not Do

**RLS should:**
- Enforce row visibility rules (e.g., only show `DRAFT` events to their organizers)
- Prevent writes to rows the user does not own
- Provide a safety net if Express RBAC is bypassed

**RLS should NOT:**
- Encode complex multi-step business logic (that belongs in Express route handlers)
- Replace Express RBAC checks (they are complementary, not alternatives)
- Perform heavy subqueries on every row scan (keep policies index-friendly)

## Policy Design Principles
1. **Per-transaction scoping**: Always use `withUserContext` wrapper. Never set `app.user_id` at connection level.
2. **Use the helper function**: Reference `current_user_id()` in all policies, not raw `current_setting(...)` calls.
3. **Minimize subqueries**: `EXISTS` subqueries must use indexed columns. Avoid nested subqueries.
4. **Explicit grants**: Default state is `DENY ALL`. Every access must be explicitly granted.
5. **Mirror RBAC**: Every RLS policy must have a corresponding Express RBAC check. Discrepancies are security defects.

## Security Invoker Views
Views used to filter records execute with the permissions of the calling user (`security_invoker = true`), meaning RLS cascades securely through views.

## RLS Lifecycle
Policies are defined in standard SQL migration files (`.sql`) and managed entirely via source control. They are applied by the database administrator on each deployment.
