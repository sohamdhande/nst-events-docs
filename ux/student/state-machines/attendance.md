# Attendance State Machine

This models the student-facing attendance state lifecycle on the Student Web App.

*The Student Web App displays attendance results only. Physical check-in (QR scanning) occurs externally.*

## Student-Facing Attendance States

```mermaid
stateDiagram-v2
    [*] --> NoRecord

    NoRecord --> PRESENT : External check-in recorded
    NoRecord --> ABSENT : Session closed (system)
    NoRecord --> ABSENT : Admin manual mark

    ABSENT --> EXCUSED : Dispute approved
    ABSENT --> DisputePending : Student files dispute

    DisputePending --> EXCUSED : Admin approves
    DisputePending --> ABSENT : Admin rejects

    PRESENT --> [*]
    EXCUSED --> [*]
```

## Backend Record States
*(Internal — the backend determines transitions. The Web App reads the result.)*

```mermaid
stateDiagram-v2
    [*] --> Unmarked
    Unmarked --> PRESENT : External QR Scan / Manual Mark
    Unmarked --> ABSENT : Session Closed (System job)
    ABSENT --> EXCUSED : Dispute Approved
```

## Backend References
- **Read Route**: `GET /users/me/attendance`
- **Dispute Route**: `POST /attendance/disputes`
- **Models**: `AttendanceRecord`, `AttendanceSession`, `AttendanceDispute`
- **Enums**: `AttendanceStatus` (`PRESENT`, `ABSENT`, `EXCUSED`), `AttendanceMethod` (`QR`, `MANUAL`, `SYSTEM`)
