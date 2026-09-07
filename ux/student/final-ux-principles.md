# Final Student UX Principles

These principles are derived from the completed product audit — backend behavior, established UX decisions, and cross-screen consistency verification. They are not generic design guidelines.

---

## 1. Server-Authoritative Mutations

Every action that changes a student's commitment (Register, Cancel, Accept Invitation, Leave Team, Submit Dispute) must wait for an explicit backend response before updating the UI.

- No optimistic registration.
- No optimistic attendance marking.
- No optimistic team membership changes.
- Loading states must be clear during mutations.

**Source**: Backend uses PostgreSQL RPCs with atomic state transitions (`register_event`, `accept_invitation`, `leave_team`). The frontend cannot predict the outcome.

---

## 2. Context Before Complexity

A student should understand their situation before being asked to act.

- Event Detail shows state before presenting CTAs.
- Team view shows membership and readiness before inviting.
- Attendance shows the record before offering dispute.

**Anti-pattern**: Showing a "Register" button before the student has read what the event is.

---

## 3. One Primary Action Per Context

Each screen or state should have exactly one obvious next action.

- Event Detail (not registered) → Register
- Event Detail (registered) → View Team or Cancel
- Invitation (pending) → Accept or Decline
- Attendance (absent) → Report Issue

**Anti-pattern**: Presenting Register, Cancel, View Team, and Check Attendance as equally weighted options.

---

## 4. No Unnecessary Screens

Registration, cancellation, and invitation responses are embedded interactions — not separate pages.

- Registration is a confirmation dialog inside Event Detail.
- Cancellation is a confirmation dialog inside Event Detail.
- Invitation acceptance is an inline action on the invitation card.

**Anti-pattern**: Registration Wizard, Registration Success Page, separate Waitlist Page.

---

## 5. Physical Check-In Is External

The Student Web App does NOT contain a QR scanner, camera, or any physical check-in mechanism.

It displays:
- Attendance status (PRESENT / ABSENT / No Record)
- Attendance history
- Dispute submission and resolution

**Source**: `POST /attendance/mark` requires a TOTP payload from a physical QR code — this is consumed by an external scanning mechanism, not the student's browser.

---

## 6. Waitlist Promotion Is Automatic

When capacity opens, the backend automatically promotes the next waitlisted student to `REGISTERED`. There is no student "Accept Promotion" or "Claim Spot" action.

The student learns about promotion through:
1. SSE notification (`WAITLIST_PROMOTED`)
2. Notification bell update
3. My Events state change on next visit

**Source**: `cancel_registration` RPC returns promoted user IDs and immediately calls `enqueueNotification`.

---

## 7. Hide Administrative Concepts

Students never see:
- `DRAFT`, `PENDING_APPROVAL`, `REJECTED` event states
- `is_locked` boolean
- `SQLSTATE` error codes
- `RLS denied` messages
- Internal team status distinctions beyond what affects their actions

Students see:
- Events that are `PUBLISHED` (available) or `ARCHIVED` (past)
- Clear, neutral error messages

---

## 8. State-Aware UI

Actions appear only when valid. The UI removes or disables actions based on current backend state.

- No "Register" if already registered
- No "Invite" if team is full
- No "Accept" on an expired invitation
- No "Report Issue" if dispute window has closed
- No "Cancel" if event is locked

**Source**: Each mutation endpoint validates preconditions and returns specific error codes.

---

## 9. Notifications Are Interactive Destinations

Notifications deep-link to the relevant current state, not to a historical snapshot.

- Team invitation notification → Team Invitation (current state)
- Waitlist promotion → Event Detail (shows REGISTERED)
- Dispute resolution → Dispute detail (shows APPROVED/REJECTED)

**Source**: Notification `metadata.routing` contains `target` and `params` fields.

---

## 10. Primary Navigation Contains Destinations, Not Actions

The four primary nav items (Home, Campus, My Events, Profile) are places.

Actions live inside those places:
- Registration lives inside Event Detail (reached via Campus or My Events)
- Dispute lives inside Attendance History (reached via My Events)
- Invitation response lives inside the notification or Home

**Anti-pattern**: Adding "Register", "Check In", or "Dispute" to the primary navigation.

---

## 11. Asynchronous Changes Surface Through Context

When state changes happen while the student is elsewhere (waitlist promotion, team changes, dispute resolution), the student learns through:

1. **SSE → Notification bell** (immediate, if connected)
2. **Refetch on navigation** (standard, always works)
3. **Refetch on reconnect** (SSE `onopen` handler)

There is no polling. There is no guaranteed instant UI update for every state domain.

---

## 12. Club Membership Is Admin-Assigned

Students cannot self-join clubs. The backend has no student-facing "Join Club" endpoint. Club membership is managed by `CLUB_ADMIN` or `PLATFORM_ADMIN`.

The student UX displays:
- Club directory (read-only browsing)
- My Clubs (clubs they already belong to)
- Club events (browsable from Club Detail)
