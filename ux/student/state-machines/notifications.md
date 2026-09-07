# Notification State Machine

This models the lifecycle of a student's notification inbox items.

```mermaid
stateDiagram-v2
    [*] --> UNREAD : Received from system
    
    UNREAD --> READ : Student views / clicks
    UNREAD --> READ : Student clicks "Mark all read"
    
    READ --> [*] : Auto-archived eventually (not exposed in UI)
```

## Backend References
- **Routes**: `PATCH /notifications/:id/read`, `PATCH /notifications/read-all`
- **Models**: `Notification`
- **Field**: `readAt` (DateTime or null)
