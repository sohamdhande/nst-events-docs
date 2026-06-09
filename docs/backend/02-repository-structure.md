# Repository Structure

> **Status**: IMPLEMENTATION-READY | **Version**: 1.0 | **Date**: 2026-06-09
> **Authority**: ADR-003 (Monorepo Architecture), MASTER_CONTEXT.md

---

## Monorepo Root

```
nst-events/
├── apps/
│   ├── api/                    # Express REST API server (nst-api)
│   ├── worker/                 # pgmq notification worker (nst-worker)
│   ├── mobile/                 # Expo React Native app
│   └── dashboard/              # Next.js admin dashboard
├── packages/
│   ├── shared/                 # Shared TypeScript types, constants, validators
│   ├── config/                 # Shared ESLint, TSConfig, Prettier configs
│   └── database/               # Prisma schema, migrations, seed, DB utilities
├── docker/
│   ├── docker-compose.yml      # Local dev: PostgreSQL + pgmq
│   ├── Dockerfile.api          # Production image for nst-api
│   └── Dockerfile.worker       # Production image for nst-worker
├── k8s/
│   ├── api-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── postgres-statefulset.yaml
│   ├── secrets.yaml
│   └── ingress.yaml
├── turbo.json                  # Turborepo pipeline config
├── package.json                # Root workspace config
├── .env.example
├── .github/
│   └── workflows/
│       ├── ci.yml              # Lint, typecheck, test
│       └── deploy.yml          # Build + deploy to NST Cluster
└── docs/                       # Architecture documentation (this repo)
```

---

## Folder Ownership & Purpose

### `apps/api/` — Express REST API (`nst-api`)

**Purpose**: The single HTTP entry point for all client applications. Handles authentication, RBAC, route handling, and database access via Prisma.

**Ownership**: Backend team

**Internal Structure**:
```
apps/api/
├── src/
│   ├── index.ts                # Express app bootstrap + listen
│   ├── app.ts                  # Express app factory (middleware chain)
│   ├── config/
│   │   └── env.ts              # Typed environment variable loader
│   ├── middleware/
│   │   ├── authenticate.ts     # JWT verification → req.user
│   │   ├── authorize.ts        # RBAC role checks (requireRole, requireClubRole)
│   │   ├── rate-limit.ts       # express-rate-limit configuration
│   │   ├── error-handler.ts    # Global error handler (RFC 7807)
│   │   └── validate.ts         # Zod schema validation middleware
│   ├── modules/
│   │   ├── auth/
│   │   │   ├── auth.router.ts
│   │   │   ├── auth.service.ts
│   │   │   └── auth.schema.ts  # Zod request/response DTOs
│   │   ├── users/
│   │   │   ├── users.router.ts
│   │   │   ├── users.service.ts
│   │   │   └── users.schema.ts
│   │   ├── clubs/
│   │   │   ├── clubs.router.ts
│   │   │   ├── clubs.service.ts
│   │   │   └── clubs.schema.ts
│   │   ├── events/
│   │   │   ├── events.router.ts
│   │   │   ├── events.service.ts
│   │   │   └── events.schema.ts
│   │   ├── registrations/
│   │   │   ├── registrations.router.ts
│   │   │   ├── registrations.service.ts
│   │   │   └── registrations.schema.ts
│   │   ├── attendance/
│   │   │   ├── attendance.router.ts
│   │   │   ├── attendance.service.ts
│   │   │   ├── attendance.schema.ts
│   │   │   └── totp.ts         # TOTP generation + validation
│   │   ├── notifications/
│   │   │   ├── notifications.router.ts
│   │   │   ├── notifications.service.ts
│   │   │   └── notifications.schema.ts
│   │   ├── leaderboard/
│   │   │   ├── leaderboard.router.ts
│   │   │   ├── leaderboard.service.ts
│   │   │   └── leaderboard.schema.ts
│   │   ├── admin/
│   │   │   ├── admin.router.ts
│   │   │   ├── admin.service.ts
│   │   │   └── admin.schema.ts
│   │   └── sse/
│   │       ├── sse.router.ts
│   │       └── sse.service.ts
│   ├── lib/
│   │   ├── prisma.ts           # Prisma client singleton
│   │   ├── db.ts               # withUserContext() wrapper
│   │   ├── jwt.ts              # JWT sign/verify utilities
│   │   └── errors.ts           # Typed error classes (AppError, NotFoundError, etc.)
│   └── types/
│       └── express.d.ts        # Extended Express Request type (req.user)
├── tests/
│   ├── integration/
│   ├── unit/
│   └── helpers/
│       └── test-db.ts          # Test database setup/teardown
├── package.json
└── tsconfig.json
```

