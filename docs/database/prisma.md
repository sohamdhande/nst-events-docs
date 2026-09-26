# Prisma ORM Integration & Client Architecture

This document defines how Prisma ORM is integrated into the NST-Events monorepo, which applications consume it, and its operational limitations.

---

## 1. Version & Package Configuration

- **Prisma Version:** `5.22.0` (Pinned identically across `prisma` CLI and `@prisma/client`)
- **Package Location:** `packages/database`
- **Schema Location:** `packages/database/prisma/schema.prisma`
- **Preview Features Enabled:**
  ```prisma
  generator client {
    provider        = "prisma-client-js"
    previewFeatures = ["views"]
  }
  ```

---

## 2. Monorepo Consumption Matrix

| Application / Package | Directly Imports Prisma? | Connection Mechanism | Notes |
| :--- | :--- | :--- | :--- |
| **`apps/api`** | **YES** | Imports `@nst/database` (`prisma` + `withUserContext`) | Connects as `nst_app` (`DATABASE_URL`). Sets transaction-local context (`app.user_id`) for RLS. |
| **`apps/worker`** | **YES** | Imports `PrismaClient` from `@nst/database` | Overrides datasource to `WORKER_DATABASE_URL` connecting as `nst_worker`. Executes raw SQL for `SKIP LOCKED` job claiming. |
| **`apps/dashboard`** | **NO** | Zero direct database connection | Communicates strictly via HTTPS REST and SSE APIs to `apps/api`. Persisting access tokens or querying the DB directly is strictly prohibited. |
| **`apps/mobile`** | **NO** | Zero direct database connection | Communicates strictly via HTTPS REST and SSE to `apps/api`. |
| **Migrations & Seed** | **YES** | Prisma CLI via `package.json` scripts | Connects as `postgres` admin (`ADMIN_DATABASE_URL`) with full DDL/superuser privileges. |

---

## 3. Client Export Architecture

Prisma is packaged inside `packages/database/src` with a singleton wrapper and transaction context helper:

```mermaid
graph TD
    Schema[packages/database/prisma/schema.prisma] -->|prisma generate| PClient[@prisma/client in node_modules]
    PClient --> ClientTs[packages/database/src/client.ts: singleton prisma]
    ClientTs --> ContextTs[packages/database/src/context.ts: withUserContext]
    ContextTs --> IndexTs[packages/database/src/index.ts: re-exports]
    IndexTs --> API[apps/api]
    IndexTs --> Worker[apps/worker]
```

### 3.1 The `withUserContext` Helper
Because PostgreSQL Row Level Security relies on session settings, `packages/database/src/context.ts` provides a transaction-bound wrapper:

```typescript
export async function withUserContext<T>(
  userId: string | undefined | null,
  fn: (tx: Prisma.TransactionClient) => Promise<T>,
  db: PrismaClient = prisma
): Promise<T> {
  return db.$transaction(async (tx) => {
    const value = userId || '';
    // Sets transaction-local configuration (true flag = is_local)
    await tx.$executeRaw`SELECT set_config('app.user_id', ${value}, true)`;
    return fn(tx);
  });
}
```

> [!IMPORTANT]
> The `true` parameter in `set_config('app.user_id', ${value}, true)` ensures the setting is **transaction-local**. When the transaction commits or aborts, the setting is instantly discarded. This prevents identity contamination across pooled database connections.

---

## 4. What Prisma Cannot Represent (Raw SQL Boundary)

Prisma's declarative schema cannot natively represent several core PostgreSQL primitives used in NST-Events. These must be maintained via raw SQL migrations:

### 4.1 PostGIS Spatial Columns
- In `schema.prisma`:
  ```prisma
  locationGeofence Unsupported("geography")? @map("location_geofence")
  ```
- Spatial queries and transformations (`ST_DWithin`, `ST_MakePoint`, `ST_SetSRID`, `ST_AsGeoJSON`) cannot be queried using standard Prisma model queries (e.g. `prisma.event.findMany`). They must use `prisma.$queryRaw` or raw SQL functions.

### 4.2 Full-Text Search tsvectors
- In `schema.prisma`:
  ```prisma
  searchVector Unsupported("tsvector")? @default(dbgenerated("to_tsvector('english'::regconfig, ((COALESCE(title, ''::text) || ' '::text) || COALESCE(description, ''::text)))")) @map("search_vector")
  ```
- Maintained via generated column expressions and GIN indexing in raw SQL.

### 4.3 Database Triggers & Stored Procedures (RPCs)
Prisma has no model for stored functions, RPCs, or triggers:
- Stored functions (`mark_attendance`, `sync_offline_attendance`, `register_event`, `create_team`, `cancel_team`, `resolve_attendance_dispute`, `verify_qr_signature`) exist purely in PostgreSQL migrations and are executed via `prisma.$executeRaw` or `prisma.$queryRaw`.
- Triggers (`enforce_global_role_protection`, `audit_club_memberships_trigger`, `audit_attendance_records_trigger`) run transparently inside PostgreSQL on DML operations.

### 4.4 Materialized Views
- `club_leaderboard_mv` and `student_leaderboard_mv` are managed entirely in PostgreSQL migrations.
- Concurrent refresh queries (`REFRESH MATERIALIZED VIEW CONCURRENTLY ...`) are invoked using `prisma.$executeRawUnsafe`.

### 4.5 Row Level Security & Grants
`ALTER TABLE ... ENABLE ROW LEVEL SECURITY`, `FORCE ROW LEVEL SECURITY`, and `CREATE POLICY` statements cannot be declared in `schema.prisma`.

---

## 5. Migration Strategy & Operational Constraints

### 5.1 Commands
- **Local Development:** `pnpm --filter @nst/database db:migrate` (`prisma migrate dev`)
- **CI / Production Deployment:** `pnpm --filter @nst/database db:migrate:deploy` (`prisma migrate deploy`)
- **Type Generation:** `pnpm --filter @nst/database build` (`prisma generate`)

### 5.2 Critical Operational Gotcha: Enum Alteration Deadlock (PostgreSQL 55P04)
Prisma wraps each migration script inside a single atomic transaction:
```sql
BEGIN;
-- migration.sql content
COMMIT;
```
In PostgreSQL, **new enum values cannot be referenced in DML statements within the same transaction that created them**. 

For example, this crashes with error `55P04: unsafe use of new value ... of enum type`:
```sql
-- WITHIN A SINGLE MIGRATION FILE:
ALTER TYPE "AssignmentSource" ADD VALUE IF NOT EXISTS 'INSTITUTIONAL_EMAIL_INFERENCE';
UPDATE "user_academic_profiles" SET "assignment_source" = 'INSTITUTIONAL_EMAIL_INFERENCE'; -- FAILS!
```

#### The Safe Workaround (Type Swap Pattern)
When migrating enum values, migrations must use the Type-Swap pattern demonstrated in `20260907000000_corrective_schema_drift`:
```sql
CREATE TYPE "AssignmentSource_new" AS ENUM ('INSTITUTIONAL_EMAIL_INFERENCE', 'ADMIN_OVERRIDE');
ALTER TABLE "user_academic_profiles" 
  ALTER COLUMN "assignment_source" 
  TYPE "AssignmentSource_new" 
  USING ("assignment_source"::text::"AssignmentSource_new");
ALTER TYPE "AssignmentSource" RENAME TO "AssignmentSource_old";
ALTER TYPE "AssignmentSource_new" RENAME TO "AssignmentSource";
DROP TYPE "AssignmentSource_old";
```
