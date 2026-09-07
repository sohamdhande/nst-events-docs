# Student Web UX — Implementation Specification

> **Purpose**: This is the master handoff document for implementing the NST Events Student Web Application. A future AI coding agent or engineer should read this document first to understand the complete product, then reference the linked authoritative documents for detailed specifications.

---

## Implementation Status

| Layer | Status |
|---|---|
| **Backend (API, DB, SSE)** | ✅ IMPLEMENTED — All required routes, RPCs, models, and SSE channels exist. |
| **Student Web UI** | ⏳ PLANNED — Students currently redirect to `/student-access`. All screens documented here are the target implementation. |
| **Admin/Organizer Web UI** | ✅ IMPLEMENTED — Exists under `apps/dashboard/(app)/admin` and related routes. |

---

## 1. Product Mental Model

The student thinks in terms of:

| Question | Destination |
|---|---|
| "What matters right now?" | **Home** |
| "What's happening around me?" | **Campus** (Discover, Leaderboard, Clubs) |
| "What am I committed to?" | **My Events** |
| "Who am I here?" | **Profile** |
| "What changed?" | **Notifications** |

Contextual questions answered by deeper screens:

| Question | Destination |
|---|---|
| "Should I participate?" | **Event Detail** |
| "What is my team doing?" | **Team** |
| "Was I marked present?" | **Attendance Status / History** |
| "Can I fix an attendance problem?" | **Dispute** |

> **Authoritative doc**: [mental-model.md](mental-model.md)

---

## 2. Information Architecture

```text
STUDENT WEB APP
├── Home
├── Campus
│   ├── Discover
│   ├── Leaderboard
│   └── Clubs
├── My Events (Upcoming / Waitlist / Past)
└── Profile
    ├── My Clubs
    └── Notification Preferences

Global: Notifications (bell icon)

Contextual:
  Event Detail → Registration (dialog) / Team / Attendance
  Club Detail → Club Events
  Team Invitation → Team
  Attendance History → Dispute
```

> **Authoritative doc**: [information-architecture.md](information-architecture.md)

---

## 3. Navigation

- **Primary**: Home, Campus, My Events, Profile (sidebar or bottom tabs)
- **Secondary**: Sub-tabs within Campus and My Events
- **Contextual**: Event Detail, Club Detail, Team, Notifications (reached via context)
- **Interactions**: Registration, Cancellation, Invitation response (embedded dialogs, not separate pages)

> **Authoritative doc**: [final-navigation.md](final-navigation.md)

---

## 4. Screen Inventory

16 screens + 5 dialogs + 4 contextual states. See full catalog:

> **Authoritative doc**: [final-screen-inventory.md](final-screen-inventory.md)

---

## 5. State Model

All student-facing states derive from backend enums:

| Domain | States |
|---|---|
| Registration | `REGISTERED`, `WAITLISTED`, `CANCELLED`, `NOT_REGISTERED` (derived) |
| Team | `FORMING`, `REGISTERED`, `WAITLISTED`, `CANCELLED` |
| Invitation | `PENDING`, `ACCEPTED`, `DECLINED`, `CANCELLED`, `EXPIRED` |
| Attendance | `PRESENT`, `ABSENT`, `EXCUSED`, `(No Record)` (derived) |
| Dispute | `PENDING`, `APPROVED`, `REJECTED` |
| Event | `PUBLISHED` (visible), `CANCELLED`, `ARCHIVED` |

> **Authoritative doc**: [state-model.md](state-model.md)

---

## 6. State → Action Mapping

Every state has exactly the actions the backend supports:

- REGISTERED → Cancel Registration (if not locked)
- WAITLISTED → Cancel Registration (if not locked)
- NOT REGISTERED → Register (if event open)
- PENDING invitation → Accept / Decline
- ABSENT / No Record → Report Issue (if within dispute window)

> **Authoritative doc**: [state-action-matrix.md](state-action-matrix.md)

---

## 7. State → Screen Mapping

