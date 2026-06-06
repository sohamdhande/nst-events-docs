# Dashboard Components

Built exclusively using Next.js and Tailwind CSS.

## Navigation & Structure
* **SidebarNavigation**: Dynamic, role-based context tree.
* **ContextSwitcher**: Dropdown mapping to `active_campus_id` or club context.
* **CommandPalette**: Cmd+K overlay for global macros.
* **BreadcrumbTrail**: Reflects flat routing paths.
* **PageHeader**: Title, contextual actions, and active filters.

## Data Management
* **DataTable**: Virtualized list supporting sorting, pagination, and multi-select.
* **FilterBar**: URL-state driven filter controls.
* **DetailDrawer**: Slide-out panel for entity edits without losing table context.

## Administration
* **BulkActionBar**: Appears when DataTable items are selected (e.g., "Approve 5 Events").
* **ImpersonationBanner**: High-visibility yellow banner warning the Platform Admin they are acting as another user.
* **AuditLogDiffViewer**: Code-diff style viewer for comparing `old_record` and `new_record` JSON.
* **RoleAssignmentPanel**: Admin UI for elevating users to Club Admins.

## Workflow
* **ApprovalQueueItem**: Dedicated card for Faculty Mentors with Approve/Reject/Feedback actions.
* **AnalyticsCard**: High-level KPI display.

## Operations & Governance
* **AttendanceDisputeQueue**: Review list for failed check-in evidence.
* **ClubLeadershipTransferPanel**: Multi-step flow for transferring Club Admin role.
* **CompetitionResultsEntry**: Batch form for assigning winners.
* **CompetitionResultsReview**: Audit surface for Faculty.
