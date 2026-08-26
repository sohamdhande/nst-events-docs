# Specification Gaps

## TAB-CUSTOMIZATION-001
- AREA: Mobile Tab Customization
- MISSING INFORMATION: Which tabs can be customized and how
- STATUS: PRODUCT DECISION REQUIRED (This does not block the Web implementation. For Mobile, this feature can be deferred without affecting other active screens.)

## GET_HOME_FEED-001
- AREA: Mobile Home Screen
- MISSING INFORMATION: The exact endpoint behavior
- STATUS: RESOLVED — V1 USES EXISTING APIS

## WAITLIST-001
- AREA: Waitlist
- STATUS: RESOLVED — EXISTING REGISTRATIONS FILTER SUFFICIENT

## ANALYTICS-001
- AREA: Analytics
- STATUS: DEFERRED — NOT V1

## ANNOUNCEMENTS-001
- AREA: Announcements
- STATUS: DEFERRED — NOT V1

## POINTS-ADJUSTMENT-001
- AREA: Admin Points Adjustment
- STATUS: DEFERRED — PRODUCT/DATA-MODEL FUTURE WORK

## DATA-FIELDS-CLUB-001
- STATUS: RESOLVED. Populated in DATA_CONTRACT.md via Prisma Schema inspection.

## DATA-FIELDS-ATTENDANCE-001
- STATUS: RESOLVED. Populated in DATA_CONTRACT.md via Prisma Schema inspection.

## DATA-FIELDS-LEADERBOARD-001
- STATUS: RESOLVED. Populated in DATA_CONTRACT.md via actual API validation.

## SSE-PAYLOAD-001
- STATUS: RESOLVED. Investigated Postgres migrations and Express routers to fully extract deterministic event types (`registration_count`, `waitlist_update`, `heartbeat`). Fully documented in DATA_CONTRACT.md.

## TOPBAR-CONTRACT-001
- STATUS: RESOLVED. TopBar = approved shared Web shell component.

## CONTEXTSWITCHER-BEHAVIOR-001
- STATUS: RESOLVED. ContextSwitcher = presentation-only V1 component; no backend/context mutation.

## WEB-14-PAGINATION-001
- STATUS: RESOLVED
- RESOLUTION: Cursor-based forward pagination exposed through a "Load More" control.

## TEAM REGISTRATION SEMANTICS
- STATUS: RESOLVED
- RESOLUTION: Participants must register through a team workflow. Individual event registration is not permitted for TEAM events.

## ATTENDANCE TYPE SEMANTICS
- STATUS: RESOLVED
- RESOLUTION: An event configured as SINGLE permits exactly one attendance session. An event configured as MULTI_SESSION permits multiple attendance sessions.

## TEAM WAITLIST SEMANTICS
- STATUS: RESOLVED
- RESOLUTION: When the event capacity has been reached, a team that reaches `minimum_team_size` is placed in a WAITLISTED state. V1 creates waitlisted teams. The waitlist applies to the entire team, maintaining FIFO order based on the timestamp they entered the waitlist. Capacity is not consumed until the team is promoted.

## EVENT LOCK SEMANTICS
- STATUS: RESOLVED
- RESOLUTION: Lock grace period is exactly 24 hours after `event.end_time`. Manual lock is permitted before the event ends. A locked event unconditionally blocks new registrations. Permanent locking after the deadline is enforced lazily by backend mutation checks without a physical background cron physically setting `is_locked`.

## ACADEMIC BATCH & AUDIENCE - WEB UX CONTRACT
- STATUS: RESOLVED (Documented only)
- RESOLUTION: The future Web form for Create/Edit event will have an "AUDIENCE & ACCESS" section containing Primary Club, Visibility, and Audience. Audience options are "All Students" or "Specific Batches". "Specific Batches" allows multiple AcademicBatch selections.

## ACADEMIC BATCH & AUDIENCE - MOBILE CONTRACT
- STATUS: RESOLVED (Documented only)
- RESOLUTION: Targeted events are filtered server-side. Event detail may display audience (e.g., "B.Tech CSE AI/ML · 2025–2029"). Registration failures caused by audience restrictions must have a clean semantic application error (`AUDIENCE_NOT_ELIGIBLE`).

## ACADEMIC BATCH & AUDIENCE - MIGRATION CONTRACT
- STATUS: RESOLVED (Documented only)
- RESOLUTION: The initial rollout sequence is:
  1. Create AcademicProgram records.
  2. Create AcademicBatch records.
  3. Infer student cohort from institutional emails.
  4. Resolve AcademicBatch.
  5. Assign UserAcademicProfile.
  6. Identify unresolved users.
  7. Platform/Faculty Admin resolves exceptions.
  8. Enable audience enforcement.

## TEAM INVITATION EXPIRATION
- AREA: Teams
- STATUS: RESOLVED
- RESOLUTION: Invitation expiration is set to 72 hours. Expiry is evaluated lazily when the invitee attempts to accept (throwing `INVITATION_EXPIRED`). A periodic `pg_cron` job (daily) cleans up expired invitations. The team LEADER may resend an invitation to a user whose previous invitation expired (creating a new invitation record).

## ADMIN TEAM MEMBER REMOVAL & LEADERSHIP ROLES
- AREA: Teams
- STATUS: RESOLVED
- RESOLUTION: `PLATFORM_ADMIN`, `FACULTY_ADMIN`, and `CLUB_ADMIN` (for their own event) have explicit administrative authority to execute `cancelTeam`, `removeMember`, and `transferLeadership` as management overrides. These permissions will be formally documented in the API Routing Matrix and RPC Catalog.
