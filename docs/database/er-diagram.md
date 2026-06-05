# Complete ER Diagram

```mermaid
erDiagram
    USERS ||--o{ CLUB_MEMBERSHIPS : joins
    CLUBS ||--o{ CLUB_MEMBERSHIPS : has
    CLUBS ||--o{ EVENT_CLUBS : sponsors
    EVENTS ||--o{ EVENT_CLUBS : associated_with
    EVENTS ||--o{ EVENT_SESSIONS : contains
    EVENTS ||--o{ EVENT_REGISTRATIONS : has_individual
    EVENTS ||--o{ TEAMS : hosts
    TEAMS ||--o{ TEAM_MEMBERS : includes
    USERS ||--o{ TEAM_MEMBERS : forms
    USERS ||--o{ EVENT_REGISTRATIONS : books
    EVENT_SESSIONS ||--o{ ATTENDANCE_RECORDS : tracks
    USERS ||--o{ ATTENDANCE_RECORDS : marked_present
    USERS ||--o{ NOTIFICATIONS : receives
    USERS ||--o{ LEADERBOARD_SCORES : earns
    USERS ||--o{ AUDIT_LOGS : triggers
```

## Relationship Explanations
* **USERS to CLUB_MEMBERSHIPS**: A user can hold different roles across different clubs. This is the core RBAC lookup.
* **EVENTS to EVENT_SESSIONS**: Every event has at least one session. Multi-day workshops will have multiple sessions. Attendance is always linked to a session.
* **EVENTS to TEAMS**: Registration can either be individual (`event_registrations` tied directly to user) or team-based.\n