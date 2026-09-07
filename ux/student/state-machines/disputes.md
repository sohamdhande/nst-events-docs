# Attendance Disputes State Machine

This models the lifecycle of a student filing an exception request for missed attendance.

```mermaid
stateDiagram-v2
    [*] --> Draft : Student identifies issue
    Draft --> PENDING : Submit reason & evidence
    
    PENDING --> APPROVED : Admin Review
    PENDING --> REJECTED : Admin Review
    
    APPROVED --> [*] : Attendance Record updated to EXCUSED or PRESENT
    REJECTED --> [*]
```

## Backend References
- **Routes**: `POST /attendance/disputes`
- **Models**: `AttendanceDispute`
- **Enums**: `DisputeStatus` (`PENDING`, `APPROVED`, `REJECTED`)
- **Note**: The student cannot cancel or edit a dispute once submitted. There is only one terminal transition decided by the admin.
