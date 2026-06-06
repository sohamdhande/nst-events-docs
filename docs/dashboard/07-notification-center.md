# Notification Center

The Notification Center is strictly for operational system alerts.

## Notification Types
* Waitlist Promotions
* Approval Requests (to Faculty)
* Attendance Alerts
* Role Changes
* System Events

## Notification Lifecycle
Generated via backend triggers, delivered via WebSockets/Push, marked as read, and eventually archived.

## Deep Links & Action Routing
Clicking a notification immediately navigates the dashboard shell. E.g., clicking "Pending Approval" routes to `/approvals/[event_id]`.

## Priority Rules
Urgent actions (Approvals) float to the top and trigger unread badges.
