# PHASE DOCS-03 — NST EVENTS ADMINISTRATION & IDENTITY ENGINEERING ROADMAP

## A. Purpose
This document serves as the canonical implementation order and engineering architecture roadmap for the next major NST Events administration/security phase. It dictates the approach for Platform Admin safety, the NST Student Directory, login eligibility, academic identity resolution, global role management, session freshness, CSV imports, and the Users & Roles frontend workspace.

## B. Engineering Principles
A. Backend is authoritative for authorization.
B. Frontend controls visibility/UX only.
C. Global Role is separate from Club Role.
D. NST Student eligibility is separate from User existence.
E. Academic Program/Batch assignment is separate from Student eligibility.
F. Directory removal is not User deletion.
G. Security invariants must be enforced transactionally/server-side.
H. Bulk operations must be bounded, idempotent, observable, and auditable.
I. Existing academic identity inference should be reused rather than duplicated.
J. Cache invalidation is a UX optimization, not a security boundary.
K. Every security-sensitive mutation must have explicit authorization and auditability.
L. Every race-sensitive invariant must be protected at the transaction/database boundary.

## C. Domain Boundaries
The canonical domain model follows a strict hierarchy and separation of concerns:

User Identity
    ↓
NST Student Eligibility
    ↓
Academic Identity
    ├── Academic Program
    └── Academic Batch
    ↓
Global Role
    ↓
Club Membership
    └── Club Role

Important distinctions:
- **User ≠ Student Directory Entry**: A user account may exist without student eligibility, and a directory entry can exist before a user logs in.
- **Student Directory ≠ Academic Profile**: Eligibility to log in is separate from the academic cohort assignment.
- **Academic Profile ≠ Global Role**: A user's academic standing does not determine their platform-wide permissions.
- **Global Role ≠ Club Role**: Platform administration permissions (`PLATFORM_ADMIN`, `FACULTY_ADMIN`) are strictly distinct from club-specific permissions (`CLUB_ADMIN`, `FACULTY_MENTOR`). These domains must not be collapsed.

## D. Current Architecture
Currently, student eligibility is loosely enforced by validating the Google OAuth email against a trusted domain list (`ALLOWED_EMAIL_DOMAINS`). There is no pre-authorized directory. Role changes are managed via `POST /v1/admin/users/:userId/role`, but this endpoint suffers from a critical race condition allowing the last `PLATFORM_ADMIN` to be demoted concurrently. Existing academic identity is inferred from email via `parseAdypuEmail` on the first login. Session revocation updates `revokedAt` on refresh tokens but does not immediately invalidate active JWTs or frontend state.

## E. Phase Sequence
The following implementation order is canonical and locked. Do not reorder these phases unless a later implementation audit proves a dependency requires it:

API-33: PLATFORM ADMIN INVARIANT
↓
API-34: STUDENT DIRECTORY DOMAIN / DATA CONTRACT
↓
API-35: STUDENT DIRECTORY API + AUTHORIZATION
↓
API-36: LOGIN ELIGIBILITY ENFORCEMENT
↓
API-37: CSV IMPORT / BULK DIRECTORY OPERATIONS
↓
API-38: SESSION + ROLE FRESHNESS / RECONCILIATION
↓
WEB-52: STUDENTS / ADMIN ROLES WORKSPACE
↓
TEST-53: SECURITY / CONCURRENCY / END-TO-END HARDENING

## F. API-33 Platform Admin Invariant
**Objective:** Never allow the system to reach zero `PLATFORM_ADMIN` users.
**Vulnerability:** The current pattern of `findUnique` → `validate` → `update` is vulnerable to concurrent demotion.
**Requirement:** The implementation phase must evaluate and select a robust server/database-level strategy (e.g., row locking, advisory lock, serializable transaction, atomic conditional mutation, or database invariant). No frontend-only solution is acceptable.
**Required Test Scenarios:**
1. Admin A demotes Admin B.
2. Admin B demotes Admin A concurrently.
3. Last remaining Platform Admin attempts demotion.
4. Concurrent promotions/demotions.
5. Self-demotion.
6. Unauthorized role mutation.

