# Club Directory & Membership Workflow

## Overview
Students can view the Club Directory on Campus. "My Clubs" on the Profile is a read-only list populated by admin assignment.

## Flow
```mermaid
sequenceDiagram
    participant S as Student
    participant UI as Campus -> Clubs
    participant API as Clubs API

    S->>UI: Navigate to Clubs
    UI->>API: GET /clubs
    API-->>UI: List of ACTIVE clubs
    UI->>S: Show Club Cards
    
    S->>UI: Click a Club
    UI->>API: GET /clubs/:id
    API-->>UI: Club details + event list
    UI->>S: Show Club Profile
    
    S->>UI: Navigate to Profile -> My Clubs
    UI->>API: GET /users/me
    API-->>UI: Includes club_memberships array
    UI->>S: Display assigned clubs
```

## Backend References
- **Routes**: `GET /clubs`, `GET /users/me`
- **Models**: `Club`, `ClubMembership`
- **Constraints**: Students cannot join a club via the API. `club_memberships` mutations are strictly admin-only.
