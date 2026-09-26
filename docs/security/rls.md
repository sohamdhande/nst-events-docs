# Row Level Security (RLS) & Database Context Architecture

This document defines the security architecture of PostgreSQL Row Level Security (RLS), transaction-local context injection, helper functions, and `SECURITY DEFINER` execution boundaries.

> [!NOTE]
> For the comprehensive table-by-table SQL policy definitions and schema catalog, refer to [Database RLS Master Documentation](../database/rls.md).

---

## 1. Zero-Trust Database Architecture

In standard web architectures, the database blindly trusts the application server (a single database user executes all queries on behalf of all tenants). If an SQL injection or authorization logic bug occurs in the backend, the entire database is compromised.

NST-Events enforces **Zero-Trust Database Security**:
1. Application queries execute as the unprivileged PostgreSQL role `nst_app`.
2. `nst_app` is explicitly configured with `NOSUPERUSER` and `NOBYPASSRLS`.
3. Every sensitive table has `ALTER TABLE ... FORCE ROW LEVEL SECURITY;` enabled.
4. PostgreSQL natively evaluates access predicates on every `SELECT`, `INSERT`, `UPDATE`, and `DELETE` statement.

---

## 2. Transaction-Local User Context (`withUserContext`)

Because PostgreSQL connections are pooled across concurrent Express requests, setting session variables naively (e.g. `SET app.user_id = '...'`) would leak user identities across requests sharing the same physical connection.

To guarantee complete isolation, NST-Events uses **transaction-local session variables**:

```typescript
// packages/database/src/context.ts
export async function withUserContext<T>(
  userId: string | undefined | null,
  fn: (tx: Prisma.TransactionClient) => Promise<T>,
  db: PrismaClient = prisma
): Promise<T> {
  return db.$transaction(async (tx) => {
    const value = userId || '';
    // The third parameter 'true' makes the configuration transaction-local (SET LOCAL)
    await tx.$executeRaw`SELECT set_config('app.user_id', ${value}, true)`;
    return fn(tx);
  });
}
```

### 2.1 Context Lifecycle & Safety Guarantees
- **Atomic Scope:** `SELECT set_config('app.user_id', ${userId}, true)` is strictly scoped to the transaction block (`tx`).
- **Connection Return:** When the transaction commits or rolls back, PostgreSQL automatically resets `app.user_id` to its default state.
- **Rollback Behavior:** If an error occurs inside `fn(tx)`, the transaction rolls back, cleanly discarding both data mutations and the session context.
- **Unauthenticated / Missing Context:**
  If a query executes outside `withUserContext`, `app.user_id` is empty. The SQL helper handles this gracefully:
  ```sql
  CREATE OR REPLACE FUNCTION current_user_id() RETURNS uuid AS $$
    SELECT nullif(current_setting('app.user_id', true), '')::uuid;
  $$ LANGUAGE sql STABLE;
  ```
  `nullif('', '')` returns `NULL`. All policies expecting an authenticated user evaluate to `false`, safely denying access.

---

## 3. The Recursion Breaker (`can_see_user_as_organizer`)

During Phase 18 security testing, a severe circular dependency was discovered:
1. An organizer querying `events` triggers an RLS check on `event_registrations`.
2. The `event_registrations` policy checks if the attendee exists in `users`.
3. The `users` policy checks if the caller can see the attendee via `event_registrations`.
4. Result: Infinite policy recursion and query crash.

### Resolution
In `20260811074226_secure_users_rls`, the recursion was cleanly decoupled by introducing `can_see_user_as_organizer`:
```sql
CREATE OR REPLACE FUNCTION can_see_user_as_organizer(target_user_id uuid) 
RETURNS boolean LANGUAGE plpgsql SECURITY DEFINER SET search_path TO 'public', 'pg_catalog' AS $$
BEGIN
  RETURN EXISTS (
    SELECT 1 FROM event_clubs ec 
    JOIN club_memberships cm ON ec.club_id = cm.club_id 
    WHERE cm.user_id = current_user_id() 
      AND cm.role IN ('CLUB_ADMIN', 'CORE_MEMBER', 'FACULTY_MENTOR')
      AND (
        EXISTS (SELECT 1 FROM event_registrations er WHERE er.event_id = ec.event_id AND er.user_id = target_user_id)
        OR EXISTS (SELECT 1 FROM attendance_sessions asess JOIN attendance_records ar ON ar.session_id = asess.id WHERE asess.event_id = ec.event_id AND ar.user_id = target_user_id)
      )
  );
END;
$$;
```
Because it executes as `SECURITY DEFINER`, it evaluates the organizational relationship internally without triggering the caller's RLS policies, terminating the recursive cycle while preserving zero-trust privacy.

---

## 4. `SECURITY DEFINER` Hardening & Search Path Protection

A common vulnerability in PostgreSQL functions is **search path hijacking**: if a `SECURITY DEFINER` function runs with an unqualified `search_path`, an attacker who creates an object in a public or temporary schema can execute arbitrary code with superuser privileges.

### Hardening Invariant (Phase SEC01)
In migration `20260827151000_phase_sec01_rls_hardening`, **every single stored procedure and helper function** was pinned to an explicit, immutable search path:
```sql
ALTER FUNCTION public.<function_name>(...) SET search_path TO 'public', 'pg_catalog';
```
This guarantees that object resolution is restricted exclusively to standard system and application schemas, completely neutralizing search path hijacking.

### Controlled Procedural Bypasses
Certain complex operations run as `SECURITY DEFINER` to bypass standard RLS restrictions under strictly governed PL/pgSQL logic:
1. **`mark_attendance`:** Inserts into `attendance_records` and `leaderboard_scores` atomically. Access is guarded by dynamic TOTP signature validation, physical geofence verification, and device collision advisory locks.
2. **`register_event`:** Updates `events.registration_count` and inserts into `event_registrations`. Access is guarded by atomic capacity checks and academic batch validation.
3. **`upsert_oauth_user`:** Inserts first-time users before any user ID exists to bind to `withUserContext`. Guarded by verified Google OAuth token claims.