## G. API-34 Student Directory
**Domain Model:** `AuthorizedStudent` (preferred).
**Fields:** Must cover at minimum normalized email, eligibility/access state, created timestamp, updated timestamp, and actor/audit metadata where appropriate.
**Rules:**
- Do NOT duplicate Program, Batch, Club Role, or Global Role unless there is a concrete domain reason.
- The directory must support pre-authorizing students before their first login.
- Clearly distinguish CURRENT, PLANNED, DEFERRED, and PRODUCT DECISION REQUIRED fields.

## H. API-35 Student Directory API
**Capabilities:** list students, search students, add student, remove student, bulk import, eligibility state.
**Security:** Only `PLATFORM_ADMIN` may mutate the directory. Backend must enforce this independently of the UI. BOLA/IDOR protection is mandatory.

## I. API-36 Login Eligibility
Login must rigorously follow this sequence:
1. Authenticate Google identity.
2. Normalize email.
3. Evaluate Student Directory eligibility.
4. Deny unauthorized student access.
5. Resolve Program/Batch using existing academic identity logic (`parseAdypuEmail`).
6. Preserve existing assignment-source precedence (`AssignmentSource.ADMIN_OVERRIDE` must not be overwritten by email inference).
7. Issue session only when eligibility rules are satisfied.
**Rule:** The existing academic inference must remain the canonical resolver. Do not create a second Program/Batch parser. Domain matching alone must not grant student access.

## J. API-37 CSV Import
**Concept:** A bounded bulk operation where the initial source of truth is the `email`.
**Rules:**
- Program/Batch should continue to be resolved by the existing academic identity mechanism, not duplicated into the CSV.
- **Security Requirements:** Streaming parser, body limits, row limits, rate limiting, audit logging, deterministic validation. No naïve full-file in-memory parsing.
- Unresolved decisions (e.g., partial vs. transactional import, max rows, error reports) are marked as PRODUCT DECISION REQUIRED (see Section O).

## K. API-38 Session / Role Freshness
**Gap:** Revoking refresh tokens does not invalidate already-issued JWTs immediately. The frontend current-user cache also holds stale role data.
**Target Architecture Requirements:** Define what invalidates access, how revoked users are rejected (HTTP 403), how role changes propagate (e.g., forcing a refetch), current-user cache reconciliation, and stale authorization UX. Security-sensitive authorization remains backend-authoritative.

## L. WEB-52 Users & Roles
Build only after API layers are stable. Extend the existing Admin Users foundation.
**Workspace:** `Users & Roles` split into `[ Students ]` and `[ Admin Roles ]`.
- **Students:** search, import CSV, add student, remove student, authoritative Program/Batch, access state.
- **Admin Roles:** search, role filter, role change, role audit visibility.
**Rule:** Only `PLATFORM_ADMIN` can mutate either workspace. Do not expose Club Role mutation as Global Role mutation.

## M. TEST-53 Hardening
Mandatory test coverage (integration and frontend behavior, not just source-string tests):
- Security, Concurrency, BOLA, Role escalation, Last-admin invariant, CSV import, Duplicate imports, Directory removal, Login eligibility, Academic identity, Admin override, Session revocation, Stale-role reconciliation, Role/Club-role isolation, Audit logging.

## N. Security Requirements
Mandatory:
- `PLATFORM_ADMIN`-only global role mutation.
- `PLATFORM_ADMIN`-only Student Directory mutation.
- Backend BOLA enforcement.
- No self-escalation.
- Last-admin invariant.
- No client-trusted role claims.
- No client-trusted student eligibility.
- No client-trusted Program/Batch.
- Bounded CSV imports.
- Auditability of sensitive operations.
- Concurrency-safe mutations.

## O. Product Decisions
These explicitly locked decisions require approval:
1. **Student + Admin coexistence:** Recommended: allow a user to remain in Student Directory while holding an admin role.
2. **Student Directory removal:** Revoke student eligibility, do not delete User/domain data.
3. **CSV import:** Recommended: malformed file → fail transaction; valid file rows → deterministic/idempotent processing. Final partial-success behavior must be explicitly approved.
4. **Missing Program/Batch:** Directory eligibility and academic assignment remain separate. Define whether unresolved academic assignment blocks login or creates an "assignment pending" state.
5. **Last Platform Admin:** Zero admins must be impossible.