**Allowed Dependencies**: `@nst/database`, `@nst/shared`
**Forbidden Dependencies**: `@nst/mobile`, `@nst/dashboard` — API must never import frontend code

---

### `apps/worker/` — Background Worker (`nst-worker`)

**Purpose**: Separate Node.js process that polls pgmq for notification messages and dispatches them via the Expo Push API. Runs as its own Kubernetes Deployment (1 replica). Does NOT handle HTTP traffic.

**Ownership**: Backend team

**Internal Structure**:
```
apps/worker/
├── src/
│   ├── index.ts                # Worker bootstrap + polling loop
│   ├── consumers/
│   │   └── notification.consumer.ts  # pgmq poll → Expo Push API
│   ├── lib/
│   │   ├── expo-push.ts        # Expo Push API client
│   │   └── queue.ts            # pgmq read/delete/dlq helpers
│   └── health.ts               # HTTP health check for K8s liveness probe
├── package.json
└── tsconfig.json
```

**Allowed Dependencies**: `@nst/database`, `@nst/shared`
**Forbidden Dependencies**: `@nst/api`, `@nst/mobile`, `@nst/dashboard`

---

### `apps/mobile/` — Expo React Native

**Purpose**: Student-first mobile application. All campus users interact with events, attendance, notifications, and leaderboards through this app.

**Ownership**: Frontend (Mobile) team

**Allowed Dependencies**: `@nst/shared`
**Forbidden Dependencies**: `@nst/database`, `@nst/api`, `@nst/worker` — mobile must never import backend or database code

---

### `apps/dashboard/` — Next.js Admin Dashboard

**Purpose**: Web application for Club Admins, Faculty Mentors, Faculty Admins, and Platform Admins. Handles event management, approval workflows, attendance review, analytics, audit log inspection, and operations mode.

**Ownership**: Frontend (Web) team

**Allowed Dependencies**: `@nst/shared`
**Forbidden Dependencies**: `@nst/database`, `@nst/api`, `@nst/worker`

---

### `packages/database/` — Prisma Schema & Migrations

**Purpose**: Single source of truth for the database schema. Contains the Prisma schema file, all migration files (both Prisma-managed and raw SQL), seed scripts, and the `withUserContext` utility.

**Ownership**: Backend team (schema changes require senior review)

**Internal Structure**:
```
packages/database/
├── prisma/
│   ├── schema.prisma           # Prisma schema (tables, enums, relations)
│   ├── migrations/
│   │   ├── 0001_init/
│   │   │   └── migration.sql   # Prisma-generated DDL
│   │   ├── 0002_extensions/
│   │   │   └── migration.sql   # CREATE EXTENSION postgis, pgmq, etc.
│   │   ├── 0003_rls_policies/
│   │   │   └── migration.sql   # All RLS policies + current_user_id()
│   │   ├── 0004_triggers/
│   │   │   └── migration.sql   # Audit triggers, soft-delete cascades, updated_at
│   │   ├── 0005_views/
│   │   │   └── migration.sql   # Soft-delete views (security_invoker)
│   │   ├── 0006_rpcs/
│   │   │   └── migration.sql   # PostgreSQL stored procedures
│   │   ├── 0007_materialized_views/
│   │   │   └── migration.sql   # Leaderboard MVs
│   │   ├── 0008_search/
│   │   │   └── migration.sql   # Generated tsvector columns + GIN indexes
│   │   ├── 0009_pgcron/
│   │   │   └── migration.sql   # pg_cron job definitions
│   │   └── 0010_pgmq_queues/
│   │       └── migration.sql   # pgmq queue creation
│   └── seed.ts                 # Development seed data
├── src/
│   ├── client.ts               # Prisma client singleton export
│   └── context.ts              # withUserContext(userId, fn) utility
├── package.json
└── tsconfig.json
```

