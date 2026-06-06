# Sidebar Architecture

## Header Zone
Displays the active `active_campus_id` context and the primary User/Role switcher.

## Primary Navigation Zone
Dynamic list of routes generated strictly based on the resolved JWT/DB permissions of the active role.
* Examples: "Events", "Approvals", "Analytics".

## Footer Zone
Persistent links for User Settings, Help, and Logout.

## Collapse Behavior
The sidebar supports a collapsed state (icon-only mode) to maximize the content canvas during data-heavy operations.

## Mobile Behavior
On mobile web, the sidebar hides behind a hamburger menu and acts as an off-canvas overlay.

## State Management
Sidebar active states are derived entirely from the URL path.
