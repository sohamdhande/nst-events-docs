# Student Capability Map

This document maps all student capabilities to their underlying backend implementations.

## Event Discovery & Exploration
| Capability | Backend Endpoint / State | Constraints |
|------------|--------------------------|-------------|
| View Upcoming Events | `GET /events` | Returns only `PUBLISHED` events. Respects visibility (PUBLIC/PRIVATE) and audience rules. |
| Search/Filter Events | `GET /events` (query params) | Supports TSVector search, club filters, and date filters. |
| View Event Detail | `GET /events/:id` | Returns metadata, capacity, lock status, and club relation. |

## Registration (Individual)
| Capability | Backend Endpoint / State | Constraints |
|------------|--------------------------|-------------|
| View own status | `GET /events/:id/my-registration` | Returns `REGISTERED`, `WAITLISTED`, or `null`. |
| Register for Event | `POST /events/:id/register` | Subject to `max_capacity`. Event must be unlocked. |
| Cancel Registration| `DELETE /events/:id/register` | Event must be unlocked. |
| Waitlist Promotion | Backend Queue (`Worker`) | Automatic. Student cannot trigger manually. |

## Registration (Team)
| Capability | Backend Endpoint / State | Constraints |
|------------|--------------------------|-------------|
| Create Team | `POST /events/:id/teams` | Creator becomes leader. |
| Invite Member | `POST /teams/:id/invitations` | Invitee receives notification. |
| Accept/Decline | `POST /teams/:id/invitations/:invId/...`| Only invitee can accept. |
| View Invites | `GET /users/me/team-invitations` | Returns `PENDING` invitations. |
| Transfer Leader| `POST /teams/:id/transfer-leadership` | Leader only. |
| Leave Team | `DELETE /teams/:id/leave` | Event must be unlocked. |

## Attendance
| Capability | Backend Endpoint / State | Constraints |
|------------|--------------------------|-------------|
| View Status | `GET /users/me/attendance` | Returns records (Status: `PRESENT`/`EXCUSED`/`ABSENT`). |
| Dispute | `POST /attendance/disputes` | Student reports issue if eligible. |

## Disputes
| Capability | Backend Endpoint / State | Constraints |
|------------|--------------------------|-------------|
| File Dispute | `POST /attendance/disputes` | Requires reason string and optional evidence URLs. |
| View Disputes | `GET /attendance/disputes` | Returns `PENDING`, `APPROVED`, `REJECTED`. |

## Clubs & Leaderboard
| Capability | Backend Endpoint / State | Constraints |
|------------|--------------------------|-------------|
| Discover Clubs | `GET /clubs` | Shows active clubs. |
| View Club | `GET /clubs/:id` | Includes club events and members. |
| View Leaderboard| `GET /leaderboard/students` | Returns points and rank derived from materialized views. |

## Notifications
| Capability | Backend Endpoint / State | Constraints |
|------------|--------------------------|-------------|
| View Unread | `GET /notifications` | |
| Mark Read | `PATCH /notifications/:id/read` | |
| Preferences | `GET/PATCH /notifications/preferences`| Edit push/email toggles. |
