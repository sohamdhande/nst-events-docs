# User Management Screen

## 1. Identity
- **Screen ID**: WEB-14
- **Screen Name**: User Management
- **Platform**: Web
- **Implementation Status**: PLANNED
- **Specification Status**: READY
- **Canonical Source Documents**: `DATA_CONTRACT.md`, `02-api-routing-matrix.md`

## 2. Route
- **Route**: `/(app)/admin/users`

## 3. Role / Access
- **Visibility**: PLATFORM_ADMIN
- **Access**: PLATFORM_ADMIN
- **UX Guard**: Redirect to `/dashboard` if lacking permissions.

## 4. Entry Points
- From `WEB-11` (Admin Hub).

## 5. Exit / Navigation
- **Back**: Returns to Admin Hub.

## 6. Layout Hierarchy
```text
SidebarNavigation (bg: #111827)
├── div (bg: #F9FAFB, padding: space-6)
│   ├── BreadcrumbTrail
│   ├── div (Header)
│   │   ├── h1 (text: "User Management")
│   │   └── input (Search by Email)
│   └── table
│       ├── tr (header: Name, Email, Role, Actions)
│       └── tr (user row)
│           ├── td (fullName)
│           ├── td (email)
│           ├── td (globalRole badge)
│           └── td (Button: Change Role)
│   └── div (Pagination)
│       └── button (Load More)
```

## 7. Component Map
- **SidebarNavigation**: Context tree (from 01-component-inventory)
- **Table**: Data display
- **Badge**: Role visualizer (STUDENT, FACULTY_ADMIN, PLATFORM_ADMIN)
- **Button**: Action trigger
- **Input**: Search field

## 8. Content / Data Map
| UI Element | Source Field | Source Type | Format / Transformation |
|---|---|---|---|
| Name | `fullName` | string | Raw Text (fallback to "Unknown" if null) |
| Email | `email` | string | Raw Text |
| Role | `globalRole` | enum | UPPERCASE (STUDENT, FACULTY_ADMIN, PLATFORM_ADMIN) |

## 9. API Map
- **Method**: `GET`
- **Path**: `/v1/admin/users`
- **Purpose**: Fetch users for management list.
- **Status**: **IMPLEMENTED**. (Phase 21J)

- **Method**: `POST`
- **Path**: `/v1/admin/users/:id/role`
- **Purpose**: Adjust a user's global role.
- **Status**: **IMPLEMENTED**. (Phase 21J)

## 10. UI States
- **Loading**: Skeleton rows for the table.
- **Empty**: N/A (Admin must exist).
- **Error**: Error boundary.
- **Error State**: Show global alert banner if the user fetch fails.

## 11. Interaction Specification
### Role Mutation
- **Trigger**: Click "Change Role"
- **Action**: Opens a modal to select a new `global_role`. Submit fires `POST` mutation to update role.

### Pagination
1. **Initial page fetch**: Retrieves the first page of users with a defined `limit`.
2. **`next_cursor` detection**: If the response contains `pagination.next_cursor != null`, the "Load More" control becomes visible.
3. **Load More visibility**: Only rendered when a valid `next_cursor` is present.
4. **Load More click behavior**: Triggers a fetch for the next page of users.
5. **Next-cursor request**: The API call explicitly uses `cursor=<next_cursor>`.
6. **Append behavior**: Successful responses append the newly fetched users to the existing list.
7. **Duplicate prevention**: Existing users are not duplicated in the list.
8. **Loading/disabled behavior**: The "Load More" button is disabled and displays a loading state while the request is pending.
9. **Terminal condition**: When the new fetch returns `pagination.next_cursor == null`, the "Load More" control disappears.
10. **Next-page error behavior**: If the fetch fails, existing users remain visible, and the global error state safely reports the failure without erasing the table.

## 12. Form Specification
- **Role Change Modal** (Inferred UX requirement for the role change action):
  - Field: `globalRole` (Select dropdown).
  - Values: `STUDENT`, `FACULTY_ADMIN`, `PLATFORM_ADMIN`.

## 13. Responsive / Adaptation
- **Desktop (>=1024px)**: Full table display.
- **Tablet/Mobile**: Horizontal scroll on table or stacked card layout.

## 14. Accessibility
- **Role**: `main`.

## 15. Motion
- **Animation**: Default table transitions.

## 16. Security
- Route and API must strictly validate `PLATFORM_ADMIN`.

## 17. Cache / Server State
- **Query Key**: `['admin-users', searchFilter]`
- **Stale Behavior**: N/A until endpoint exists.
- **Invalidation**: N/A.

## 18. Acceptance Criteria
- AC-WEB-14-01: Table rendering cannot be verified until `GET /v1/admin/users` is implemented (BE-CONFIRMED-005).
- AC-WEB-14-02: Role mutation cannot be verified until `POST /v1/admin/users/:id/role` is implemented (BE-CONFIRMED-006).

## 19. Specification Gaps / Open Decisions

- **OPEN UX ASSUMPTION**: I have included a "Search by Email" input and a "Change Role" modal because these are standard UX patterns for user management tables, but they are not explicitly sourced from a specific UI mockup. If filtering by email is not supported by the future `GET /v1/admin/users` endpoint, the search input must be removed.
