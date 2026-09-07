# Student UX Edge-Case Matrix

This document maps all edge cases, race conditions, and error states for the Student Web Application, based directly on backend API behavior and database capabilities.

| Domain | Scenario | Initial State | Trigger | Backend Result | Student-Facing State | Affected Screens | Primary Action | Recovery | Realtime / Notifications |
|---|---|---|---|---|---|---|---|---|---|
| **Registration** | Capacity Race | Event Detail (Open) | Student clicks Register while another student takes final spot | `WAITLISTED` (via `register_event`) | `Waitlisted` | Event Detail, My Events | Wait for promotion | None required | - |
| **Registration** | Event Locked Race | Event Detail (Open) | Student clicks Register but event was locked by Admin | HTTP Error (`EVENT_LOCKED`) | `Registration Unavailable` | Event Detail | Dismiss error | - | - |
| **Registration** | Duplicate Request | Submitting | Student clicks Register twice | Idempotent Success | `Registered` | Event Detail | View My Events | - | - |
| **Waitlist** | Auto Promotion | Waitlisted | Another student cancels | `REGISTERED` | `Registered` | Event Detail, My Events | View Details | - | `WAITLIST_PROMOTED` Notification |
| **Team Invitation**| Capacity Race | Pending Invite | Student accepts invite, but team just became full | HTTP 422 Error | `Expired / Unavailable` | Team Invitation | Dismiss | Ask leader to manage space | - |
| **Team Invitation**| Duplicate Invite | Not Member | Student clicks accept twice | Idempotent Success | `Accepted` (Member) | Team Invitation, My Events| View Team | - | - |
| **Team Invitation**| Event Ends | Pending Invite | Student accepts after event ends | HTTP Error (`EVENT_LOCKED`) | `Expired / Unavailable` | Team Invitation | Dismiss | - | - |
| **Disputes** | Duplicate Submission| No Record | Student clicks Submit twice | Unique constraint / Idempotent | `Pending / Under Review`| Dispute, Attendance History| View Resolution | - | - |
| **Disputes** | Window Expired | No Record | Student tries to submit after `dispute_window_expires_at` | HTTP 403 Forbidden | CTA Disappears / Error | Attendance History | None | - | - |
| **Attendance** | Backend Marking | Absent | Student is scanned externally | `PRESENT` | `Attendance Recorded` | Event Detail, Attendance History | View Details | - | - |
| **Network** | Offline Mutation | Authenticated | Student registers offline | Request fails | `Network Error` | Current Screen | Retry | Safe to retry (backend is idempotent) | - |
| **Session** | Token Expired | Authenticated | Student performs action with expired session | HTTP 401 Unauthorized | `Session Expired` | Current Screen | Log in | Original intent preserved via redirect if supported | - |
| **Data Stale** | Capacity Changed | Event Detail (Open) | Capacity fills while tab is inactive | N/A (UI shows open) | (Changes on action) | Event Detail | Register (will result in WAITLISTED) | Handled gracefully by Capacity Race | - |
| **Auth/Links** | Protected Link | Unauthenticated| User clicks deep link to Team Invitation | HTTP 401 | `Login Required` | Login | Google Sign In | Redirect to Team Invitation on success | - |
