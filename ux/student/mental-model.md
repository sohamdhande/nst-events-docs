# Student Mental Model

The student experience is designed around natural student goals, intentionally obfuscating backend concepts like "Registration Entities" or "Attendance Records".

## Core Questions vs. Product Areas

| Student Question | Mental Concept | Product Area / Screen |
|------------------|----------------|-----------------------|
| "What do I need to do right now?" | Action | **Home** (Action Center) |
| "What's happening on campus?" | Discovery | **Campus** → Discover |
| "What have I committed to?" | Schedule | **My Events** (Upcoming tab) |
| "Am I going to get in?" | Waiting | **My Events** (Waitlist tab) |
| "Was I marked present?" | Proof | **My Events** (Past tab) → Attendance |
| "Where is my team?" | Group | **Event Detail** → Team Roster |
| "What have I achieved?" | Status | **Campus** → Leaderboard |
| "Which clubs am I part of?" | Identity | **Profile** → My Clubs |
| "What changed while I was gone?" | Updates | **Notifications** |

## Mental Model Diagram

```mermaid
flowchart TD
    Me((Me, the Student))

    Me --> Action[Action: "What must I do now?"]
    Me --> Schedule[Schedule: "What am I doing later?"]
    Me --> Community[Community: "What else is out there?"]

    Action --> CheckIn[Check into event]
    Action --> AcceptInvite[Accept team invite]
    
    Schedule --> ViewConfirmed[View confirmed events]
    Schedule --> CheckWaitlist[Check waitlist status]
    Schedule --> Cancel[Cancel if I can't go]

    Community --> BrowseEvents[Browse upcoming events]
    Community --> CheckRank[Check my leaderboard rank]
    Community --> ViewClubs[View active clubs]

    CheckIn -.-> Home
    AcceptInvite -.-> Home
    
    ViewConfirmed -.-> MyEvents
    CheckWaitlist -.-> MyEvents
    Cancel -.-> MyEvents

    BrowseEvents -.-> Campus
    CheckRank -.-> Campus
    ViewClubs -.-> Campus
```
