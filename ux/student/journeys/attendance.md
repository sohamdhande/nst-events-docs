# Attendance & Dispute Journey

This journey maps the student experience for checking attendance outcomes and filing a dispute when an error occurs.

*Note: The physical check-in and QR scanning occur externally. The Web App reflects the results.*

```mermaid
flowchart TD
    Start[Event Day: Session Concludes]
    Home[Check Status]
    
    Home --> EventDetail[Open Event Detail or My Events]
    
    EventDetail --> BackendCheck[GET /attendance]
    BackendCheck --> Result{Validation}
    
    Result -- Present --> PresentState[View 'Attendance recorded']
    Result -- Absent --> AbsentState[View 'No attendance record']
    
    AbsentState --> Eligible{Within Dispute Window?}
    
    Eligible -- Yes --> Dispute[Click 'Report an issue']
    Dispute --> Submit[Provide Reason & Evidence]
    Submit --> Pending[State: PENDING]
    
    Pending --> AdminReview[Admin Reviews]
    AdminReview --> Excused[State: EXCUSED]
    
    Eligible -- No --> Done[Cannot file dispute]
```

## Backend References
- **Routes**: `GET /users/me/attendance`, `POST /attendance/disputes`
- **Models**: `AttendanceRecord`, `AttendanceDispute`
