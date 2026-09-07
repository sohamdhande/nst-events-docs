# Teams State Machine

This models the complex state transitions for Team creation, invitation, and event registration.

## Team Lifecycle

```mermaid
stateDiagram-v2
    [*] --> FORMING : Create Team

    FORMING --> WAITLISTED : Register Team (event full)
    FORMING --> REGISTERED : Register Team (capacity available)
    
    WAITLISTED --> REGISTERED : Automatic Promotion
    
    REGISTERED --> CANCELLED : Admin/Leader cancels
    WAITLISTED --> CANCELLED : Admin/Leader cancels
    FORMING --> CANCELLED : Admin/Leader cancels

    CANCELLED --> [*]
    REGISTERED --> [*] : Event Ends
```

## Team Invitation Lifecycle

```mermaid
stateDiagram-v2
    [*] --> PENDING : Invite sent

    PENDING --> ACCEPTED : Student accepts
    PENDING --> DECLINED : Student declines
    PENDING --> EXPIRED : Time elapses
    PENDING --> CANCELLED : Leader revokes

    ACCEPTED --> [*]
    DECLINED --> [*]
    EXPIRED --> [*]
    CANCELLED --> [*]
```

## Backend References
- **Routes**: `POST /events/:id/teams`, `POST /teams/:id/invitations`, `POST /teams/:id/invitations/:invitationId/accept`
- **Models**: `Team`, `TeamInvitation`
- **Enums**: `TeamStatus` (`FORMING`, `REGISTERED`, `WAITLISTED`, `CANCELLED`), `InvitationStatus` (`PENDING`, `ACCEPTED`, `DECLINED`, `CANCELLED`, `EXPIRED`)
