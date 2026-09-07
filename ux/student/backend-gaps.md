# Student UX Backend Gaps

This document captures backend limitations that affect the student UX but cannot be resolved without backend changes.

---

## 1. Club Self-Join

| Field | Value |
|---|---|
| **Problem** | Students cannot request to join a club through the UI. |
| **Current Backend Behavior** | Club membership is managed exclusively by `CLUB_ADMIN` or `PLATFORM_ADMIN` via `POST /clubs/:id/members`. There is no student-facing join or request-to-join endpoint. |
| **Required UX** | Ideally, a student browsing Club Detail could request membership or express interest. |
| **Why Current Behavior Is Insufficient** | The student sees clubs in the directory but has no way to become a member without out-of-band communication with a club admin. |
| **Relevant Source** | `apps/api/src/modules/clubs/clubs.router.ts` — `POST /:id/members` requires `canManageClubMembers` with `CLUB_ADMIN` role. |
| **Possible Backend Change** | Add a `POST /clubs/:id/join-request` endpoint or a self-join flow if club settings allow it. |
| **Current UX Adaptation** | The student UX does NOT present a "Join Club" button. My Clubs shows only clubs the student already belongs to. |

---

## 2. Deep-Link Destination Preservation on Login

| Field | Value |
|---|---|
| **Problem** | When an unauthenticated user clicks a deep link (e.g., `/events/:id`), the current auth callback always redirects to `/dashboard`. |
| **Current Backend Behavior** | `auth.router.ts` → `res.redirect(303, \`${env.WEB_APP_URL}/dashboard\`)` — hardcoded redirect to `/dashboard` after OAuth callback. |
| **Required UX** | After login, the student should land on the page they originally tried to access. |
| **Why Current Behavior Is Insufficient** | A student clicking a shared event link while logged out loses context and must manually navigate back to the event. |
| **Relevant Source** | `apps/api/src/modules/auth/auth.router.ts:79` |
| **Possible Backend Change** | Store the original destination in the OAuth state parameter or a session cookie and redirect to it after callback. |
| **Current UX Adaptation** | The documentation notes that deep-link preservation is desired but not currently implemented. The student lands on Home after login. |

---

## 3. Dispute Resolution Notification to Student

| Field | Value |
|---|---|
| **Problem** | The notification type `ATTENDANCE_DISPUTE_RESOLVED` exists in the enum, but it must be verified that the actual dispute resolution service calls `enqueueNotification` to the student. |
| **Current Backend Behavior** | `resolveAttendanceDispute` in `attendance.service.ts` handles resolution, but the notification dispatch to the student depends on the implementation. |
| **Required UX** | The student should receive a notification when their dispute is approved or rejected. |
| **Relevant Source** | `apps/api/src/modules/attendance/attendance.service.ts` — `resolveAttendanceDispute` function. |
| **Possible Backend Change** | Verify and ensure `enqueueNotification` is called with type `ATTENDANCE_DISPUTE_RESOLVED` targeting the disputing student. |
| **Current UX Adaptation** | The documentation assumes this notification exists. If it doesn't fire, the student discovers the resolution only by revisiting Attendance History. |

---

## Status

All gaps identified are non-blocking for initial student UX implementation. The product works without these capabilities — they represent quality-of-life improvements.

No gaps require inventing new backend states or contradicting existing behavior.
