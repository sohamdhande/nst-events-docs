# Event System Architecture

## Core Data Model
To maintain simplicity and query performance across the global feed, all events utilize a **Generic Event Model**.
Instead of separate tables for Hackathons, Workshops, etc., we use a single `events` table.

* **Standard Columns**: `id`, `title`, `description`, `start_time`, `end_time`, `location_geofence`. (Note: Events connect to clubs via the `event_clubs` join table for M:N relationships. At least one club must be attached to every event.)
* **Dynamic Typing**: `event_type` (Enum: WORKSHOP, HACKATHON, GUEST_LECTURE, etc.)
* **Flexible Metadata**: `metadata` (JSONB)
  * *Example for Hackathon*: `{"team_size_min": 2, "team_size_max": 4}`
  * *Example for Workshop*: `{"prerequisites": ["Laptop", "Python 3 installed"]}`

## Event Approval Workflow & Lifecycle
Every event transitions through a strict state machine to ensure faculty oversight and quality control.

### States:
1. **DRAFT**: The event is being created/edited by a Core Member or Club Admin. It is not visible to anyone else.
2. **PENDING_APPROVAL**: The Club Admin submits the event. It is locked for editing and awaits Faculty Mentor review.
3. **PUBLISHED**: The Faculty Mentor approves the event. It becomes visible to the target audience (Public or Private).
4. **ARCHIVED**: The event has concluded and is locked from further attendance or registration changes.

### State Flow
`DRAFT` → `PENDING_APPROVAL` → `PUBLISHED` → `ARCHIVED`
