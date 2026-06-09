# Backend Implementation Roadmap

> **Status**: IMPLEMENTATION-READY | **Version**: 1.0 | **Date**: 2026-06-09
> **Authority**: Derived from frozen architecture (Database Freeze V1, API Freeze V1, Security Freeze V1)

---

## Phase 0: Repository Setup

| Field | Detail |
|---|---|
| **Prerequisites** | Node.js 20 LTS, Docker Desktop, NST GitHub access, PostgreSQL 16+ (Docker) |
| **Deliverables** | Turborepo monorepo, Docker Compose (PostgreSQL + pgmq), TS strict mode, ESLint/Prettier, CI pipeline, `.env.example` |
| **Definition of Done** | `npm install` succeeds; `docker compose up` starts PG+pgmq; `npm run dev` starts API with hot reload; TS compiles zero errors |
| **Risks** | Turborepo workspace config may conflict with Prisma generated client paths; pgmq Docker image availability |
| **Dependencies** | None — Phase 0 is the foundation |

---

## Phase 1: Database Foundation

| Field | Detail |
|---|---|
| **Prerequisites** | Phase 0 complete, Docker Compose PostgreSQL running |
| **Deliverables** | Prisma schema (all tables); SQL migrations for extensions (`uuid-ossp`, `pgcrypto`, `postgis`, `pgmq`), `current_user_id()` helper, all enums, all tables with constraints/indexes/FKs, soft-delete views (`security_invoker=true`), `INSTEAD OF DELETE` triggers, cascade triggers, `updated_at` triggers, generated `search_vector` columns (GIN), GiST index on `location_geofence`, partial unique indexes; seed script; `pg_cron` jobs (leaderboard MV refresh every 5 min, expired token cleanup) |
| **Definition of Done** | `npx prisma migrate dev` applies cleanly; all constraints enforced; `current_user_id()` returns UUID inside transaction; seed data loads; PostGIS `ST_DWithin` query works |
| **Risks** | Prisma cannot manage PG functions/RLS/triggers/views natively — require raw SQL migrations; PostGIS may need custom Docker image; pgmq availability varies by provider |
| **Dependencies** | Phase 0 |

---

## Phase 2: Authentication

| Field | Detail |
|---|---|
| **Prerequisites** | Phase 1 complete (users, refresh_tokens tables), Google OAuth credentials |
| **Deliverables** | `GET /auth/google` (redirect), `GET /auth/google/callback` (code exchange, `id_token` verify, domain restrict `@adypu.edu.in`/`@newtonschool.co`, upsert on `google_sub`), `POST /auth/refresh` (rotate token, new JWT), `POST /auth/logout` (revoke, clear cookie); JWT 15-min HS256 `{sub: user_id}`; opaque refresh token (SHA-256 hashed, 30-day, HttpOnly/Secure/SameSite=Strict cookie); `authenticate` middleware; `withUserContext(userId, fn)` Prisma transaction wrapper |
| **Definition of Done** | New user registers via OAuth with correct `google_sub`; returning user matches `google_sub` without duplication; JWT expires at 15 min; refresh rotation works; revoked tokens rejected; non-institutional domains rejected; `withUserContext` correctly sets `app.user_id` |
| **Risks** | OAuth redirect URIs environment-specific; refresh token rotation race condition (two simultaneous refresh requests) |
| **Dependencies** | Phase 1 |

---

## Phase 3: Core Domain Models

| Field | Detail |
|---|---|
| **Prerequisites** | Phase 2 complete |
| **Deliverables** | RBAC middleware (live role resolution from `club_memberships`); route handlers for `GET/PATCH /users/me`, `GET /clubs`, `GET /clubs/:id`, `POST /clubs` (Platform Admin, RPC), `POST/DELETE/PATCH /clubs/:id/members/:userId`; RLS policies for users, clubs, club_memberships; club search `GET /search?q=&type=clubs` |
| **Definition of Done** | Club CRUD with RBAC; membership role changes audited; users update only own profile; RLS independently rejects unauthorized access; FTS returns ranked results |
| **Risks** | Role resolution logic must be consistent between Express middleware and RLS; club creation RPC must atomically create club + initial CLUB_ADMIN |
| **Dependencies** | Phase 2 |

