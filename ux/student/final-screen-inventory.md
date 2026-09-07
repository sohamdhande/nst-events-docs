# Final Student Screen Inventory

This document catalogs every student-facing UI artifact, its type, entry point, and purpose.

---

## Screens (Full Navigation Destinations)

| # | Screen | Route | Purpose | Primary Action | Spec |
|---|---|---|---|---|---|
| 1 | **Home** | `/dashboard` | Prioritized action center | Act on urgent items | [home.md](screens/home.md) |
| 2 | **Discover** | `/campus/discover` | Browse available events | Open Event Detail | [discover.md](screens/discover.md) |
| 3 | **Leaderboard** | `/campus/leaderboard` | View student rankings | — (read-only) | [leaderboard.md](screens/leaderboard.md) |
| 4 | **Clubs** | `/campus/clubs` | Browse all clubs | Open Club Detail | [clubs.md](screens/clubs.md) |
| 5 | **My Events** | `/my-events` | Participation timeline | Open Event Detail | [my-events.md](screens/my-events.md) |
| 6 | **Profile** | `/profile` | Identity and settings | — | [profile.md](screens/profile.md) |
| 7 | **Notifications** | `/notifications` | Activity feed | Follow notification | [notifications.md](screens/notifications.md) |
| 8 | **Event Detail** | `/events/:id` | Evaluate and participate | Register / View Team | [event-detail.md](screens/event-detail.md) |
| 9 | **Club Detail** | `/clubs/:id` | Understand a club | Browse club events | [clubs.md](screens/clubs.md) |
| 10 | **Team** | `/events/:id/teams/:teamId` | Team workspace | Invite / View Status | [team.md](screens/team.md) |
| 11 | **Team Invitation** | Inline (from Notification/Home) | Respond to invite | Accept / Decline | [team-invitation.md](screens/team-invitation.md) |
| 12 | **Attendance History** | `/users/me/attendance` | Review attendance records | Report Issue | [attendance-history.md](screens/attendance-history.md) |
| 13 | **Attendance Dispute** | `/attendance/disputes/:id` | Submit or view dispute | Submit / View Status | [disputes.md](screens/disputes.md) |
| 14 | **My Clubs** | `/profile/my-clubs` | View joined clubs | Open Club Detail | [my-clubs.md](screens/my-clubs.md) |
| 15 | **Notification Preferences** | `/profile/preferences` | Control notifications | Toggle preferences | [notification-preferences.md](screens/notification-preferences.md) |
| 16 | **Authentication** | `/login` | Sign in | Google Sign In | [authentication.md](screens/authentication.md) |

---

## Modals / Dialogs (Embedded Interactions)

| # | Dialog | Parent Screen | Trigger | Purpose |
|---|---|---|---|---|
| 1 | **Registration Confirmation** | Event Detail | Click "Register" | Confirm registration intent |
| 2 | **Cancel Registration** | Event Detail | Click "Cancel Registration" | Confirm cancellation |
| 3 | **Leave Team** | Team | Click "Leave Team" | Confirm leaving team |
| 4 | **Transfer Leadership** | Team | Click "Transfer Leadership" | Select new leader |
| 5 | **Remove Member** | Team | Click remove on member | Confirm removal |

---

## Contextual States (Not Separate Screens)

| # | State | Location | Description |
|---|---|---|---|
| 1 | Registration result | Event Detail | In-place state update after backend response |
| 2 | Attendance status | Event Detail | Shows PRESENT / ABSENT / No Record inline |
| 3 | Team readiness | Team view | Shows formation progress, member count, pending invites |
| 4 | Dispute status | Attendance History | Shows PENDING / APPROVED / REJECTED inline |

---

## Not Student Screens (Excluded)

These exist in the codebase but are NOT student-facing:

| Item | Reason |
|---|---|
| Admin event management | Admin-only routes under `/admin` |
| Event creation/editing | Organizer workflow |
| Attendance marking (QR) | External mechanism, not Web App |
| Club creation/management | Admin-only (`PLATFORM_ADMIN` / `CLUB_ADMIN`) |
| Queue monitoring | Admin infrastructure |

---

## Implementation Status

| Status | Meaning |
|---|---|
| **PLANNED** | All student screens listed above are planned. The current `apps/dashboard` serves admin/organizer flows. Students are currently redirected to `/student-access`. |
| **BACKEND READY** | All backend routes, models, and SSE channels required by these screens exist and are implemented. |
