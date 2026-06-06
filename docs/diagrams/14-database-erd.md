# 14 High-Level ERD

This Entity-Relationship Diagram outlines the core database entities and their relationships within the PostgreSQL database.

```mermaid
erDiagram
    USERS ||--o{ ROLES : "has"
    USERS ||--o{ CLUB_MEMBERSHIPS : "belongs to"
    CLUBS ||--o{ CLUB_MEMBERSHIPS : "has"
    
    CLUBS ||--o{ EVENTS : "hosts primary"
    CLUBS ||--o{ EVENT_CLUBS : "associated with"
    EVENTS ||--o{ EVENT_CLUBS : "has associated"
    
    EVENTS ||--o{ EVENT_REGISTRATIONS : "has"
    USERS ||--o{ EVENT_REGISTRATIONS : "registers for"
    
    EVENTS ||--o{ ATTENDANCE_SESSIONS : "has"
    ATTENDANCE_SESSIONS ||--o{ ATTENDANCE_RECORDS : "generates"
    USERS ||--o{ ATTENDANCE_RECORDS : "marks"
    
    ATTENDANCE_RECORDS ||--o{ ATTENDANCE_DISPUTES : "can have"
    
    USERS ||--o{ NOTIFICATIONS : "receives"
    
    USERS ||--|| LEADERBOARD : "has rank"
```
