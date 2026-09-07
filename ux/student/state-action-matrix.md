# Student State → Action Matrix

This matrix maps each meaningful student state to the precise actions available, derived directly from the underlying backend capabilities.

## Event Availability States

| State | Primary Action | Secondary Actions | Unavailable Actions | Where state appears |
|---|---|---|---|---|
| **PUBLISHED (Open)** | Register | - | - | Event Detail |
| **LOCKED / UNAVAILABLE**| (Action Disabled) | - | Register | Event Detail |
| **ARCHIVED / ENDED** | View History | - | Register | Event Detail |

## Event Registration States (Individual)

| State | Primary Action | Secondary Actions | Unavailable Actions | Where state appears |
|---|---|---|---|---|
| **NOT REGISTERED** | Register | Join Waitlist (if full) | Cancel registration | Event Detail |
| **REGISTERED** | View event / attendance status | Cancel registration | Register again | Event Detail, My Events |
| **WAITLISTED** | None (wait for automatic promotion) | Cancel registration | Register again, manually accept promotion (automatic only) | Event Detail, My Events |

## Team Registration States

| State | Primary Action | Secondary Actions | Unavailable Actions | Where state appears |
|---|---|---|---|---|
| **FORMING** (Leader) | Invite members | Cancel team | Register (until ready) | Event Detail (Team flow), My Events |
| **FORMING** (Member) | None (wait for leader) | Leave team | Invite members, Register | Event Detail (Team flow), My Events |
| **REGISTERED** | View event / attendance status | Cancel team (Leader) / Leave (Member) | Register again | Event Detail (Team flow), My Events |
| **WAITLISTED** | None (wait for automatic promotion) | Cancel team (Leader) / Leave (Member) | Register again, manually accept promotion (automatic only) | Event Detail (Team flow), My Events |

## Team Invitation States

| State | Primary Action | Secondary Actions | Unavailable Actions | Where state appears |
|---|---|---|---|---|
| **PENDING** | Accept | Decline | Ask to join | Home, Notifications, Team Invitation |
| **ACCEPTED** | View Team | - | Accept again, Decline | Team Invitation |
| **DECLINED** | Dismiss | - | Accept | Team Invitation |
| **EXPIRED** | Dismiss | - | Accept, Decline | Team Invitation |

## Attendance States

| State | Primary Action | Secondary Actions | Unavailable Actions | Where state appears |
|---|---|---|---|---|
| **No attendance record** | Report an issue (if eligible) | - | Check in manually | Event Detail, Attendance History |
| **Absent** | Report an issue | - | Check in | Event Detail, Attendance History |
| **Attendance recorded** | View Details | Report an issue (if disputed) | Check in again | Event Detail, Attendance History |
| **Excused** | View Details | - | Report an issue | Event Detail, Attendance History |

## Dispute States

| State | Primary Action | Secondary Actions | Unavailable Actions | Where state appears |
|---|---|---|---|---|
| **Eligible** | Report an issue | - | View resolution | Attendance History, Event Detail |
| **PENDING** | View dispute | - | Report another issue | Attendance History, Dispute |
| **APPROVED** | View resolution | - | Report another issue | Attendance History, Dispute |
| **REJECTED** | View resolution | - | Report another issue | Attendance History, Dispute |