---

## Phase 4: Event System

| Field | Detail |
|---|---|
| **Prerequisites** | Phase 3 complete |
| **Deliverables** | Route handlers: `POST /events`, `GET /events` (cursor-paginated), `GET /events/:id`, `PATCH /events/:id`, `DELETE /events/:id`, `GET/POST /events/:id/sessions`; RPCs: `submit_event_for_approval`, `approve_event`, `reject_event`, `lock_event`, `unlock_event`; event state machine; multi-club via `event_clubs`; JSONB metadata validation per `event_type`; RLS for events, event_clubs, attendance_sessions |
| **Definition of Done** | State transitions follow DRAFT→PENDING_APPROVAL→PUBLISHED→ARCHIVED; only Faculty can approve; multi-club events work; locked events reject registrations/scans; soft-delete cascades |
| **Risks** | State transitions must be atomic with audit logging; approval notification must be enqueued in same transaction |
| **Dependencies** | Phase 3 |

---

## Phase 5: Registration System

| Field | Detail |
|---|---|
| **Prerequisites** | Phase 4 complete |
| **Deliverables** | RPCs: `register_event` (SELECT FOR UPDATE), `create_team`, `join_team`, `leave_team`, `process_waitlist`; route handlers: `POST /events/:id/register`, `POST /events/:id/teams`, `POST /teams/:id/join`, `DELETE /teams/:id/leave`, `GET /events/:id/registrations`, `GET /users/me/registrations`; waitlist auto-promote; RLS for event_registrations, teams, team_members |
| **Definition of Done** | 500+ concurrent registrations don't oversell capacity; waitlist promoted FIFO; team size constraints from event metadata; team leader transfer on leave; SELECT FOR UPDATE verified under load |
| **Risks** | Race conditions in team join/leave; waitlist promotion must be atomic |
| **Dependencies** | Phase 4 |

---

## Phase 6: Attendance System

| Field | Detail |
|---|---|
| **Prerequisites** | Phase 5 complete |
| **Deliverables** | `POST /attendance/generate-qr` (TOTP, CLUB_ADMIN/CORE_MEMBER); `POST /attendance/mark` → `mark_attendance` RPC (TOTP + geofence + device collision); `POST /attendance/sync-offline` (organizer-only batch); `GET /events/:id/attendance`, `GET /users/me/attendance`; dispute system: `POST/GET/PATCH /attendance/disputes`; device collision detection; `audit_metadata` JSONB population |
| **Definition of Done** | TOTP rotates 15s, expired rejected; geofence via ST_DWithin; device collision flagged not rejected; offline sync restricted to CLUB_ADMIN/CORE_MEMBER; disputes within 24h of event end; all attendance writes trigger audit logs |
| **Risks** | TOTP clock drift; PostGIS under 200+ simultaneous scans; mock location detection reliability |
| **Dependencies** | Phase 5 |

---

## Phase 7: Notifications

| Field | Detail |
|---|---|
| **Prerequisites** | Phase 4+ (event state transitions trigger notifications) |
| **Deliverables** | `GET /notifications` (paginated inbox), `PATCH /notifications/:id/read`, `PATCH /notifications/read-all`; `GET/PATCH /notifications/preferences`; pgmq message enqueueing from RPCs; notification payload schema (title, body, type, metadata with routing) |
| **Definition of Done** | In-app notifications created atomically with triggers; preferences respected; read/unread tracked; metadata has valid deep-link routing |
| **Risks** | Fan-out for 1000+ users must be efficient; RLS prevents cross-user reads |
| **Dependencies** | Phase 4 |

---

## Phase 8: SSE (Server-Sent Events)

| Field | Detail |
|---|---|
| **Prerequisites** | Phase 6 complete |
| **Deliverables** | `GET /events/:id/live` streaming live attendance count, registration count, waitlist position, lock/unlock status; JWT auth on connection; 30s heartbeat; `Last-Event-ID` reconnection; connection cleanup; rate limit (max 3 SSE per user) |
| **Definition of Done** | Clients receive updates within 2s; connections survive pod restarts via `Last-Event-ID`; connection pool bounded; authenticated only |
| **Risks** | Open connections consume server resources; K3s load balancer timeout must accommodate long-lived connections; **cross-replica fan-out via PG LISTEN/NOTIFY is required for correctness in the 2–3 replica topology — this is a P1 deliverable (T-094), not an optional enhancement** |
| **Dependencies** | Phase 6 |

