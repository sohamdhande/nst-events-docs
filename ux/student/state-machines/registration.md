# Registration State Machine

This diagram models the lifecycle of a student's registration for an individual event.

```mermaid
stateDiagram-v2
    [*] --> Unregistered

    Unregistered --> REGISTERED : Register (capacity available)
    Unregistered --> WAITLISTED : Register (event full)
    Unregistered --> Unregistered : Register (event locked/ended) -> API Error

    WAITLISTED --> REGISTERED : Automatic Waitlist Promotion
    
    REGISTERED --> CANCELLED : Cancel Registration
    WAITLISTED --> CANCELLED : Cancel Registration

    CANCELLED --> REGISTERED : Re-register (capacity available)
    CANCELLED --> WAITLISTED : Re-register (event full)

    REGISTERED --> [*] : Event ends
    CANCELLED --> [*] : Event ends
    WAITLISTED --> [*] : Event ends
```

## Backend References
- **Routes**: `POST /events/:id/register`, `DELETE /events/:id/register`
- **Model**: `EventRegistration`
- **Enum**: `RegistrationStatus` (`REGISTERED`, `WAITLISTED`, `CANCELLED`)
- **Note**: Waitlist promotion is handled entirely asynchronously by `NotificationJob` workers executing queue tasks. No student input is required.