## P. Dependency Graph
- **API-33**: depends on existing global role mutation.
- **API-34**: depends on database architecture decision.
- **API-35**: depends on API-34.
- **API-36**: depends on API-34 + API-35.
- **API-37**: depends on API-35 + approved CSV policy.
- **API-38**: depends on API-33 + existing auth/session architecture.
- **WEB-52**: depends on API-35 + API-36 + API-37 where applicable.
- **TEST-53**: depends on completed implementation layers.

## Q. Anti-Patterns
- Do not trust email domain alone.
- Do not assign students from frontend.
- Do not make CSV the academic Program/Batch source of truth without a product decision.
- Do not use global role to infer Club role.
- Do not use Club role to infer global role.
- Do not solve last-admin safety in frontend.
- Do not delete Users when removing Student Directory eligibility.
- Do not parse giant CSVs entirely into memory.
- Do not use cache as authorization.
- Do not create duplicate authentication/academic identity systems.
- Do not build frontend before backend contracts are stable.

## R. Remaining Gaps
- CSV parsing infrastructure is missing.
- Cache reconciliation strategy for immediately reflecting demotions in the target user's UI is undefined.
- Database locking strategy for the last-admin invariant needs to be selected.

## S. Re-Audit Findings
The locked engineering sequence correctly orders the foundational changes. API-33 ensures safety before any new admin mechanics are added. API-34/35 establish the directory, allowing API-36 to enforce it safely without locking out existing users. API-37 builds the bulk loader. The sequence has no hidden dependencies. The separation of `StudentDirectory` and `User` prevents catastrophic cascading deletions. Existing RLS policies and audit logs safely tolerate these additions.

## T. Implementation Readiness
The roadmap is structured and dependencies are clear. However, several critical product decisions (handling of partial CSV failures, missing Program/Batch behavior) remain unresolved.

## U. Final Decision
ROADMAP LOCKED — PRODUCT DECISIONS REQUIRED

## V. WEB-54C Phase Output (Admin Roles & Provisioning)
The Admin Roles directory was refactored under WEB-54C to accurately represent administrative authority rather than all platform users.

**1. Admin Directory Definition:**
The Admin Directory (`GET /v1/admin/users?scope=administrators`) is strictly filtered server-side to include only:
- Global Admins (`PLATFORM_ADMIN`, `FACULTY_ADMIN`, `FACULTY_MENTOR`)
- Club Admins (users with an active `CLUB_ADMIN` record in `club_memberships`)
Ordinary `STUDENT` users are excluded at the database level.

**2. Role Isolation:**
`CLUB_ADMIN` remains explicitly isolated as a `ClubRole` within `club_memberships` and is never merged into the `GlobalRole` enum.

**3. Provisioning Domains:**
- **@newtonschool.co**: Must be provisioned as `FACULTY_MENTOR`, `FACULTY_ADMIN`, or `PLATFORM_ADMIN`. They cannot be provisioned as `STUDENT` or given Club Admin roles.
- **@adypu.edu.in**: Must be provisioned with a specific Club selection as `CLUB_ADMIN`. Their `GlobalRole` is hardcoded to `STUDENT`.
- Provisioning endpoints validate these domain invariants strictly.

**4. First-Login Handoff:**
Pre-provisioned Club Admins have their `club_memberships` preserved during their first Google OAuth login. The dummy identity is merged into the real identity seamlessly while handling unique constraint violations.

**5. Self-Demotion Safety:**
The backend universally rejects self-role mutation (`Cannot change own role`) to protect the last-admin invariant. The frontend `platform_admin_count` is used as advisory UI state to provide contextual warnings, but the backend remains the authoritative boundary.

**6. Frontend UI:**
The Admin Roles workspace features a compact directory view integrating "Administrative Role" and "Club" columns, allowing multi-club admins to display cleanly. Action menus are context-dependent (e.g., "View Club" for Club Admins, "Change Global Role" for Global Admins).
