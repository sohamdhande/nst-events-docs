# Profile & Preferences Workflow

## Overview
The profile is a minimal, primarily read-only surface for identity and settings.

## Flow
```mermaid
sequenceDiagram
    participant S as Student
    participant UI as Profile Page
    participant API as Users API

    S->>UI: Navigate to Profile
    UI->>API: GET /users/me
    UI->>API: GET /notifications/preferences
    
    API-->>UI: Identity, Batch, Memberships, Prefs
    
    UI->>S: Display User Info (Read-only)
    UI->>S: Display Notification Toggles
    
    S->>UI: Toggle "Event Reminders" OFF
    UI->>API: PATCH /notifications/preferences
    API-->>UI: Success
```

## Backend References
- **Routes**: `GET /users/me`, `GET /notifications/preferences`, `PATCH /notifications/preferences`
- **Models**: `User`, `NotificationPreference`
- **Constraints**: Profile editing (Name, Batch) is not supported via student endpoints. These are synced from the institutional directory or modified by Admins.
