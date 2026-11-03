# Mobile Leaderboard Screen (DECOMMISSIONED)

> **Status: Removed**. We have decided not to build student or club leaderboards. This screen and its routes have been completely removed from the application.

## 1. Identity
- **Screen ID**: MOB-13
- **Screen Name**: Leaderboard
- **Platform**: Mobile
- **Implementation Status**: DECOMMISSIONED / REMOVED
- **Specification Status**: DECOMMISSIONED
- **Canonical Source Documents**: `DATA_CONTRACT.md`, `02-api-routing-matrix.md`

## 2. Route
- **Route**: `/(app)/leaderboard` (REMOVED)

## 3. Role / Access
- **Visibility**: Authenticated User
- **Access**: Any
- **UX Guard**: Redirect to Auth if unauthenticated.

## 4. Entry Points
- Tab bar navigation (Leaderboard tab).

## 5. Exit / Navigation
- **Click Student**: Navigates to student profile (future phase, currently no-op).
- **Click Club**: Navigates to `/(app)/clubs/[id]`.

## 6. Layout Hierarchy
```text
Screen (bg: #F9FAFB)
├── Header (text: "Leaderboard")
├── TabSwitcher (Students | Clubs)
└── Swiper/Pager
    ├── Slide 1 (Students List)
    │   └── FlatList
    │       └── Card (Student Rank Item)
    │           ├── span (Rank Number)
    │           ├── img (Avatar - placeholder)
    │           ├── h3 (Display Name)
    │           └── div (Points breakdown: Total, Attendance, Contribution, Competition)
    └── Slide 2 (Clubs List)
        └── FlatList
            └── Card (Club Rank Item)
                ├── span (Rank Number)
                ├── h3 (Club Name)
                ├── p (Stats: event_count, member_count)
                └── span (Total Points)
```

## 7. Component Map
- **TabSwitcher**: Context switcher (from component library)
- **FlatList**: Native scrollable list
- **Card**: Item wrapper

## 8. Content / Data Map
| UI Element | Source Field | Source Type | Format / Transformation |
|---|---|---|---|
| Student Name | `display_name` | string | Raw Text (fallback to placeholder if null) |
| Total Points (Student) | `total_points` | number | `{total_points} pts` |
| Points Breakdown | `attendance_points`, `contribution_points`, `competition_points` | number | Display as micro-stats row. (Note: These will currently return 0 per BE-CONFIRMED-013) |
| Club Name | `club_name` | string | Raw Text |
| Total Points (Club) | `total_points` | number | `{total_points} pts` |
| Club Stats | `event_count`, `member_count` | number | `{event_count} Events • {member_count} Members` |

## 9. API Map
- **Method**: `GET`
- **Path**: `/v1/leaderboard/students`
- **Purpose**: Fetch top students.
- **Consumed Fields**: `userId`, `display_name`, `total_points`, `attendance_points`, `contribution_points`, `competition_points`, `last_refreshed_at`
- **Status**: Exists, but breakdown points are stubbed (BE-CONFIRMED-013).

- **Method**: `GET`
- **Path**: `/v1/leaderboard/clubs`
- **Purpose**: Fetch top clubs.
- **Consumed Fields**: `clubId`, `club_name`, `total_points`, `event_count`, `member_count`, `last_refreshed_at`
- **Status**: Exists.

## 10. UI States
- **Loading**: Skeleton items for both lists.
- **Empty**: "No leaderboard data available yet."
- **Error**: Standard error boundary with retry.

## 11. Interaction Specification
- **Trigger**: Swipe left/right or tap Tabs.
- **Action**: Switch between Students and Clubs lists.

## 12. Form Specification
- **Not Applicable**.

## 13. Responsive / Adaptation
- **Mobile**: Native full screen lists.

## 14. Accessibility
- **Role**: `list`.
- **Keyboard/VoiceOver**: Read rank and points clearly.

## 15. Motion
- **Animation**: Smooth horizontal slide between tabs.

## 16. Security
- Read-only data.

## 17. Cache / Server State
- **Query Key**: `['leaderboard', 'students']`, `['leaderboard', 'clubs']`
- **Stale Behavior**: 5 minutes.
- **Invalidation**: Pull to refresh.

## 18. Acceptance Criteria
- AC-MOB-13-01: Displays students sorted by total points.
- AC-MOB-13-02: Displays clubs sorted by total points.
- AC-MOB-13-03: The UI should handle `attendance_points`, `contribution_points`, and `competition_points` gracefully when they return as 0.

## 19. Specification Gaps / Open Decisions
- **DEFERRED DEPENDENCY (POINTS BREAKDOWN)**: While the endpoints exist, the specific point breakdown fields (`attendance_points`, `contribution_points`, `competition_points`) are intentionally hardcoded to 0 in the database materialized view, deferred to Milestone 2 (BE-CONFIRMED-013). The frontend should be built to expect these fields as 0.
