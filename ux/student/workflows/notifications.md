# Notifications Workflow

## Overview
Notifications route students to actionable destinations.

## Routing Flow
```mermaid
flowchart LR
    Bell[Click Notification] --> Type{Notification Type}
    
    Type -- WAITLIST_PROMOTED --> EventDetail[Event Detail Page]
    Type -- TEAM_INVITATION --> Home[Home Dashboard Pending Actions]
    Type -- ATTENDANCE_OPEN --> EventDetail
    Type -- DISPUTE_RESOLVED --> DisputeDetail[Dispute Details Page]
    Type -- CLUB_ANNOUNCEMENT --> ClubDetail[Club Profile]
```

## Interaction Flow
```mermaid
sequenceDiagram
    participant S as Student
    participant UI as Notification Dropdown
    participant API as Notifications API

    S->>UI: Open Bell
    UI->>API: GET /notifications
    API-->>UI: List of notifications
    
    S->>UI: Click unread notification
    UI->>API: PATCH /notifications/:id/read
    UI->>S: Navigate to target destination
```

## Backend References
- **Routes**: `GET /notifications`, `PATCH /notifications/:id/read`
- **Models**: `Notification`
- **Structure**: The `metadata` JSON field on the Notification model contains the routing information (e.g., `eventId`, `teamId`).
