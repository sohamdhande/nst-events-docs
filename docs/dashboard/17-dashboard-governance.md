# Dashboard Governance Rules

## Two-Click Rule
No critical operational workflow may exceed a depth of `Sidebar → Screen → Action`. 

## Command Parity Rule
Every meaningful UI action (e.g., creating an event, approving an event) MUST have an equivalent executable action within the Command Palette.

## URL State Rule
Filters, Tabs, Sorting, and View Modes MUST persist in URL parameters (e.g., `?tab=attendance&sort=desc`). Reloading the page must restore the exact visual state.

## State Isolation Rule
Role changes must invoke a full React component re-mount. No permission leakage is tolerated between contexts.

## Multi-Campus Rule
No campus-specific hardcoded logic may exist inside dashboard components. All dashboard rendering must strictly derive from `active_campus_id`.
