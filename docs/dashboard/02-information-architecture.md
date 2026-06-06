# Information Architecture

## Overview Domain
High-level summaries, cross-club analytics, and global notifications.

## Organization Domain
Entity management. Profiles for Clubs, core membership rosters, and faculty assignments.

## Operations Domain
The core workspace. Event creation, draft management, approval queues, and live Operations Mode.

## Communication Domain
Separated strictly into:
* **Notification Center**: Operational alerts (approvals, role changes).
* **Announcement Center**: Broadcast messages (campus notices).

## Administration Domain
Platform-level settings, audit log review, global user management.

## Entity Relationships: Multi-Club Event Ownership
**CRITICAL**: The architecture does NOT model `Club → Event`. It models `Event ↔ Multiple Clubs`.
* **Example**: A Hackathon is co-hosted by the Developer Club and the AI Club.
* **Impact**: Breadcrumbs and navigation paths reflect the multi-ownership model. An event is accessed via `/events/[id]`, not nested under `/clubs/[id]/events/[id]`.
