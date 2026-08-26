# Notifications Screen

## 1. Identity
- **Screen ID**: WEB-07
- **Screen Name**: Notifications
- **Platform**: Web
- **Implementation Status**: PLANNED
- **Specification Status**: SPECIFICATION COMPLETE
- **Canonical Source Documents**: `DATA_CONTRACT.md`, `02-api-routing-matrix.md`

## 2. Route
- **Route**: `/(app)/notifications`

## 3. Role / Access
- **Visibility**: Authenticated User
- **Access**: Any
- **UX Guard**: Redirect to `/login` if unauthenticated.

## 4. Entry Points
- Global Sidebar Navigation -> "Notifications" (Bell Icon).

## 5. Exit / Navigation
- **Click Notification**: Navigates to target deep link (e.g., `/(app)/events/[id]`) based on the notification type/body.

## 6. Layout Hierarchy
```text
AppShell
├── SidebarNavigation (bg: #111827)
├── TopBar
└── MainContent (bg: #F9FAFB, padding: space-6)
    ├── BreadcrumbTrail
    ├── div (Header)
    │   └── h1 (text: "Notifications")
    └── div (layout: stack)
        └── Card (Notification Item)
            ├── div (Unread Indicator Dot)
            ├── h3 (Title)
            ├── p (Body)
            └── span (createdAt Timestamp)
```

## 7. Component Map
- **SidebarNavigation**: Context tree (from 01-component-inventory)
- **Card**: Item wrapper (from 07-component-library)

## 8. Content / Data Map
| UI Element | Source Field | Source Type | Format / Transformation |
|---|---|---|---|
| Title | `title` | string | Raw Text |
| Body | `body` | string | Raw Text |
| Timestamp | `createdAt` | string ISO | Relative time formatting (e.g., "2 hours ago") |
| Unread State | `readAt` | string ISO (nullable) | If `readAt` is null, show unread indicator (blue dot/bold text) |

## 9. API Map
- **Method**: `GET`
- **Path**: `/v1/notifications`
- **Purpose**: Fetch user notifications.
- **Consumed Fields**: `id`, `type`, `title`, `body`, `readAt`, `createdAt`

- **Method**: `PATCH`
- **Path**: `/v1/notifications/:id/read`
- **Purpose**: Mark a specific notification as read (updates `readAt`).

## 10. UI States
- **Loading**: Skeleton stack items.
- **Empty**: "No notifications yet."
- **Error**: Standard Error Boundary.

## 11. Interaction Specification
- **Trigger**: Click Notification.
- **Action**: Fire `PATCH /v1/notifications/:id/read` mutation (if `readAt` is currently null), then navigate to relevant screen based on `type`.

## 12. Form Specification
- **Not Applicable**.

## 13. Responsive / Adaptation
- **Desktop (>=1024px)**: Constrained width list centered or left-aligned.
- **Tablet/Mobile**: Full width list.

## 14. Accessibility
- **Role**: `list` for the notification wrapper, `listitem` for each card.

## 15. Motion
- **Animation**: Default transitions.

## 16. Security
- Endpoint must enforce that a user can only fetch and mutate their own notifications.

## 17. Cache / Server State
- **Query Key**: `['notifications']`
- **Stale Behavior**: 5 minutes (or handled via push/SSE invalidation if applicable).
- **Invalidation**: Update local cache optimistically on `PATCH` to set `readAt = now()`.

## 18. Acceptance Criteria
- AC-WEB-07-01: Successfully fetches and displays list from `GET /v1/notifications`.
- AC-WEB-07-02: Correctly derives unread state from `readAt == null`.
- AC-WEB-07-03: Clicking an unread notification fires `PATCH /v1/notifications/:id/read`.

## 19. Specification Gaps / Open Decisions
- **DEFERRED DEPENDENCY (ANNOUNCEMENTS)**: The global `Announcements` feature (CRUD operations for admins, and fetching for users) is explicitly deferred to V2 (BE-CONFIRMED-011). This specification is restricted *strictly* to `Notifications` (personal alerts). Any "Announcements" tab or UI section within this screen must be omitted.
- **OPEN UX ASSUMPTION**: The deep-linking behavior on click (navigating to a specific screen based on notification `type`) is standard UX, but the specific routing map for each `type` string (e.g., `"EVENT_APPROVED"` -> `/(app)/events/[id]`) is not explicitly defined in the design docs. This logic will need to be fleshed out during implementation.
