# My Events Workflow

## Overview
My Events serves as the student's personal commitment log. It filters `EventRegistration` records into logical, actionable groups.

## Flow
```mermaid
sequenceDiagram
    participant S as Student
    participant UI as My Events Page
    participant API as GET /users/me/registrations

    S->>UI: Navigate to My Events
    UI->>API: Fetch Registrations
    API-->>UI: List of registrations + Event data
    
    UI->>UI: Filter into Tabs
    
    Note over UI: Upcoming Tab
    UI->>S: Display REGISTERED events (End Date > Now)
    
    Note over UI: Waitlist Tab
    UI->>S: Display WAITLISTED events (End Date > Now)
    
    Note over UI: Past Tab
    UI->>S: Display all events where End Date < Now, regardless of status
```

## Backend References
- **Routes**: `GET /users/me/registrations`
- **Models**: `EventRegistration`
- **Actions available from here**: 
  - Cancel Registration (If Upcoming/Waitlisted and Unlocked)
  - File Dispute (If Past and eligible)
  - View Attendance History (If Past)
