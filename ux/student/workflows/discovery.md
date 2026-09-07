# Event Discovery Workflow

## Overview
Students discover events through the `Campus` -> `Discover` tab. The backend exclusively surfaces events that are in the `PUBLISHED` state, protecting draft and pending events automatically via API filtering.

## Flow
```mermaid
sequenceDiagram
    participant S as Student
    participant UI as Discover Page
    participant API as GET /events

    S->>UI: Navigates to Campus -> Discover
    UI->>API: GET /events?visibility=PUBLIC
    API-->>UI: List of PUBLISHED events
    
    UI->>S: Displays Event Cards (chronological)
    
    S->>UI: Types "Hackathon" in Search
    UI->>API: GET /events?search=Hackathon (debounced)
    API-->>UI: Filtered TSVector results
    
    S->>UI: Selects "Technical" Club Filter
    UI->>API: GET /events?club_id=123
    API-->>UI: Filtered results
```

## Backend References
- **Routes**: `GET /events`
- **Models**: `Event`, `EventClub`
- **Query Params**: `search`, `club_id`, `state` (hardcoded to `PUBLISHED` for students on the frontend, also enforced by RLS).

## Event Card UI Requirements
- Display: Title, Primary Club, Start Date/Time, Location, Event Type.
- Contextual Tags: `TEAM` or `INDIVIDUAL`. `SOLD OUT` (if registration count >= max capacity).
