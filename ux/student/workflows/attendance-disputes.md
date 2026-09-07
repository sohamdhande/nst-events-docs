# Attendance Disputes Workflow

## Overview
If a student misses a check-in due to a system error or valid exception, they can file a dispute within a specific time window.

## Flow
```mermaid
sequenceDiagram
    participant S as Student
    participant UI as Dispute Form
    participant API as POST /attendance/disputes

    S->>UI: Click "File Dispute" on Past Event
    UI->>UI: Display Form (Reason, Evidence)
    
    S->>UI: Fill text + Upload Image
    UI->>UI: Client uploads image to CDN
    UI->>API: POST /attendance/disputes (reason, [evidenceUrls])
    
    API-->>UI: Status: PENDING
    UI->>S: Show "Under Review" state
    
    Note over API: ...Later, Admin Approves...
    
    API->>S: Notification "Dispute Approved"
    
    S->>UI: View Past Event
    UI->>API: GET /users/me/attendance
    API-->>UI: Record Status: EXCUSED
    UI->>S: Display "Excused" badge
```

## Backend References
- **Routes**: `POST /attendance/disputes`, `GET /attendance/disputes`
- **Models**: `AttendanceDispute`
- **Constraints**: Evidence must be uploaded to a bucket/CDN by the frontend before sending the URLs to the backend. The backend stores string URLs in `evidence_urls`.
