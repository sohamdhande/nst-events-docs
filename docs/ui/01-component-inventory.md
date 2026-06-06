# UI Component Inventory

This is the master inventory of all UI components across the NST-Events platform.

## Navigation
* **SidebarNavigation**: Dashboard. Core context tree.
* **ContextSwitcher**: Dashboard. Role/Club dropdown.
* **CommandPalette**: Dashboard. Global search/action overlay.
* **BottomNavigationItem**: Mobile. Tab bar item.
* **ActionSheetMenu**: Mobile. Bottom drawer menu.
* **BreadcrumbTrail**: Dashboard. Flat routing trace.

## Data & Display
* **DataTable**: Shared. High-density data grid.
* **DetailDrawer**: Dashboard. Slide-out entity viewer.
* **AuditLogDiffViewer**: Dashboard. Visual diffs for administrative audits.
* **Carousel**: Mobile. Horizontal swipeable list.

## Forms & Inputs
* **TextInput**, **PasswordInput**, **TextArea**: Shared.
* **SelectDropdown**, **DatePicker**, **DateRangePicker**, **TimePicker**: Shared.
* **ParticipantRoleSelector**: Dashboard. Inline dropdown for `ATTENDEE` to `SPEAKER` upgrades.
* **ClubAssociationSelector**: Dashboard. Multi-club event ownership mapper.
* **RoleAssignmentPanel**: Dashboard. Faculty management of club admins.
* **CapacityControl**: Dashboard. Numeric counter for event limits.
* **RichTextEditor**, **FileUploadDropzone**: Dashboard.

## Events & Attendance\n* **AttendanceDisputeForm**: Mobile. Dispute entry.\n* **AttendanceDisputeQueue**: Dashboard. Admin review.\n* **ClubLeadershipTransferPanel**: Dashboard. Handover workflow.
* **EventCard**, **FeaturedEventCard**: Mobile.
* **AttendanceCheckInWidget**: Mobile. Swipe-to-checkin UI.
* **LocationValidationIndicator**: Mobile. GPS verification status.
* **DynamicQRCodeRenderer**: Shared. TOTP rendering.

## Leaderboard & Analytics
* **TopPerformerCard**, **AchievementBadge**, **ProgressToNextRank**: Mobile.
* **CompetitionResultsTable**: Dashboard. Entry form for `WINNER`, `TOP_10`.
* **LeaderboardPodium**: Shared. Visual placement.
* **TrendCard**, **BarChart**, **AttendanceHeatmap**, **DonutChart**: Dashboard.

## Operations Mode
* **LiveScannerOverlay**: Dashboard. Distraction-free QR scanning.
* **ManualOverrideToggle**: Dashboard. Bypassing QR checks.
* **LiveMetricHUD**: Dashboard. Real-time attendance capacity.

## Admin
* **BulkActionBar**: Dashboard. Multi-select table actions.
* **ImpersonationBanner**: Dashboard. Platform Admin debugging indicator.
