# Returning Student Journey

This journey maps the experience of an active student opening the application on a typical day.

```mermaid
flowchart TD
    Open[Open Application]
    Home[Land on Home Dashboard]
    
    Home --> Actions{Pending Actions?}
    
    Actions -- Yes --> Resolve[Accept Invites / Read Alerts]
    Actions -- No --> Schedule[Review 'Upcoming Commitments' card]
    
    Resolve --> Schedule
    
    Schedule --> Notifications[Check Notification Bell]
    Notifications --> Updates[See Waitlist Promotions / Announcements]
    
    Updates --> Campus[Navigate to Campus]
    Campus --> Leaderboard[Check Leaderboard Rank changes]
    
    Leaderboard --> Discover[Browse new Events]
```

## Backend References
- **Routes**: `GET /users/me/team-invitations`, `GET /users/me/registrations`, `GET /notifications`, `GET /leaderboard/students`
- **UX Goal**: The Home screen immediately captures attention if action is required, otherwise seamlessly transitions the student into community discovery.
