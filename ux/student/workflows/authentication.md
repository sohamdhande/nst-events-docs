# Authentication & Onboarding Workflow

## Overview
NST Events uses a strict institutional-email gated authentication system. The backend validates users against a pre-authorized directory before granting access.

## Flow
```mermaid
sequenceDiagram
    participant S as Student
    participant UI as Login Page
    participant API as Auth API (Google)
    participant DB as AuthorizedStudents

    S->>UI: Click "Login with Institution"
    UI->>API: GET /auth/google
    API->>API: Google OAuth Consent
    API->>DB: Check email in AuthorizedStudents
    
    alt Email Not Authorized
        API-->>UI: Redirect /unauthorized
        UI->>S: Show "Domain Not Authorized" error
    else Email Authorized
        API->>DB: Create/Update User (Role: STUDENT)
        API-->>UI: Set HttpOnly Cookie & redirect /dashboard
        UI->>S: Land on Home Action Center
    end
```

## Backend References
- **Routes**: `GET /auth/google`, `GET /users/me`
- **Models**: `User`, `AuthorizedStudent`

## Edge Cases
- **Revoked Status**: If `DirectoryStatus === REVOKED`, the login is immediately rejected, and the UI displays an "Access Revoked" message.
- **First Login**: Handled seamlessly by the backend; no explicit frontend onboarding multi-step wizard exists in the current backend design.