Every state appears consistently across affected screens:

> **Authoritative doc**: [state-screen-matrix.md](state-screen-matrix.md), [cross-screen-consistency.md](cross-screen-consistency.md)

---

## 8. Critical Product Decisions

### Registration
Registration is an interaction embedded inside Event Detail. There is no standalone Registration Page, Registration Wizard, or Registration Success Page.

```text
Event Detail → Register → Confirm (dialog) → Backend → Result in-place
```

### Attendance
The Student Web App is for viewing attendance status and managing exceptions ONLY. It does NOT contain a QR scanner, camera, or any physical check-in mechanism.

### Waitlist
Waitlist promotion is fully automatic. There is no "Accept Promotion", "Claim Spot", or "Confirm Place" action.

### Clubs
Students cannot self-join clubs. Membership is admin-assigned. See [backend-gaps.md](backend-gaps.md).

---

## 9. Major Workflows

| Workflow | Authoritative Doc |
|---|---|
| Registration | [screens/registration.md](screens/registration.md) |
| Team creation & invitation | [workflows/team-registration.md](workflows/team-registration.md), [screens/team.md](screens/team.md) |
| Waitlist promotion | [workflows/waitlist.md](workflows/waitlist.md) |
| Attendance result viewing | [workflows/attendance.md](workflows/attendance.md) |
| Dispute submission | [workflows/attendance-disputes.md](workflows/attendance-disputes.md), [screens/disputes.md](screens/disputes.md) |
| Notification handling | [workflows/notifications.md](workflows/notifications.md) |
| Authentication | [workflows/authentication.md](workflows/authentication.md) |

---

## 10. Day-in-the-Life Journeys

9 realistic end-to-end scenarios covering: first event discovery, individual/team registration, waitlist promotion, attendance verification, dispute filing, club discovery, notification following, and returning user experience.

> **Authoritative doc**: [journeys/day-in-the-life.md](journeys/day-in-the-life.md)

---

## 11. Edge Cases

Complete edge-case matrix covering registration races, team full races, invitation expiry, dispute window expiry, network failures, session expiry, stale data, and deep-link handling.

> **Authoritative doc**: [edge-case-matrix.md](edge-case-matrix.md)

---

## 12. Mutation Retry Safety

| Mutation | Idempotent | Safe to Retry |
|---|---|---|
| Register | Yes | Yes |
| Cancel | Yes | Yes |
| Accept Invitation | Yes | Yes |
| Decline Invitation | No | Conditional |
| Leave Team | Yes | Yes |
| Transfer Leadership | No | Conditional |
| Submit Dispute | No | No (show existing) |
| Update Preferences | Yes | Yes |

> **Authoritative doc**: [final-journey-matrix.md](final-journey-matrix.md) (Mutation Retry Safety Matrix section)

---

## 13. Realtime Behavior

| Domain | Mechanism | Frontend Hook | Fallback |
|---|---|---|---|
| Notifications | SSE `/notifications/live` | `useRealtimeNotifications` | Refetch on reconnect (`onopen`) |
| Event state | SSE `/events/:id/live` | `useEventLiveUpdates` | Refetch on navigation |
| Waitlist promotion | Via notification SSE | `useRealtimeNotifications` | Refetch on navigation |
| Team changes | Via notification SSE | `useRealtimeNotifications` | Refetch on navigation |
| Leaderboard | No SSE | — | Standard REST refresh |
| Discover | No SSE | — | Standard REST refresh |

> **Authoritative doc**: [realtime-ux.md](realtime-ux.md)

---

## 14. Backend Constraints

- All mutations are server-authoritative. No optimistic UI for commitments.
- Event lock (`is_locked`) prevents all registration/team mutations.
- Event expiry (`end_time + 24h`) prevents all mutations.
- Team capacity is enforced atomically in PostgreSQL RPCs.
- Attendance marking requires TOTP payload (external mechanism).
- Club membership has no self-join endpoint.

