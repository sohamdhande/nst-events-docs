# Attendance Result Workflow

## Overview
Attendance is captured externally (via a separate physical or mobile client). The Student Web Application is responsible for reflecting the resulting state and managing exceptions.

## Flow
```mermaid
sequenceDiagram
    participant S as Student
    participant UI as Event Detail / History
    participant API as GET /attendance
    
    S->>UI: View Attendance Status
    UI->>API: Fetch current status
    API-->>UI: Record (PRESENT / ABSENT / EXCUSED)
    
    alt PRESENT
        UI->>S: "Attendance recorded"
    else ABSENT
        UI->>S: "No attendance record"
        opt If Eligible
            S->>UI: Click "Report an issue"
            UI-->>S: Open dispute workflow
        end
    else EXCUSED
        UI->>S: "Excused"
    end
```

## Backend References
- **Routes**: `GET /users/me/attendance`
- **Models**: `AttendanceRecord`, `AttendanceDispute`
