# Student Day-in-the-Life Journeys

This document simulates realistic end-to-end student scenarios, verifying that the complete product works as one coherent experience.

---

## Journey 1: New Student Discovers First Event

**Starting state**: Just logged in for the first time. No registrations. No clubs.

```text
Login
→ Home (empty: no upcoming events, no pending actions)
→ Home shows "Discover events" CTA
→ Campus → Discover
→ Browse available events
→ Open interesting event
→ Event Detail (Event info, capacity, registration CTA)
→ Click Register
→ Confirmation dialog ("Register for Workshop X?")
→ Confirm
→ Backend: POST /events/:id/register → REGISTERED
→ Event Detail updates in-place: "✓ Registered"
→ Optional: Click "View in My Events"
→ My Events → Upcoming tab shows the event
```

**Screens touched**: Home, Discover, Event Detail, My Events
**Backend mutations**: `register_event`
**Edge cases**: Capacity full → WAITLISTED; Event locked → Error
**Interaction count**: 5 (Home → Discover → Event → Register → Confirm)

---

## Journey 2: Student Registers for Team Event

**Starting state**: Authenticated. Sees a team event on Discover.

```text
Discover → Event Detail (team event)
→ "This is a team event" section
→ No team → "Create Team" / "Join Team" options
→ Click "Create Team"
→ Enter team name → Confirm
→ Backend: POST /events/:id/teams → team created (FORMING)
→ Team view appears: 1/4 members, leader = student
→ Search invitees
→ Backend: GET /events/:id/invitee-search?q=...
→ Select invitee → Send Invite
→ Backend: POST /teams/:id/invitations → invitation created
→ Backend sends TEAM_INVITATION_RECEIVED notification to invitee
→ Wait for acceptances
```

**Screens touched**: Discover, Event Detail, Team
**Backend mutations**: `create_team`, `inviteMember`
**Edge cases**: Event locked → Create fails; Invitee already on team → Error; Team full → Invite disabled
**Interaction count**: 7+

---

## Journey 3: Student Receives and Accepts Team Invitation

**Starting state**: Authenticated. Notification bell shows unread.

```text
→ Notification bell badge increments (SSE)
→ Open Notifications
→ See: "You have been invited to join team Alpha"
→ Click notification
→ Navigate to Team Invitation view
→ See: Who invited, Team name, Event name, Team size
→ Click Accept
→ Backend: POST /teams/:id/invitations/:invId/accept → accept_invitation RPC
→ Possible outcomes:
  - REGISTERED → Student is member, team registered
  - WAITLISTED → Student is member, team waitlisted
  - FORMING → Student is member, team still forming
→ Invitation view updates: "Accepted"
→ Navigate to Team to see full roster
```

**Screens touched**: Notifications, Team Invitation, Team
**Backend mutations**: `accept_invitation`
**Edge cases**: Team full → 422 error; Expired → Error; Event locked → Error
**Interaction count**: 3 (Notification → Invitation → Accept)

---

## Journey 4: Student Is Waitlisted and Automatically Promoted

**Starting state**: WAITLISTED for an event (individual or team).

```text
→ Another student cancels their registration
→ Backend: cancel_registration RPC promotes next waitlisted
→ Backend: enqueueNotification(WAITLIST_PROMOTED)
→ SSE: pg_notify delivers to student's /notifications/live channel
→ useRealtimeNotifications: invalidates ['notifications'] query
→ Notification bell increments
→ Student opens Notifications
→ See: "You are off the waitlist!"
→ Click notification → Event Detail
→ Event Detail shows: ✓ REGISTERED
→ My Events: event moved from Waitlist tab to Upcoming tab
```

**Screens touched**: Notifications, Event Detail, My Events
**Backend mutations**: None by student
**There is no "Accept Promotion" step.**
**Edge cases**: SSE disconnected → Reconciled on reconnect; Tab inactive → Discovered on next navigation

---

