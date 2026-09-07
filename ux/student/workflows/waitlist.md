# Waitlist Promotion Workflow

## Overview
Waitlist promotion is a fully automated backend process. The UX challenge is communicating this asynchronous state change to the student without requiring them to act.

## Flow
```mermaid
sequenceDiagram
    participant S as Student
    participant API as Worker / DB
    participant SSE as Realtime Stream
    participant UI as Student UX

    Note over API: Registered student cancels
    API->>API: NotificationJob triggered
    API->>API: Promotes next Waitlisted student to Registered
    
    API->>SSE: Publish Waitlist Promotion Event
    SSE-->>UI: Live Update
    
    UI->>S: Toast Notification "Promoted off waitlist!"
    UI->>UI: Increment Notification Bell
    
    Note over S,UI: Next time student visits My Events
    UI->>API: GET /users/me/registrations
    API-->>UI: Event is now REGISTERED
    UI->>S: Event displays in 'Upcoming' tab instead of 'Waitlist'
```

## Backend References
- **Models**: `NotificationJob`, `EventRegistration`
- **Constraints**: 
  - There is no "Accept" endpoint for waitlists.
  - Waitlist promotion requires no student action.
  - The SSE stream (`/sse/notifications/live`) powers the immediate toast notification.
