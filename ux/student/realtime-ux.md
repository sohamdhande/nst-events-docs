# Student Realtime UX

This document defines how Server-Sent Events (SSE) and backend background workers affect the student UI without requiring a manual page refresh.

## Implemented Realtime Mechanisms

### SSE: `/notifications/live`
- **Frontend**: `useRealtimeNotifications` hook establishes an EventSource connection.
- **Reconnection**: Native EventSource auto-reconnect. On reconnect, the `onopen` handler invalidates the `['notifications']` query to reconcile any missed notifications.
- **Heartbeat**: Backend sends heartbeat every 30 seconds.
- **Auth**: Token passed via query parameter (EventSource limitation).

### SSE: `/events/:id/live`
- **Frontend**: `useEventLiveUpdates` hook (if connected to a specific event page).
- **Purpose**: Pushes event state changes (lock, attendance session open/close) to listeners on Event Detail.

---

## 1. Waitlist Promotion
- **Backend trigger**: `cancel_registration` RPC promotes next waitlisted student and calls `enqueueNotification` with type `WAITLIST_PROMOTED`.
- **SSE delivery**: `pg_notify` fires `NOTIFICATION_CREATED` event to the user's notification channel.
- **UI consequence**:
  1. The `useRealtimeNotifications` hook receives the event and invalidates `['notifications']`.
  2. Notification bell badge increments.
  3. If the user navigates to My Events, the event appears in the Upcoming tab (via refetch).
- **Fallback**: If SSE was disconnected, the `onopen` reconciliation refetches notifications on reconnect.

## 2. Event Lock / State Changes
- **Backend trigger**: Admin locks/unlocks event, or event end-time passes.
- **SSE delivery**: `/events/:id/live` stream pushes state change.
- **UI consequence**:
  1. If the user is on Event Detail, registration CTAs immediately become disabled.
  2. A `LOCKED` indicator appears.
- **Fallback**: If not connected to the event SSE channel, the lock state is discovered on next navigation to Event Detail (standard GET refetch).

## 3. Attendance Session State
- **Backend trigger**: Admin creates or closes an attendance session.
- **SSE delivery**: `/events/:id/live` stream pushes session open/close event.
- **UI consequence**:
  1. For registered students viewing Event Detail, the attendance status section updates.
  2. The Web App does NOT present a QR scanner. It informs the student that attendance is open.
- **Fallback**: Standard refetch on navigation.

## 4. Team Invitations
- **Backend trigger**: Leader sends invitation, generating `TEAM_INVITATION_RECEIVED` notification.
- **SSE delivery**: `/notifications/live` pushes `NOTIFICATION_CREATED`.
- **UI consequence**:
  1. Notification bell increments.
  2. If on Home, the pending actions section can surface the invitation on next render/refetch.
- **Fallback**: Standard notification list refetch on navigation.

## 5. Team State Changes
- **Backend trigger**: Member joins/leaves, team is registered/waitlisted/cancelled.
- **SSE delivery**: Relevant notification types (`TEAM_REGISTERED`, `TEAM_WAITLISTED`, `TEAM_CANCELLED`, etc.) pushed via `/notifications/live`.
- **UI consequence**:
  1. Notification bell increments.
  2. Targeted query invalidation: `useRealtimeNotifications` invalidates `['events']` on team state change notifications.
- **Fallback**: Refetch on navigation to Event Detail or My Events.

## 6. Dispute Resolution
- **Backend trigger**: Admin resolves dispute, generating `ATTENDANCE_DISPUTE_RESOLVED` notification.
- **SSE delivery**: `/notifications/live` pushes notification.
- **UI consequence**:
  1. Notification bell increments.
  2. Attendance History reflects the new state on next navigation/refetch.
- **Fallback**: Standard refetch on navigation.

---

## What Does NOT Have Realtime Support

| Domain | Mechanism | Reason |
|---|---|---|
| Leaderboard | Standard REST polling / refresh | Uses PostgreSQL materialized views; no SSE channel. |
| Discover (event list) | Standard REST polling / refresh | Aggregate endpoint; no per-user SSE channel. |
| Club directory | Standard REST polling / refresh | No SSE channel implemented. |
| Attendance History | Refetch on navigation | No dedicated SSE channel for attendance records. |
