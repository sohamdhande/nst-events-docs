# Backend Changes Required

> **Notice**: All items from the original BACKEND-GAP-### audit have been verified against the codebase. Competition results endpoints and leadership transfer endpoints are confirmed as GENUINELY MISSING and are now marked as DEFERRED.

The following backend features were historically identified as gaps. This document now serves as a **HISTORICAL STATUS TRACKER**. It is NOT an open implementation list. All active V1 requirements have been met or explicitly deferred.

| ID | ORIGINAL REQUIREMENT | CURRENT STATUS | CURRENT API | FRONTEND IMPACT | NOTES |
|---|---|---|---|---|---|
| BE-CONFIRMED-001 | `GET /v1/dashboard/summary` | IMPLEMENTED | `GET /v1/dashboard/summary` | Dashboard Unblocked | Implemented in Phase 21J |
| BE-CONFIRMED-002 | `GET /v1/home/feed` | CLOSED AS DEFwERRED / NO NEW API REQUIRED | None | MOB-03 Uses Existing APIs | Home composes from existing APIs |
| BE-CONFIRMED-003 | `GET /v1/events/:id/waitlist` | CLOSED AS NOT REQUIRED FOR V1 | None | N/A | Waitlist logic is handled internally via RPC |
| BE-CONFIRMED-004 | `GET /v1/events/:id/teams` | IMPLEMENTED | `GET /v1/events/:id/teams` | Teams Unblocked | Implemented in Phase 21J |
| BE-CONFIRMED-005 | `GET /v1/admin/users` | IMPLEMENTED | `GET /v1/admin/users` | User Mgmt Unblocked | Implemented in Phase 21J |
| BE-CONFIRMED-006 | `POST /v1/admin/users/:id/role` | IMPLEMENTED | `POST /v1/admin/users/:userId/role` | User Mgmt Unblocked | Implemented in Phase 21J |
| BE-CONFIRMED-007 | `GET /v1/admin/audit-logs` | IMPLEMENTED | `GET /v1/admin/audit-logs` | Admin Unblocked | Implemented in Phase 21J |
| BE-CONFIRMED-008 | `POST /v1/admin/points/adjust` | DEFERRED — PRODUCT/DATA-MODEL DECISION | None | Points remain immutable | Design decision required; direct editing is architecturally prohibited |
| BE-CONFIRMED-009 | `GET /v1/events/:id/attendance/export` | IMPLEMENTED | `GET /v1/events/:id/attendance/export`| Attendance Unblocked | Implemented in Phase 21J |
| BE-CONFIRMED-010 | `GET /v1/analytics/*` | CLOSED AS NOT REQUIRED FOR V1 | None | Analytics Deferred | Not required for current V1 |
| BE-CONFIRMED-011 | `GET/POST /v1/announcements` | DEFERRED — NOT V1 | None | Broadcast Deferred | Broadcast feature deferred |
| BE-CONFIRMED-012 | Additional SSE types | DEFERRED | `GET /v1/events/:id/live` | Handled manually | Only `registration_count`, `waitlist_update`, and `heartbeat` are currently emitted |
| BE-CONFIRMED-013 | Leaderboard point breakdowns | STUBBED / DEFERRED | `GET /v1/leaderboard/*` | Displays `0` for breakdowns | Backend returns hardcoded 0s for breakdown points; total_points works |
| BE-CONFIRMED-014 | Competition Results | DEFERRED — NOT REQUIRED FOR V1 | None | N/A | Feature genuinely missing from backend |
| BE-CONFIRMED-015 | Leadership Transfer | DEFERRED — NOT REQUIRED FOR V1 | None | N/A | Feature genuinely missing from backend |
