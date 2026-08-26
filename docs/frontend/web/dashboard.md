# Dashboard Screen

## 1. Identity
- **Screen ID**: WEB-02
- **Screen Name**: Dashboard
- **Platform**: Web
- **Implementation Status**: PLANNED
- **Specification Status**: READY
- **Canonical Source Documents**: `11-dashboard-landing-pages.md`, `02-api-routing-matrix.md`, `DATA_CONTRACT.md`

## 2. Route
- **Route**: `/(app)/dashboard`

## 3. Role / Access
- **Visibility**: Authenticated User
- **Access**: Any
- **UX Guard**: Redirect to `/login` if unauthenticated.

## 4. Entry Points
- Successful Login -> `/(app)/dashboard`
- Global Sidebar Navigation -> "Dashboard" (or "Home")

## 5. Exit / Navigation
- **Quick Action Clicks**: Navigates to corresponding feature (e.g., `/(app)/events`, `/(app)/approvals`).

## 6. Layout Hierarchy
```text
AppShell
├── SidebarNavigation (bg: #111827)
│   └── ContextSwitcher
├── TopBar
└── MainContent (bg: #F9FAFB, padding: space-6)
    ├── BreadcrumbTrail
    ├── h1 (text: "Dashboard", color: #111827, typography: text-2xl, font-weight: 600)
    └── div (layout: grid)
        ├── Card (bg: #FFFFFF, border: rounded-lg)
        │   ├── h2 (text: "Upcoming Events")
        │   └── [Widget Content]
        ├── Card (bg: #FFFFFF, border: rounded-lg)
        │   ├── h2 (text: "Pending Approvals")
        │   └── [Widget Content]
        └── Card (bg: #FFFFFF, border: rounded-lg)
            ├── h2 (text: "My Clubs Summary")
            └── [Widget Content]
```

## 7. Component Map
- **SidebarNavigation**: Context tree (from 01-component-inventory)
- **ContextSwitcher**: Role dropdown (from 01-component-inventory)
- **BreadcrumbTrail**: Routing trace (from 01-component-inventory)
- **Card**: Widget wrapper (from 07-component-library)

## 8. Content / Data Map
| UI Element | Source Field | Source Type | Format / Transformation |
|---|---|---|---|
| "Upcoming Events" Widget | `title`, `startTime` | string, string ISO | Response shape from GET /v1/dashboard/summary. |
| "Pending Approvals" Widget | `title` | string | Response shape from GET /v1/dashboard/summary. |
| "My Clubs Summary" Widget | `name`, `member_count` | string, number | Response shape from GET /v1/dashboard/summary. |

## 9. API Map

## 10. UI States
- **Loading**: Render `Card` components containing skeleton blocks (bg: `#F3F4F6` pulsing).
- **Empty**: Render empty states inside respective `Card` widgets (e.g., "No upcoming events").
- **Error**: Standard Error Boundary per widget. Text: "Failed to load widget." Action: "Retry" button.
- **Unauthorized/Forbidden**: Redirects instantly per UX Guard.

## 11. Interaction Specification
- **Trigger**: Click on a widget item or Quick Action.
- **Action**: Navigate to the deep-linked resource (e.g. `/(app)/event-detail`).

## 12. Form Specification
- **Not Applicable** (This is a read-only aggregation screen).

## 13. Responsive / Adaptation
- **Desktop (>=1024px)**: Grid layout (e.g., 2 or 3 columns) for widgets. Sidebar pinned.
- **Tablet (768px - 1023px)**: Sidebar collapses. Widget grid reduces to 2 columns.
- **Mobile (<768px)**: Single column stack for widgets.

## 14. Accessibility
- **Role**: `main` for page content.
- **Keyboard Interaction**: `Tab` through widgets and Quick Actions.
- **Focus**: Distinct focus ring on interactive elements.

## 15. Motion
- **Animation**: Default platform transitions.

## 16. Security
- Only fields returned by the respective API endpoints are rendered.

## 17. Cache / Server State
- **Query Key**: `['dashboard', 'summary']`
- **Stale Behavior**: N/A until GET /v1/dashboard/summary exists.
- **Invalidation**: N/A until GET /v1/dashboard/summary exists.

## 18. Acceptance Criteria
1. Screen mounts and fetches data necessary for widgets.
2. Loading skeleton is visible during network requests.
3. Widgets accurately render mapped fields from the APIs.
4. Role-specific widgets render correctly (e.g. "Pending Approvals" only renders if user has permissions to view them).
5. Quick Actions successfully navigate to their target routes.

## 19. Specification Gaps / Open Decisions
- **None**. The gap BE-CONFIRMED-001 has been fully implemented in the backend (Phase 21J).
