# 08 Attendance Sequence Flow

This sequence diagram provides a step-by-step breakdown of the attendance marking process from the student's mobile app to the backend database.

```mermaid
sequenceDiagram
    participant Student
    participant App as Mobile App
    participant Backend as Supabase Backend
    participant DB as PostgreSQL Database
    
    Student->>App: Scans Dynamic QR Code
    App->>App: Captures Geolocation
    App->>Backend: Submit Attendance (QR payload, Location)
    Backend->>Backend: Validate HMAC & TOTP
    Backend->>Backend: Validate Geofence
    alt Valid Scan
        Backend->>DB: Create Attendance Record
        DB-->>Backend: Record Created
        Backend-->>App: Success Response
        App-->>Student: Show Success Screen
    else Invalid Scan (e.g., Location mismatch)
        Backend-->>App: Error Response
        App-->>Student: Show Error / Prompt Dispute
    end
```