> **Authoritative doc**: [backend-constraints.md](backend-constraints.md)

---

## 15. Backend Gaps

3 identified gaps, all non-blocking:
1. Club self-join (no endpoint)
2. Deep-link destination preservation on login (hardcoded redirect)
3. Dispute resolution notification to student (needs verification)

> **Authoritative doc**: [backend-gaps.md](backend-gaps.md)

---

## 16. UX Principles

12 principles derived from the product audit:
1. Server-authoritative mutations
2. Context before complexity
3. One primary action per context
4. No unnecessary screens
5. Physical check-in is external
6. Waitlist promotion is automatic
7. Hide administrative concepts
8. State-aware UI
9. Notifications are interactive destinations
10. Primary navigation contains destinations
11. Asynchronous changes surface through context
12. Club membership is admin-assigned

> **Authoritative doc**: [final-ux-principles.md](final-ux-principles.md)

---

## 17. Backend API Routes (Student-Facing)

| Endpoint | Method | Purpose |
|---|---|---|
| `/events` | GET | List events (Discover) |
| `/events/:id` | GET | Event Detail |
| `/events/:id/register` | POST | Register for event |
| `/events/:id/register` | DELETE | Cancel registration |
| `/events/:id/my-registration` | GET | Current registration status |
| `/events/:id/teams` | GET | List teams for event |
| `/events/:id/teams` | POST | Create team |
| `/events/:id/invitee-search` | GET | Search eligible invitees |
| `/teams/:id/join` | POST | Join team |
| `/teams/:id/leave` | DELETE | Leave team |
| `/teams/:id/invitations` | POST | Invite member |
| `/teams/:id/invitations/:invId/accept` | POST | Accept invitation |
| `/teams/:id/invitations/:invId/decline` | POST | Decline invitation |
| `/teams/:id/transfer-leadership` | POST | Transfer leadership |
| `/users/me/registrations` | GET | My registrations |
| `/users/me/attendance` | GET | My attendance records |
| `/attendance/disputes` | POST | Submit dispute |
| `/clubs` | GET | List clubs |
| `/clubs/:id` | GET | Club detail |
| `/notifications` | GET | List notifications |
| `/notifications/live` | GET (SSE) | Realtime notifications |
| `/events/:id/live` | GET (SSE) | Event live updates |
| `/leaderboard/students` | GET | Student leaderboard |
| `/auth/google` | GET | OAuth initiation |
| `/auth/refresh` | POST | Token refresh |
| `/auth/logout` | POST | Logout |

---

## 18. File Reference Map

```text
ux/student/
├── README.md                    — Overview and methodology
├── IMPLEMENTATION-SPEC.md       — THIS FILE (master handoff)
├── capability-map.md            — Backend capabilities mapped to student actions
├── information-architecture.md  — IA and Mermaid diagram
├── backend-constraints.md       — Hard backend limits
├── backend-gaps.md              — Gaps requiring backend changes
│
├── state-model.md               — Canonical state definitions
├── state-action-matrix.md       — State → available actions
├── state-screen-matrix.md       — State → screen representation
├── cross-screen-consistency.md  — Cross-screen invariants and race diagrams
├── edge-case-matrix.md          — Complete edge-case reference
│
├── final-navigation.md          — Authoritative navigation map
├── final-screen-inventory.md    — Every screen, dialog, and contextual state
├── final-journey-matrix.md      — Journey ↔ mutation ↔ edge case mapping
├── final-ux-principles.md       — Proven UX principles
│
├── mental-model.md              — Student mental model
├── navigation.md                — Planned route structure
├── principles.md                — Core UX principles
├── realtime-ux.md               — SSE behavior and fallbacks
├── open-decisions.md            — Resolved product decisions
│
├── screens/                     — Detailed screen specs (19 files)
├── workflows/                   — Step-by-step workflows (12 files)
├── journeys/                    — End-to-end journeys (7 files)
└── state-machines/              — Mermaid state diagrams (5 files)
```
