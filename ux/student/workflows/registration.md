# Event Detail & Registration Workflow

## Overview
The Event Detail page is the primary interaction point for committing to an activity. It adapts dynamically based on the student's registration status.

## Flow (Solo Registration)
```mermaid
sequenceDiagram
    participant S as Student
    participant UI as Event Detail
    participant API as Registration API

    S->>UI: View Event
    UI->>API: GET /events/:id/my-registration
    API-->>UI: null (Not registered)
    
    UI->>S: Display "Register" CTA
    S->>UI: Click "Register"
    UI->>UI: Show Confirmation Modal
    S->>UI: Confirm
    
    UI->>API: POST /events/:id/register
    
    alt Capacity Available
        API-->>UI: Status: REGISTERED
        UI->>S: Success Toast, CTA changes to "Cancel Registration"
    else Capacity Full
        API-->>UI: Status: WAITLISTED
        UI->>S: Info Toast, CTA changes to "Cancel Waitlist"
    else Event Locked
        API-->>UI: 422 Locked
        UI->>S: Error Toast "Registration closed"
    end
```

## Backend References
- **Routes**: `POST /events/:id/register`, `DELETE /events/:id/register`, `GET /events/:id/my-registration`
- **Constraints**: No optimistic UI. The UI must wait for the POST request to return before mutating the displayed status.

## UI Requirements
- Explicit confirmation modal required before submitting the POST request.
- If `is_locked` is true, hide/disable the Register and Cancel buttons completely.