---

## Phase 9: Background Workers

| Field | Detail |
|---|---|
| **Prerequisites** | Phase 7 complete |
| **Deliverables** | `nst-worker` deployment: pgmq polling (5s, batch 100), Expo Push API, retry backoff (1m/5m/15m), DLQ, delivery write-back, `DeviceNotRegistered` handling; `pg_cron`: leaderboard MV refresh (5 min), expired token cleanup, archived event auto-transition; health check endpoint |
| **Definition of Done** | 50 notifications/sec throughput; retry intervals correct; 3 failures → DLQ; DLQ queryable by Platform Admin; stale push tokens cleaned; MV refreshes without lock contention |
| **Risks** | Single replica = SPOF (pgmq visibility timeout handles restart); Expo rate limits may throttle bulk notifications |
| **Dependencies** | Phase 7 |

---

## Phase 10: Security Hardening

| Field | Detail |
|---|---|
| **Prerequisites** | Phases 2–9 feature-complete |
| **Deliverables** | Complete RLS suite vs Table Policy Matrix; rate limiting (`express-rate-limit`): auth 10/min/IP, attendance 5/min/user, QR 10/min/user, registrations 10/min/user, teams 20/hr/user, admin 100/min/user, club broadcast 2/week; CORS; Helmet.js; Zod input validation; SQL injection audit; CSRF for refresh cookie; audit log completeness (no NULL actors) |
| **Definition of Done** | OWASP Top 10 checklist passes; rate limits enforced; every RLS policy has RBAC counterpart; audit logs have non-NULL actor_id; security headers pass scan |
| **Risks** | Rate limiting across replicas needs shared state or per-replica limits; CSRF may conflict with mobile cookie handling |
| **Dependencies** | Phases 2–9 |

---

## Phase 11: Testing

| Field | Detail |
|---|---|
| **Prerequisites** | Phase 10 complete |
| **Deliverables** | **Unit**: JWT, TOTP, role resolution, Zod schemas. **Integration**: OAuth flow (mocked Google), registration capacity locking, attendance with geofence, event state machine, soft-delete cascade, RLS enforcement, notification pipeline. **Load**: 500 concurrent registrations, 200 scans/min, 1000-user broadcast. **Tooling**: Vitest/Jest, Supertest, test DB with migration+seed+teardown, CI pipeline |
| **Definition of Done** | 80%+ coverage on business logic; all RPCs have integration tests; concurrency tests verify no overselling; RLS tests verify independent enforcement; CI green on all branches |
| **Risks** | Integration tests need PG with PostGIS+pgmq; mocking Google OAuth requires careful id_token simulation |
| **Dependencies** | Phase 10 |

---

## Phase 12: Deployment

| Field | Detail |
|---|---|
| **Prerequisites** | Phase 11 complete |
| **Deliverables** | Docker images: `nst-api`, `nst-worker`; K8s manifests: nst-api Deployment (2–3 replicas), nst-worker Deployment (1 replica), PG StatefulSet or external, Secrets (`DATABASE_URL`, `JWT_SECRET`, `EXPO_ACCESS_TOKEN`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`), Ingress (Cloudflare Tunnel → `*.nstsdc.org`), liveness/readiness probes; DB migration strategy (Prisma migrate deploy + raw SQL); rollback procedure; monitoring |
| **Definition of Done** | API at `api.nstsdc.org`; worker processing notifications; migrations apply cleanly; health checks return 200; TLS via Cloudflare; rollback tested |
| **Risks** | NST Cluster worker nodes are limited to 8GB RAM. A strict 3-node CNPG + Distributed MinIO deployment will cause OOM cascades. Downgrade to 2-node CNPG + external S3, or request dedicated DB nodes. Multi-replica SSE fan-out (PG NOTIFY) and strict 3-node HA are deferred to "Future Scaling Options" if the cluster is upgraded. |
| **Dependencies** | Phase 11, Open Decision: DB Hosting Provider, Open Decision: NST Cluster Deployment Strategy |
