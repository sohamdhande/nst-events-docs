# Event Participation Journey

This journey maps the standard end-to-end flow for participating in a solo event, including waitlisting and attendance.

```mermaid
flowchart TD
    Discover[Find Event on Campus]
    Evaluate[Read Event Detail & Requirements]
    Register[Click Register]
    
    Register --> Full{Event Full?}
    
    Full -- Yes --> Waitlisted[State: WAITLISTED]
    Full -- No --> Registered[State: REGISTERED]
    
    Waitlisted --> Worker[Backend Worker Promotes]
    Worker --> Notified[Receive Promotion Notification]
    Notified --> Registered
    
    Registered --> MyEvents[Track in My Events]
    
    MyEvents --> DayOf[Event Day Arrives]
    DayOf --> CheckIn[Attend Event / External Check-in]
    
    CheckIn --> Present[Web App reflects PRESENT]
    
    Present --> Leaderboard[Points Awarded]
    Leaderboard --> History[View in Attendance History]
```

## Backend References
- **Routes**: `POST /events/:id/register`, `GET /users/me/attendance`
- **Models**: `EventRegistration`, `AttendanceRecord`, `Notification`
- **Workers**: `NotificationJob` for asynchronous waitlist promotion.
