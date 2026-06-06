# Context Switching

## Role Switching
A user with multiple roles (e.g., Student and Club Admin) utilizes a dropdown to switch their operational context.

## Club Switching
A user who is an admin for multiple clubs switches the active club context, dynamically rebuilding the sidebar and dashboard widgets.

## Campus Switching
Driven by `active_campus_id`. (See Dashboard Governance).

## State Isolation & Security Boundaries
Role changes invoke a total component unmount and re-mount. 
No state or cache leaks between contexts. Switching from "Club Admin" to "Student" absolutely destroys the administrative UI state to prevent permission leakage.
