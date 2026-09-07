# Team Event Journey

This journey maps the complex flow required to participate in an event with `registrationType === TEAM`.

```mermaid
flowchart TD
    Event[View Team Event Detail]
    Create[Click Create Team]
    Forming[Team State: FORMING]
    
    Create --> Forming
    
    Forming --> Invite[Send Invites to Emails]
    Invite --> Pending[Invites PENDING]
    
    Pending --> Accept[Friends Accept Invites]
    Accept --> Roster[Members Added to Roster]
    
    Roster --> MinSize{Reached Min Size?}
    
    MinSize -- No --> Forming
    MinSize -- Yes --> RegisterTeam[Leader Clicks Register Team]
    
    RegisterTeam --> Cap{Capacity Open?}
    
    Cap -- Yes --> Registered[Team REGISTERED]
    Cap -- No --> Waitlisted[Team WAITLISTED]
    
    Registered --> Attend[Team attends event]
```

## Backend References
- **Routes**: `POST /events/:id/teams`, `POST /teams/:id/invitations`, `POST /teams/:id/invitations/:invId/accept`
- **Models**: `Team`, `TeamInvitation`, `EventRegistration`
- **Constraints**: Minimum size validation occurs on the API during the final `POST /teams/:id/register` (or equivalent team promotion) action.