## Journey 5: Student Checks Attendance After Event

**Starting state**: Event has concluded. Student attended physically.

```text
→ Open My Events → Past tab
→ See past event with attendance indicator
→ Open Event Detail
→ Attendance section shows: "✓ Attendance recorded"
OR
→ Open Attendance History (via Profile or My Events)
→ See attendance record: PRESENT, event name, date
```

**Screens touched**: My Events, Event Detail OR Attendance History
**Backend mutations**: None (GET /users/me/attendance)
**The Web App does NOT perform QR scanning. It reads the result.**

---

## Journey 6: Student Finds Incorrect Attendance

**Starting state**: Event concluded. Student attended but is marked ABSENT or has no record.

```text
→ Open Attendance History
→ See: "No attendance record" or "Absent" for the event
→ Click "Report an Issue" (if within dispute window)
→ Dispute form opens
→ Provide reason and optional evidence
→ Submit
→ Backend: POST /attendance/disputes → dispute created (PENDING)
→ Form updates: "Under review"
→ Wait for admin resolution
→ Eventually: ATTENDANCE_DISPUTE_RESOLVED notification
→ Open dispute or Attendance History
→ See: APPROVED → status changed to EXCUSED
  OR
→ See: REJECTED → status remains ABSENT
```

**Screens touched**: Attendance History, Dispute form, Notifications
**Backend mutations**: `submitAttendanceDispute`
**Edge cases**: Window expired → CTA hidden; Duplicate → Error shows existing; Validation failure → Form error

---

## Journey 7: Student Discovers a Club

**Starting state**: Authenticated. No clubs.

```text
→ Campus → Clubs
→ Browse club directory
→ Open interesting club
→ Club Detail: description, members, upcoming events
→ See upcoming event from this club
→ Click event → Event Detail
→ Register if interested (standard registration flow)
```

**Screens touched**: Clubs, Club Detail, Event Detail
**Backend mutations**: None (browsing) or `register_event`
**Note**: Students cannot self-join clubs. Membership is admin-assigned.

---

## Journey 8: Student Opens Notification and Follows It

**Starting state**: Multiple unread notifications.

```text
→ Notification bell shows badge count
→ Open Notifications
→ See chronological list with unread items styled distinctly
→ Click "Your team is now registered" notification
→ Navigate to Event Detail (via metadata.routing.target = 'event_details')
→ See team registration status: REGISTERED
→ If destination is unavailable: fallback to /events (metadata.routing.fallback)
```

**Screens touched**: Notifications, Event Detail
**Backend mutations**: Mark notification read
**Edge cases**: Destination gone → Fallback route; Event archived → Shows historical state

---

## Journey 9: Returning Student Opens Home

**Starting state**: Has existing registrations, pending invitation, and unread notifications.

```text
→ Open app → Home
→ See prioritized sections:
  1. Pending Actions (team invitation)
  2. Upcoming Events (next registered event)
  3. Discovery suggestions
→ Act on most relevant: Accept team invitation
→ OR navigate to upcoming event
→ OR browse new events
```

**Screens touched**: Home, then contextual destination
**Backend mutations**: Depends on action taken
**Edge cases**: No items → Empty state with "Discover events" CTA; Session expired → Login

---

## Cross-Journey Edge Case Verification

Every journey above survives:

| Edge Case | Resolution |
|---|---|
| Capacity race | Backend returns WAITLISTED; UI updates in-place |
| Event lock during action | Backend rejects; UI shows error |
| Duplicate action | Backend is idempotent; returns existing state |
| Team full during acceptance | Backend returns 422; UI shows unavailable |
| Expired invitation | Backend returns error; UI shows expired state |
| Session expiry | Login redirect; return to original destination |
| Network failure | Error message; safe to retry for idempotent mutations |
| SSE disconnected | Reconciled on reconnect via onopen refetch |
| Stale data on tab return | Refetched on navigation |
| Resource unavailable | Clear message + safe navigation back |
