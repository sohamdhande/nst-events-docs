# Development Order

---

## Critical Path

The critical path represents the longest chain of dependent work. No parallelization can shorten this path. Every item on the critical path blocks downstream work.

```
Phase 0 (Repo Setup)
  └→ Phase 1 (Database Foundation)
       └→ Phase 2 (Authentication)
            └→ Phase 3 (Core Domain — Clubs, Users, RBAC)
                 └→ Phase 4 (Event System)
                      └→ Phase 5 (Registration System)
                           └→ Phase 6 (Attendance System)
                                └→ Phase 10 (Security Hardening)
                                     └→ Phase 11 (Testing)
                                          └→ Phase 12 (Deployment)
```

**Critical path duration estimate**: ~8 weeks (2-month build timeline, 6 developers)

---

## Parallel Work Streams

Several phases can be developed in parallel once their prerequisites are met:

```
                          ┌── Phase 7 (Notifications) ──┐
Phase 4 (Events) ────────┤                              ├── Phase 9 (Workers)
                          └── Phase 8 (SSE) ────────────┘
                                (after Phase 6)
```

| Stream | Can Start After | Can Run Parallel With |
|---|---|---|
| **Stream A (Critical)**: Phases 0→1→2→3→4→5→6 | — | Nothing (serial dependency) |
| **Stream B (Notifications)**: Phase 7 | Phase 4 | Phases 5, 6, 8 |
| **Stream C (SSE)**: Phase 8 | Phase 6 | Phase 7, 9 |
| **Stream D (Workers)**: Phase 9 | Phase 7 | Phase 8 |
| **Stream E (Security)**: Phase 10 | Phases 2–9 | Nothing (requires all features) |
| **Stream F (Frontend — Mobile)** | Phase 2 (auth endpoints) | Phases 3–9 (build screens as API surfaces become available) |
| **Stream G (Frontend — Dashboard)** | Phase 3 (club/user endpoints) | Phases 4–9 |

---

## What Should Be Built First

**Strict sequential order for the backend critical path:**

1. **Monorepo + Docker Compose** (Phase 0) — 2–3 days
2. **Prisma schema + SQL migrations** (Phase 1) — 3–5 days
3. **Google OAuth + JWT + Refresh** (Phase 2) — 3–4 days
4. **Users + Clubs + RBAC middleware** (Phase 3) — 3–4 days
5. **Events CRUD + State Machine** (Phase 4) — 4–5 days
6. **Registration + Capacity Locking** (Phase 5) — 3–4 days
7. **Attendance + QR + Geofence** (Phase 6) — 5–7 days

**After Phase 4, start parallel streams:**
- Notification inbox (Phase 7) — 3–4 days
- After Phase 6: SSE (Phase 8) — 2–3 days
- After Phase 7: Worker deployment (Phase 9) — 3–4 days

---

## What Can Juniors Build Safely

These tasks have well-defined inputs, low blast radius, and clear acceptance criteria:

| Task | Phase | Why Safe |
|---|---|---|
| Docker Compose setup | 0 | Purely configuration, no business logic |
| ESLint/Prettier/TSConfig | 0 | No runtime impact |
| Seed data scripts | 1 | Read-only development tool |
| `GET /users/me` route | 3 | Simple Prisma query, no mutations |
| `GET /clubs` + `GET /clubs/:id` | 3 | Simple reads with pagination |
| `GET /events` + `GET /events/:id` | 4 | Read-only with RLS filtering |
| `GET /notifications` | 7 | Simple paginated inbox read |
| `PATCH /notifications/:id/read` | 7 | Single field update, own data only |
| `GET /notifications/preferences` | 7 | Trivial read |
| `GET /leaderboard/students` | 4+ | Read from materialized view |
| `GET /leaderboard/clubs` | 4+ | Read from materialized view |
| `GET /search` | 4+ | FTS query, read-only |
| `GET /users/me/registrations` | 5 | Own data read |
| `GET /users/me/attendance` | 6 | Own data read |
| Zod schema definitions | Any | Pure type definitions, no side effects |
| Error handler middleware | 0 | Cross-cutting, well-documented pattern |
| Unit tests for JWT utils | 2 | No database dependency |
| Unit tests for TOTP utils | 6 | No database dependency |

---

## What Requires Senior Review

These tasks involve security-critical logic, complex concurrency, or cross-cutting architectural decisions:

| Task | Phase | Why Senior Review Required |
|---|---|---|
| **Prisma schema + SQL migrations** | 1 | Foundation — errors here cascade everywhere |
| **`withUserContext` implementation** | 1 | RLS depends on this; incorrect scoping is a security hole |
| **Google OAuth callback** | 2 | Token verification, domain restriction, user upsert — security critical |
| **JWT + Refresh token rotation** | 2 | Race conditions, token reuse detection — security critical |
| **RBAC middleware** | 3 | Primary authorization layer — must be correct |
| **RLS policies (all)** | 1+ | Secondary security layer — must mirror RBAC |
| **`register_event` RPC** | 5 | `SELECT FOR UPDATE` concurrency — must not oversell |
| **`mark_attendance` RPC** | 6 | TOTP + geofence + device collision — highest complexity RPC |
| **Offline attendance sync** | 6 | Trust model for organizer-submitted data |
| **Audit triggers** | 1 | Must capture actor via `current_user_id()` — security critical |
| **Soft-delete cascade triggers** | 1 | Data integrity — incorrect cascades cause orphans or data loss |
| **`approve_event` RPC** | 4 | State machine + notification in same transaction |
| **Leadership transfer RPCs** | 3 | Complex role reassignment with Faculty fallback |
| **`force_transfer_leadership`** | 3 | SECURITY DEFINER — bypasses RLS |
| **`resolve_attendance_dispute`** | 6 | SECURITY DEFINER — bypasses RLS on attendance_records |
| **SSE connection management** | 8 | Resource leak risk, fan-out across replicas |
| **Worker retry/DLQ logic** | 9 | Message delivery guarantees |
| **K8s deployment manifests** | 12 | Production infrastructure — must be reviewed |
| **Rate limiting configuration** | 10 | Security boundary — incorrect limits = vulnerability |

