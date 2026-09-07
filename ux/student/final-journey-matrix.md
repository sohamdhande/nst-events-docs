# Final Student Journey Matrix

This matrix maps every critical student journey to its backend mutations, screens, edge cases, and realtime behavior.

---

## Individual Registration

| Field | Value |
|---|---|
| **Journey** | Student registers for an individual event |
| **Starting State** | Unauthenticated or authenticated, not registered |
| **Entry Point** | Home → Event, Discover → Event, Club → Event, Notification → Event |
| **Steps** | 1. View Event Detail → 2. Click Register → 3. Confirm (dialog) → 4. Backend responds → 5. State updates in-place |
| **Screens** | Event Detail |
| **Backend Mutation** | `POST /events/:id/register` → `register_event` RPC |
| **Realtime Events** | None (synchronous response) |
| **Notifications** | None |
| **Edge Cases** | Capacity race → WAITLISTED; Event locked → Error; Duplicate → Idempotent success |
| **End State** | Event Detail shows REGISTERED or WAITLISTED |
| **Interaction Count** | 3 (View → Register → Confirm) |
| **Optimization** | No separate page. Result appears in-place. No redirect. |

---

## Cancel Registration

| Field | Value |
|---|---|
| **Journey** | Student cancels their registration |
| **Starting State** | REGISTERED |
| **Entry Point** | Event Detail, My Events → Event Detail |
| **Steps** | 1. View Event Detail → 2. Click Cancel Registration → 3. Confirm (dialog) → 4. Backend responds → 5. State updates |
| **Screens** | Event Detail |
| **Backend Mutation** | `DELETE /events/:id/register` → `cancel_registration` RPC |
| **Realtime Events** | May trigger WAITLIST_PROMOTED for another student |
| **Notifications** | None to the cancelling student |
| **Edge Cases** | Event locked → Error; Already cancelled → Error |
| **End State** | Event Detail shows NOT REGISTERED |
| **Interaction Count** | 3 |

---

## Waitlist → Automatic Promotion

| Field | Value |
|---|---|
| **Journey** | Student is promoted from waitlist |
| **Starting State** | WAITLISTED |
| **Entry Point** | Passive (system-initiated) |
| **Steps** | 1. Another student cancels → 2. Backend promotes → 3. SSE notification arrives → 4. Student opens notification or My Events → 5. State is REGISTERED |
| **Screens** | Notifications, My Events, Event Detail |
| **Backend Mutation** | None by the student |
| **Realtime Events** | `WAITLIST_PROMOTED` via SSE `/notifications/live` |
| **Notifications** | "You are off the waitlist!" |
| **Edge Cases** | SSE disconnected → Reconciled on reconnect via refetch |
| **End State** | REGISTERED across all screens |
| **Interaction Count** | 0 (automatic) |

---

## Team Creation & Invitation

| Field | Value |
|---|---|
| **Journey** | Student creates a team and invites members |
| **Starting State** | Not on a team |
| **Entry Point** | Event Detail (team event) |
| **Steps** | 1. View Event Detail → 2. Click Create Team → 3. Enter team name → 4. Backend creates team (FORMING) → 5. Search invitees → 6. Send invitations → 7. Wait for acceptances |
| **Screens** | Event Detail, Team |
| **Backend Mutations** | `POST /events/:id/teams`, `POST /teams/:id/invitations` |
| **Realtime Events** | None to leader (except notification when invitee responds) |
| **Notifications** | `TEAM_INVITATION_ACCEPTED` / `TEAM_INVITATION_DECLINED` to leader |
| **Edge Cases** | Team full (max reached) → Invite disabled; Event locked → Create fails |
| **End State** | Team in FORMING state with members |
| **Interaction Count** | 5+ (Create → Name → Confirm → Search → Invite) |

---

## Accept Team Invitation

| Field | Value |
|---|---|
| **Journey** | Student accepts a team invitation |
| **Starting State** | Has pending invitation |
| **Entry Point** | Notification → Invitation, Home → Pending Actions |
| **Steps** | 1. View invitation → 2. Click Accept → 3. Backend responds → 4. Joined as member |
| **Screens** | Team Invitation, Team |
| **Backend Mutation** | `POST /teams/:id/invitations/:invId/accept` → `accept_invitation` RPC |
| **Realtime Events** | `TEAM_INVITATION_ACCEPTED` to leader; possibly `TEAM_REGISTERED` or `TEAM_WAITLISTED` to all members |
| **Notifications** | To leader: "A member accepted" |
| **Edge Cases** | Team full → HTTP 422; Expired → Error; Event locked → Error |
| **End State** | Student is a team member; team state may transition to REGISTERED or WAITLISTED |
| **Interaction Count** | 2 (View → Accept) |

---

## Decline Team Invitation

| Field | Value |
|---|---|
| **Journey** | Student declines a team invitation |
| **Starting State** | Has pending invitation |
| **Entry Point** | Notification → Invitation, Home → Pending Actions |
| **Steps** | 1. View invitation → 2. Click Decline → 3. Backend responds |
| **Screens** | Team Invitation |
| **Backend Mutation** | `POST /teams/:id/invitations/:invId/decline` |
| **Realtime Events** | `TEAM_INVITATION_DECLINED` to leader |
| **Edge Cases** | Already expired → Error |
| **End State** | Invitation marked DECLINED |
| **Interaction Count** | 2 |

