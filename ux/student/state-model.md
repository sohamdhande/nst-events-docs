# Canonical Student State Model

This document establishes the single source of truth for Student UX states, derived directly from the backend implementation (Prisma schema and modules).

## 1. Registration States
Derived from `RegistrationStatus` in the backend.

| Backend State | Student-Facing Term | Trigger | Meaning |
|---|---|---|---|
| `REGISTERED` | Registered | Explicit confirmation | The student has a confirmed spot. |
| `WAITLISTED` | Waitlisted | Explicit confirmation when full | The student is in the queue for a spot. |
| `CANCELLED` | Cancelled | Explicit cancellation | The student voluntarily gave up their spot. |

**Important Rules**:
- Registration is a direct interaction inside Event Detail, not a separate page.
- Promotion from `WAITLISTED` to `REGISTERED` is **automatic** when capacity opens. There is no student "accept promotion" step.

## 2. Waitlist Lifecycle
- **Trigger**: Student attempts to register when capacity is full and waitlisting is allowed.
- **State**: `WAITLISTED`
- **Promotion Trigger**: System event (e.g. another student cancels).
- **Promotion Result**: State changes to `REGISTERED`. A notification (`WAITLIST_PROMOTED`) is generated.

## 3. Team States
Derived from `TeamStatus` and `ParticipationRole`.

| Backend State | Student-Facing Term | Trigger | Meaning |
|---|---|---|---|
| `FORMING` | Forming | Team created | Team exists but is not yet finalized/registered. |
| `REGISTERED` | Registered | Team finalized/registered | Team has a confirmed spot in the event. |
| `WAITLISTED` | Waitlisted | Team registered when full | Team is in the queue for a spot. |
| `CANCELLED` | Cancelled | Team cancelled | Team registration was cancelled. |

**Roles**: 
- **Leader**: Created the team (has management rights).
- **Member**: Accepted an invitation.

## 4. Team Invitations
Derived from `InvitationStatus`.

| Backend State | Student-Facing Term | Trigger | Meaning |
|---|---|---|---|
| `PENDING` | Pending | Invitation sent | Awaiting response from invitee. |
| `ACCEPTED` | Accepted | Invitee accepts | Invitee becomes a team Member. |
| `DECLINED` | Declined | Invitee declines | Invitation is rejected. |
| `CANCELLED` | Cancelled | Leader revokes | Invitation is withdrawn. |
| `EXPIRED` | Expired | Time elapses | Invitation is no longer valid. |

## 5. Attendance States
Derived from `AttendanceStatus`.
*Note: The Student Web App is for viewing status and managing exceptions ONLY. It does NOT contain a QR scanner.*

| Backend State | Student-Facing Term | Trigger | Meaning |
|---|---|---|---|
| `PRESENT` | Attendance recorded | Scan/System/Manual | Student attended the session. |
| `ABSENT` | Absent | Manual/System mark | Explicitly marked absent by admin/system. |
| *(No Record)* | No attendance record | Default state | Student has not been checked in yet. |
| `EXCUSED` | Excused | Approved dispute | Student attendance is formally excused (from dispute). |

## 6. Dispute States
Derived from `DisputeStatus`.

| Backend State | Student-Facing Term | Trigger | Meaning |
|---|---|---|---|
| `PENDING` | Pending / Under review | Dispute filed | Waiting for administrative review. |
| `APPROVED` | Approved | Admin approves | Dispute resolved favorably (attendance updated). |
| `REJECTED` | Rejected | Admin rejects | Dispute denied. |

## 7. Event Lifecycle States
Derived from `EventState`.
*Note: These are administrative states, not student participation states.*

| Backend State | Student-Facing Term | Meaning |
|---|---|---|
| `PUBLISHED` | Available / Open | Event is visible and active for students. |
| `CANCELLED` | Cancelled | Event was cancelled by administrators. |
| `ARCHIVED` | Past Event | Event has concluded and is read-only. |

**Event Lock State**: `isLocked` (Boolean) - Prevents further changes or registrations.

## 8. Club Membership
Derived from `ClubRole`.

| Backend State | Student-Facing Term | Trigger | Meaning |
|---|---|---|---|
| `MEMBER` | Member | Admin assignment | Student belongs to the club. |
| `CORE_MEMBER` | Core Member | Admin assignment | Student is a core club member. |
| `CLUB_ADMIN` | Club Admin | Admin assignment | Student helps run the club. |

*Self-join is not currently supported by the backend.*