---

## What Can Run In Parallel

### Week 1–2: Foundation (Serial)
- **Senior Backend**: Phase 0 + Phase 1 (repo + database)
- **Mobile Frontend**: Set up Expo project, navigation structure, auth screen UI (no API integration yet)
- **Web Frontend**: Set up Next.js project, dashboard shell, sidebar navigation

### Week 3: Authentication (Serial Backend, Parallel Frontend)
- **Senior Backend**: Phase 2 (OAuth + JWT)
- **Mid Backend**: Phase 1 polish (seed data, migration testing)
- **Mobile Frontend**: Implement Google Sign-In UI flow (connect to API once Phase 2 is done)
- **Web Frontend**: Implement Google Sign-In for dashboard

### Week 4: Core Models + Events (Backend unlocks frontend)
- **Senior Backend**: Phase 3 (RBAC middleware + clubs)
- **Mid Backend**: Phase 4 start (event CRUD routes — simple reads)
- **Mobile Frontend**: Home feed, club listing screens (connect to Phase 3 API)
- **Web Frontend**: Club management dashboard, user list

### Week 5: Events + Registration
- **Senior Backend**: Phase 4 completion (state machine RPCs) + Phase 5 (registration RPCs)
- **Mid Backend**: Phase 7 start (notification inbox, preferences)
- **Junior Backend**: Read-only endpoints, Zod schemas, search
- **Mobile Frontend**: Event detail, registration flow
- **Web Frontend**: Event creation form, approval workflow

### Week 6: Attendance
- **Senior Backend**: Phase 6 (attendance marking, QR, geofence, disputes)
- **Mid Backend**: Phase 7 completion + Phase 9 start (worker)
- **Junior Backend**: Leaderboard reads, user profile endpoints
- **Mobile Frontend**: QR scanner, attendance history
- **Web Frontend**: Attendance dashboard, dispute review

### Week 7: SSE + Workers + Security
- **Senior Backend**: Phase 8 (SSE) + Phase 10 (security hardening)
- **Mid Backend**: Phase 9 completion (worker deployment)
- **Junior Backend**: Unit tests, integration test setup
- **Mobile Frontend**: Live attendance screen, notification inbox
- **Web Frontend**: Operations mode, audit log viewer

### Week 8: Testing + Deployment
- **Senior Backend**: Phase 11 (integration + load tests) + Phase 12 (K8s manifests)
- **All Backend**: Bug fixes, edge case testing
- **All Frontend**: Integration testing, polish

---

## Integration Checkpoints

These are synchronization points where frontend and backend must align:

| Checkpoint | When | What Must Be True |
|---|---|---|
| **IC-1: Auth** | End of Phase 2 | Mobile and dashboard can complete Google OAuth and receive JWT. Auth context flows through all subsequent requests. |
| **IC-2: Club Data** | End of Phase 3 | Frontend can list clubs, view members, and see role-based UI gates. |
| **IC-3: Event Feed** | End of Phase 4 | Mobile shows published events feed. Dashboard shows event creation + approval flow end-to-end. |
| **IC-4: Registration** | End of Phase 5 | Student can register for event on mobile. Organizer sees registration list on dashboard. Capacity limits enforced. |
| **IC-5: Attendance** | End of Phase 6 | Full QR scan → attendance mark → audit flow works end-to-end. Dashboard shows attendance reports. |
| **IC-6: Notifications** | End of Phase 9 | Push notifications arrive on device after event approval. In-app inbox shows unread count. |
| **IC-7: Live Updates** | End of Phase 8 | Mobile shows live attendance count during active event via SSE. |
| **IC-8: Production** | End of Phase 12 | Full system deployed on NST Cluster. All integration tests pass against production-like environment. |

---

## Frontend Dependencies on Backend

| Frontend Feature | Required Backend Phase | Minimum API Surface |
|---|---|---|
| Login screen | Phase 2 | `/auth/google`, `/auth/callback`, `/auth/refresh` |
| Home feed | Phase 4 | `GET /events`, `GET /users/me` |
| Club browsing | Phase 3 | `GET /clubs`, `GET /clubs/:id` |
| Event detail | Phase 4 | `GET /events/:id`, `GET /events/:id/sessions` |
| Event registration | Phase 5 | `POST /events/:id/register` |
| QR scanner | Phase 6 | `POST /attendance/mark` |
| QR display (organizer) | Phase 6 | `POST /attendance/generate-qr` |
| Notification inbox | Phase 7 | `GET /notifications` |
| Push notifications | Phase 9 | Worker deployment complete |
| Live attendance count | Phase 8 | `GET /events/:id/live` (SSE) |
| Leaderboard | Phase 4+ | `GET /leaderboard/students`, `/clubs` |
| Dashboard: event management | Phase 4 | Event CRUD + approval RPCs |
| Dashboard: attendance | Phase 6 | `GET /events/:id/attendance`, disputes |
| Dashboard: audit logs | Phase 10 | `GET /admin/audit-logs` |
