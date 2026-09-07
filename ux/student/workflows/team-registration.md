# Team Registration & Invitation Workflow

## Overview
Team events require a multi-step process: creating a team, inviting members, and achieving the required team constraints before participating.

## Flow: Creating & Inviting
```mermaid
sequenceDiagram
    participant Leader
    participant Invitee
    participant UI as Event / Team UI
    participant API as API

    Leader->>UI: Click "Create Team"
    UI->>API: POST /events/:id/teams (name: "Alpha")
    API-->>UI: Team Created (FORMING)
    
    Leader->>UI: Enter Invitee Email -> Click Invite
    UI->>API: POST /teams/:id/invitations
    API-->>UI: Success
    
    API->>Invitee: Notification Sent (Push + Inbox)
```

## Flow: Accepting Invite
```mermaid
sequenceDiagram
    participant Invitee
    participant UI as Home / Notifications
    participant API as API

    Invitee->>UI: View "Pending Invites" on Home
    UI->>API: GET /users/me/team-invitations
    API-->>UI: List of Invites
    
    Invitee->>UI: Click "Accept"
    UI->>API: POST /teams/:id/invitations/:id/accept
    
    alt Success
        API-->>UI: Status: ACCEPTED
        UI->>Invitee: Added to Team Roster
    else Full / Locked
        API-->>UI: 422 Error
        UI->>Invitee: "Team is full or event locked"
    end
```

## Backend References
- **Routes**: `POST /events/:id/teams`, `POST /teams/:id/invitations`, `POST /teams/:id/invitations/:invId/accept`
- **Constraints**: 
  - Team members cannot self-register; they are bound to the team's registration status (`FORMING`, `REGISTERED`, `WAITLISTED`).
  - Only the team leader can invite or remove members (`DELETE /teams/:id/members/:userId`).
