# New Student Journey

This journey maps the end-to-end experience of a student interacting with the NST Events web application for the very first time.

```mermaid
flowchart TD
    Login[Login via Institutional Google Account]
    Auth[Domain Validation & Auth Bootstrap]
    Home[Land on Home Dashboard]
    EmptyHome[Observe Empty States on Home]
    Discover[Navigate to Campus -> Discover]
    Browse[Browse Upcoming Events]
    ViewEvent[View Event Details]
    Register[Register for Event]
    MyEvents[Navigate to My Events]
    Verify[See Event under 'Upcoming' Tab]

    Login --> Auth
    Auth --> Home
    Home --> EmptyHome
    EmptyHome --> Discover
    Discover --> Browse
    Browse --> ViewEvent
    ViewEvent --> Register
    Register --> MyEvents
    MyEvents --> Verify
```

## Backend References
- **Routes**: `GET /auth/google`, `GET /users/me`, `GET /events`, `POST /events/:id/register`
- **Logic**: The `AuthorizedStudent` model validates the domain. If valid, the `User` is created with a `STUDENT` global role and a fresh session.
