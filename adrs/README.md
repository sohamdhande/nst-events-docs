# Architecture Decision Records (ADR) Index

This directory contains formal records of significant architectural decisions for the NST-Events platform.

---

## 🔒 Frozen Architecture — V1 Post-Audit Freeze

**Status**: FROZEN  
**Date**: 2026-06-09  
**Freeze Authority**: Post-audit reconciliation pass resolving all critical and high-severity findings.

All architectural contradictions identified during the technical audit have been resolved. The documentation suite is internally consistent and implementation-ready. Any further schema, API, or security deviations require a formal ADR amendment and un-freezing process.

### Resolved Audit Findings (2026-06-09)
1. **Leaderboard MV Refresh**: Changed from row-level triggers to `pg_cron` scheduled refresh every 5 minutes.
2. **Offline Attendance**: Restricted to organizer devices (`CLUB_ADMIN`/`CORE_MEMBER`) in Operations Mode. Students must validate synchronously.
3. **Registration Capacity**: Added `max_capacity` and `registration_count` to `events` table with explicit `SELECT FOR UPDATE` pattern.
4. **Audit Trigger Actor**: Triggers use `current_user_id()` reading `app.user_id` session variable via `withUserContext` wrapper.
5. **Storage URL Validation**: Express must validate domain and path before saving client-provided media URLs.
6. **Soft-Delete Cascading**: Moved from application-layer to PostgreSQL database triggers.
7. **Leadership Handover**: `FACULTY_ADMIN` can approve as fallback when no Faculty Mentor is assigned.
8. **Global Role Model**: `users.global_role` is the single source of truth. Phantom `user_roles` table removed.
9. **JWT Authorization Timing**: Club role revocations are immediate (live DB resolution). JWT TTL governs authentication session only.

### File Uploads — Deferred to Post-V1
NST-Events V1 will not support file uploads. All avatar, banner, logo, certificate, and media upload features are deferred to a later release (V1.1 / V2). V1 uses default or generated fallback assets only. **No open decisions block V1 backend implementation.**

* **ADR-008**: File Storage Provider — Deferred
* **ADR-024**: Media Storage Architecture — Deferred (depends on ADR-008)

---

## Accepted Decisions

* [ADR-001: Use Custom Node.js + Express + TypeScript + Prisma as the Primary Backend Platform](./adr-001-use-custom-express-prisma-backend.md)
* [ADR-002: Use PostgreSQL as the primary database](./adr-002-use-postgresql-as-the-primary-database.md)
* [ADR-003: Use Monorepo Architecture](./adr-003-use-monorepo-architecture.md)
* [ADR-004: Use Expo React Native](./adr-004-use-expo-react-native.md)
* [ADR-005: Use Next.js for administration dashboards](./adr-005-use-next-js-for-administration-dashboards.md)
* [ADR-006: Use PostgreSQL Row Level Security as Defense-in-Depth](./adr-006-use-postgresql-row-level-security.md)
* [ADR-007: Use Express Route Handlers and Service Modules for All Server-Side Logic](./adr-007-use-express-route-handlers.md)
* [ADR-009: Use Google OAuth via Express](./adr-009-use-google-oauth.md)
* [ADR-010: Restrict login to institutional domains](./adr-010-restrict-login-to-institutional-domains.md)
* [ADR-011: Support multi-role users](./adr-011-support-multi-role-users.md)
* [ADR-012: Authorization Uses Layered RBAC — Express First, RLS Second](./adr-012-authorization-should-primarily-be-database-driven.md)
* [ADR-013: Use Dynamic QR Attendance](./adr-013-use-dynamic-qr-attendance.md)
* [ADR-014: Support Session-Based Attendance](./adr-014-support-session-based-attendance.md)
* [ADR-015: Support Event-Based Attendance](./adr-015-support-event-based-attendance.md)
* [ADR-016: Attendance actions must be audited](./adr-016-attendance-actions-must-be-audited.md)
* [ADR-017: Use geofence validation for attendance](./adr-017-use-geofence-validation-for-attendance.md)
* [ADR-018: Support multi-club events](./adr-018-support-multi-club-events.md)
* [ADR-019: Support public and private events](./adr-019-support-public-and-private-events.md)
* [ADR-020: Support both team registration and individual registration](./adr-020-support-both-team-registration-and-individual-registration.md)
* [ADR-021: Use push notifications](./adr-021-use-push-notifications.md)
* [ADR-022: Use in-app notifications](./adr-022-use-in-app-notifications.md)
* [ADR-023: Use PostgreSQL Full Text Search](./adr-023-use-postgresql-full-text-search.md)
* [ADR-025: Administration should happen on web dashboards](./adr-025-administration-should-happen-on-web-dashboards.md)
* [ADR-026: Mobile application should be student-first](./adr-026-mobile-application-should-be-student-first.md)
* [ADR-027: Role hierarchy](./adr-027-role-hierarchy.md)
* [ADR-028: Introduce faculty roles](./adr-028-introduce-faculty-roles.md)
* [ADR-029: Use audit logs](./adr-029-use-audit-logs.md)
* [ADR-030: Attendance fraud mitigation is a first-class requirement](./adr-030-attendance-fraud-mitigation-is-a-first-class-requirement.md)
* [ADR-950: Event Type Architecture](./adr-950-event-type-architecture.md)
* [ADR-952: GPS Spoofing Strategy](./adr-952-gps-spoofing-strategy.md)
* [ADR-953: Leaderboard Scoring Model](./adr-953-leaderboard-scoring-model.md)
* [ADR-954: Event Approval Workflow](./adr-954-event-approval-workflow.md)
* [ADR-960: Database Architecture Finalization](./adr-960-database-architecture.md)
* [ADR-981: Operations Mode](./adr-981-operations-mode.md)
* [ADR-997: Merit-Based Leaderboard System](./adr-997-merit-based-leaderboard-system.md)

## Rejected Decisions

* [ADR-901: Reject Static QR Codes](./adr-901-reject-static-qr-codes.md)
* [ADR-902: Reject Mobile-Based Admin Operations](./adr-902-reject-mobile-based-admin-operations.md)
* [ADR-903: Reject Student Self Club Joining in V1](./adr-903-reject-student-self-club-joining-in-v1.md)
* [ADR-904: Reject Campus IP Validation](./adr-904-reject-campus-ip-validation.md)
* [ADR-951: Device Cryptographic Signing](./adr-951-device-cryptographic-signing.md)

## Deferred Decisions — Not In V1

* [ADR-008: File Storage Provider — Deferred to Post-V1](./adr-008-file-storage-provider.md)
* [ADR-024: Media Storage Architecture — Deferred to Post-V1](./adr-024-media-storage-architecture.md)