**Allowed Dependencies**: `prisma`, `@prisma/client`
**Forbidden Dependencies**: All app packages — database package must be a pure dependency, never importing from consumers

---

### `packages/shared/` — Shared Code

**Purpose**: TypeScript types, constants, Zod validation schemas, and utilities shared across API, worker, mobile, and dashboard.

**Internal Structure**:
```
packages/shared/
├── src/
│   ├── types/
│   │   ├── api.ts              # Request/response DTO types
│   │   ├── events.ts           # Event-related types
│   │   ├── roles.ts            # Role enums + type guards
│   │   └── errors.ts           # RFC 7807 error shape
│   ├── constants/
│   │   ├── roles.ts            # Role hierarchy constants
│   │   ├── limits.ts           # Rate limits, capacity limits
│   │   └── points.ts           # Leaderboard point values
│   └── validators/
│       └── schemas.ts          # Shared Zod schemas
├── package.json
└── tsconfig.json
```

**Allowed Dependencies**: `zod` (runtime validation)
**Forbidden Dependencies**: All app packages, `prisma`, `@prisma/client`

---

### `packages/config/` — Shared Configuration

**Purpose**: Shared ESLint, TypeScript, and Prettier configurations.

**Allowed Dependencies**: ESLint plugins, Prettier plugins
**Forbidden Dependencies**: All app and other packages

---

## Module Structure Within `apps/api/`

Each module follows a consistent pattern:

| File | Purpose |
|---|---|
| `*.router.ts` | Express Router with route definitions. Applies middleware (authenticate, authorize, validate). Delegates to service. |
| `*.service.ts` | Business logic. Calls Prisma or invokes PostgreSQL RPCs via `$queryRaw`. Never touches `req`/`res`. |
| `*.schema.ts` | Zod schemas for request validation and response typing. Exports inferred TypeScript types. |

### Module List

| Module | Domain | Key Responsibilities |
|---|---|---|
| `auth` | Authentication | Google OAuth flow, JWT issuance, refresh token rotation, logout |
| `users` | User Identity | Profile CRUD, own memberships query |
| `clubs` | Club Management | Club CRUD, membership management, search |
| `events` | Event Lifecycle | Event CRUD, state machine, sessions, multi-club mapping |
| `registrations` | Registration | Individual/team registration, waitlist, capacity locking |
| `attendance` | Attendance | QR generation, marking, offline sync, disputes |
| `notifications` | Notifications | Inbox, preferences, read/unread management |
| `leaderboard` | Gamification | Student/club leaderboard reads, admin recalculate |
| `admin` | Administration | Audit logs, platform admin operations, force transfers |
| `sse` | Real-time | Server-Sent Events for live data streaming |

---

## Dependency Rules Summary

```
packages/config     → (no deps)
packages/shared     → (no deps)
packages/database   → (no deps)
apps/api            → packages/database, packages/shared
apps/worker         → packages/database, packages/shared
apps/mobile         → packages/shared
apps/dashboard      → packages/shared
```

**Hard Rule**: No circular dependencies. No app package may import another app package. `packages/database` is consumed by backend apps only. `packages/shared` is consumed by all apps.
