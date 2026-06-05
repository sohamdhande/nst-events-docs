# Event Data Model

## Tables
* **`events`**: Base generic table with `metadata jsonb`.
* **`event_clubs`**: Mapping table resolving M:N ownership.
* **`event_approvals`**: Tracks the workflow history of the event.
* **`event_media`**: (If decoupled from metadata) references Supabase Storage bucket URLs.

## State Lifecycle
The `event_status` enum drives the entire UI.

1. `DRAFT`: Club Admin is writing the description and uploading banners.
2. `PENDING_APPROVAL`: Submitted. Locked from Club Admin edits. Appears on Faculty Mentor dashboard.
3. `PUBLISHED`: Faculty approved. Event is visible on the student feed and accepts registrations.
4. `ARCHIVED`: Event date has passed. Locked from further attendance scans or registrations. Read-only for historical analytics.