---

## View Attendance Status

| Field | Value |
|---|---|
| **Journey** | Student checks attendance status after an event |
| **Starting State** | Event attended (or missed) |
| **Entry Point** | My Events → Past → Event, Event Detail |
| **Steps** | 1. Open Event Detail or Attendance History → 2. View status |
| **Screens** | Event Detail, Attendance History |
| **Backend Mutation** | None (read-only: `GET /users/me/attendance`) |
| **Edge Cases** | No record yet → "No attendance record"; Event archived |
| **End State** | Student sees PRESENT, ABSENT, or No Record |
| **Interaction Count** | 1–2 |

---

## Submit Attendance Dispute

| Field | Value |
|---|---|
| **Journey** | Student disputes an attendance record |
| **Starting State** | ABSENT or No Record, within dispute window |
| **Entry Point** | Attendance History → Report Issue |
| **Steps** | 1. View attendance → 2. Click Report Issue → 3. Provide reason/evidence → 4. Submit → 5. Backend creates dispute |
| **Screens** | Attendance History, Dispute form |
| **Backend Mutation** | `POST /attendance/disputes` |
| **Realtime Events** | None |
| **Notifications** | None to student (student polls or revisits) |
| **Edge Cases** | Window expired → CTA hidden; Duplicate → Error; Validation failure → Form error |
| **End State** | Dispute PENDING |
| **Interaction Count** | 4 |

---

## Dispute Resolution

| Field | Value |
|---|---|
| **Journey** | Student receives dispute resolution |
| **Starting State** | Dispute PENDING |
| **Entry Point** | Notification → Dispute, Attendance History |
| **Steps** | 1. Receive notification → 2. View dispute → 3. See APPROVED or REJECTED |
| **Backend Mutation** | None by student |
| **Realtime Events** | `ATTENDANCE_DISPUTE_RESOLVED` via SSE |
| **Notifications** | "Your dispute has been resolved" |
| **Edge Cases** | SSE disconnected → Discovered on next navigation |
| **End State** | Attendance updated to EXCUSED (if approved) or remains ABSENT (if rejected) |
| **Interaction Count** | 1–2 |

---

## Discover a Club

| Field | Value |
|---|---|
| **Journey** | Student browses clubs and finds an upcoming event |
| **Starting State** | Authenticated |
| **Entry Point** | Campus → Clubs |
| **Steps** | 1. Browse clubs → 2. Open Club Detail → 3. See upcoming events → 4. Open Event Detail |
| **Screens** | Clubs, Club Detail, Event Detail |
| **Backend Mutation** | None |
| **Edge Cases** | No clubs → Empty state; No upcoming events for club → Empty state |
| **End State** | Student on Event Detail |
| **Interaction Count** | 3 |

---

## Follow Notification

| Field | Value |
|---|---|
| **Journey** | Student receives and follows a notification to its destination |
| **Starting State** | Notification received (SSE or on next visit) |
| **Entry Point** | Notification Bell → Notification |
| **Steps** | 1. See bell badge → 2. Open notifications → 3. Click notification → 4. Navigate to destination |
| **Backend Mutation** | `PATCH /notifications/:id/read` (mark read) |
| **Realtime Events** | SSE delivers notification instantly if connected |
| **Edge Cases** | Destination unavailable → Fallback route (`metadata.routing.fallback`) |
| **End State** | Student on the relevant current-state screen |
| **Interaction Count** | 3 |

---

## Returning Student (Home)

| Field | Value |
|---|---|
| **Journey** | Returning student opens the app and understands what matters |
| **Starting State** | Authenticated, has existing registrations/invitations |
| **Entry Point** | Home |
| **Steps** | 1. Open app → 2. Home shows prioritized items → 3. Act on most relevant item |
| **Screens** | Home |
| **Backend Mutation** | None (read-only surface) |
| **Edge Cases** | No items → Empty state with discovery CTA; Session expired → Login |
| **End State** | Student navigates to the most relevant context |
| **Interaction Count** | 1–2 |

---

## Mutation Retry Safety Matrix

| Mutation | Backend Endpoint | Idempotent? | Safe to Retry? | Duplicate Resolution |
|---|---|---|---|---|
| Register | `POST /events/:id/register` | Yes (via RPC) | Yes | Returns existing state |
| Cancel Registration | `DELETE /events/:id/register` | Yes (soft delete) | Yes | No-op if already cancelled |
| Accept Invitation | `POST /teams/:id/invitations/:invId/accept` | Yes (via RPC) | Yes | Returns existing membership |
| Decline Invitation | `POST /teams/:id/invitations/:invId/decline` | No (checks PENDING) | Conditional | Error if already declined |
| Leave Team | `DELETE /teams/:id/leave` | Yes (via RPC) | Yes | No-op if already left |
| Transfer Leadership | `POST /teams/:id/transfer-leadership` | No | Conditional | Error if already transferred |
| Submit Dispute | `POST /attendance/disputes` | No (unique constraint) | No — show existing | Error returned, UI shows existing dispute |
| Update Preferences | `PATCH /notifications/preferences` | Yes | Yes | Last write wins |
