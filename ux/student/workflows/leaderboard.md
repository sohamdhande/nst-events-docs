# Leaderboard Workflow

## Overview
The leaderboard exposes the student's relative ranking based on their participation and competition achievements.

## Flow
```mermaid
sequenceDiagram
    participant S as Student
    participant UI as Leaderboard Page
    participant API as GET /leaderboard/students

    S->>UI: Navigate to Campus -> Leaderboard
    UI->>API: GET /leaderboard/students
    
    Note over API: Queries PostgreSQL Materialized View
    
    API-->>UI: Paginated student ranks + Own Rank object
    UI->>S: Display Global Top 50
    
    UI->>S: Sticky bottom bar showing "My Rank" and "My Points"
```

## Backend References
- **Routes**: `GET /leaderboard/students`, `GET /leaderboard/clubs`
- **Models**: `LeaderboardScore` (but queried via Materialized Views for performance).
- **Constraints**: Point accumulation details (the "why") are abstracted behind the total score on the frontend to avoid slow aggregate joins.
