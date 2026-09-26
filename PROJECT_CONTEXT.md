# Master Context: NST-Events

## Project Summary
NST-Events is a production-grade campus platform for Newton School of Technology (NST), Pune. Target deployment is for 3000+ active students. The platform centralizes Event Management, Club Operations, dynamic QR-based Attendance, Announcements, and comprehensive Analytics. Build timeline: 2 months, 6-developer team.

## Current Architecture

| Layer | Technology | Notes |
|---|---|---|
| **Mobile App** | Expo React Native | Student-first; all campuses |
| **Web Dashboard** | Next.js | Admin, approval, operations |
| **API Server** | Node.js + Express + TypeScript | `nst-api` K3s Deployment, 2–3 replicas |
| **Queue Worker** | Node.js (same codebase) | `nst-worker` K3s Deployment, **1 replica**, separate from API |
| **ORM** | Prisma | All DB access; generates TypeScript types |
| **Database Engine** | PostgreSQL | Hosting TBD (see Open Questions) |
| **Auth** | Google OAuth 2.0 via Express | Express owns the full OAuth lifecycle |
| **Session** | 15-min JWT + 30-day DB-backed refresh token | See `docs/security/02-jwt-strategy.md` |
| **Authorization** | Express RBAC (primary) + PostgreSQL RLS (secondary) | RLS uses `current_user_id()` per-transaction |
| **Real-time** | SSE (Server-Sent Events) via Express | No WebSockets in V1 |
| **Storage** | **Deferred to post-V1 (Mostly)** | General file uploads are not in V1. However, Club Branding (Banners) is the approved pilot for the future upload architecture. See ADR-008. |
| **Notifications** | Expo Push via `nst-worker` + pgmq | Async delivery, 5s polling interval |
| **TypeScript Types** | Prisma-generated | Schema-first; types auto-generated via `prisma generate` |
| **Deployment** | NST Cluster (K3s worker nodes: 8GB RAM) | 2-node CNPG Postgres + external S3 backup (OOM prevention) |
| **Local Dev** | Docker Compose | Express + PostgreSQL, single command |

---

## User Identity Model

Users are identified by:
- `id` (UUID) — internal platform primary key, generated at first login
- `google_sub` (TEXT UNIQUE) — stable OIDC subject identifier from Google `id_token`. Used for OAuth upsert on every login. **Not a foreign key.**

Domain restriction enforced at OAuth callback: only `@adypu.edu.in` (students) and `@newtonschool.co` (faculty, management).

---

## Authorization Architecture

**Two independent layers. Both must agree for access to be granted.**

1. **Express RBAC Middleware (Primary)**: Verifies JWT, resolves effective role from `club_memberships` live, rejects with `403 Forbidden` before any query runs.
2. **PostgreSQL RLS (Secondary / Defense-in-Depth)**: Enforces row visibility and write guards at the database level using `current_user_id()` — a helper that reads `current_setting('app.user_id', true)::uuid` set per-transaction by the `withUserContext` Prisma wrapper.

RLS is a guardrail, not a full authorization system. Express makes the primary permission decision.

---

## Session Strategy

- **Access JWT**: 15-minute TTL, contains only `user_id`, HS256-signed
- **Refresh Token**: 30-day opaque token, SHA-256 hashed, stored in `refresh_tokens` table
- **Rotation**: Refresh token is rotated (revoked + new issued) on every use
- **Revocation**: Platform Admin can delete all `refresh_tokens` rows for a user to force logout
- **Cookie**: Refresh token delivered as HttpOnly, Secure, SameSite=Strict cookie
- **Full spec**: `docs/security/02-jwt-strategy.md`

---

## User Roles (Two-Tier Model)
The platform uses a two-tier role model (see `docs/backend/04-enums.md` and `docs/api/multi-role-handling.md`):
- **`global_role`** (One per user, platform-wide):
  - `STUDENT`: Base role; can view events, register, and mark attendance.
  - `FACULTY_ADMIN`: Administrative capabilities over academic/faculty-led events.
  - `PLATFORM_ADMIN`: Superuser with complete system access.
- **`club_role`** (Attached per `club_membership`, scoped to that club only; users can hold different roles in different clubs):
  - `FACULTY_MENTOR`: Faculty oversight for a specific club.
  - `CLUB_ADMIN`: Full management over a specific club's events and roster.
  - `CORE_MEMBER`: Elevated club privileges.
  - `MEMBER`: Associated with a specific club.
*(Note: There is no single ordinal ranking across the two axes — they are separate axes, not one flat ladder. ADR-027 is superseded by this two-tier model.)*

---

## Finalized Architecture Domains (FROZEN V1)

1. **Backend Implementation Blueprint** (`docs/backend/`): The canonical, step-by-step implementation guide containing the Roadmap, Repository Structure, Prisma Schema Plan, API Contract Freeze, Development Order, and exhaustive Task Breakdown.
2. **Database Architecture** (`docs/database/`): 20+ documents covering ER diagrams, indexes, queues, JSONB rules, PostGIS geofencing, soft-delete views, and atomic capacity update locks (no `SELECT FOR UPDATE`).
3. **API & Security Architecture** (`docs/api/`, `docs/security/`): Express RBAC + RLS dual-layer model, pgmq integration, live role resolution, Table Policy Matrices, JWT & session strategy (with Token Family Revocation).
4. **Mobile Navigation** (`docs/mobile/`): `get_home_feed()` delta-sync, Pessimistic Registration, Waitlist states, Domain Governance.
5. **Events & Attendance Models**: Generic JSONB `events`, 4-step approval workflows, Dynamic QR TOTP, HMAC validation.
6. **Dashboard Navigation** (`docs/dashboard/`): Single shell, Command Palette, Operations mode, strict routing.
7. **Design System** (`docs/design-system/`): Token-driven, Geist typography, NativeWind, accessibility.
8. **Leaderboard** (`docs/features/leaderboard/`): Materialized views aggregating `leaderboard_scores`, abuse prevention, audited role assignments.
9. **Component Inventory** (`docs/ui/`): 75+ React Native and Next.js components.
10. **Product Surface**: Dispute flows, leadership handovers, multi-club events.
11. **Diagram Suite** (`docs/diagrams/`): 22 Mermaid diagrams.
12. **Cluster & Infrastructure Suite** (`docs/cluster/` or root `cluster/`): 8-chapter master guide detailing the 7-node bare-metal K3s college cluster, Cloudflare tunnel edge ingress, Traefik routing, Longhorn 3-way replicated block storage, self-hosted GitHub Actions CI/CD on `nst-n6`, NetworkPolicies, and disaster recovery runbooks.

---

## Open Questions — Requires Human Decision Before Implementation

| Decision | Status | Blocking |
|---|---|---|
| **Database Hosting** | Under Review | No — any option works with Prisma |
| **File Storage Provider** | **Deferred (Mostly)** | No general file uploads are in V1. However, Club Branding (Banners) is the approved pilot, and a storage provider will be selected for it. All other file uploads are deferred to V1.1/V2. |

---

## Active Phase
- **Architecture Phase**: Complete. This repository is the implementation-grade blueprint (10/10 Readiness Score).
- **Implementation Phase**: Ready to start. Use the `docs/backend/` blueprint to initialize the Turborepo and begin the Database Foundation phase.

---

