# Global Navigation

## Sidebar
The primary vertical navigation tree. Adapts immediately upon context switches.

## Horizontal Context Tabs
Used within complex views. For example, viewing an Event uses tabs for `Details`, `Registrations`, `Attendance`, and `Analytics`.

## Breadcrumbs
Breadcrumbs trace the routing history logically. Because of Multi-Club Event Ownership, breadcrumbs use flat routing: `Events / Hackathon 2026 / Attendance` rather than nesting under a specific club.

## Deep Linking
Every operational state must support deep linking. URL paths cleanly represent entities: `/events/[event_id]/attendance`.

## URL Strategy
The URL acts as the single source of truth for the View state. See Dashboard Governance URL State Rule.
