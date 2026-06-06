# 21 End-to-End Request Lifecycle

This sequence diagram exemplifies a complete request lifecycle, specifically detailing a student registering for an event, traversing from the client to the database.

```mermaid
sequenceDiagram
    participant Student
    participant Mobile as Mobile App
    participant API as Supabase API
    participant RPC as Edge Functions / RPC
    participant RLS as Row Level Security
    participant DB as Database
    
    Student->>Mobile: Clicks "Register"
    Mobile->>API: POST /rest/v1/event_registrations
    API->>RPC: Call register_for_event()
    RPC->>RLS: Validate Session & Permissions
    alt Unauthorized
        RLS-->>RPC: Reject Access
        RPC-->>API: 403 Forbidden
        API-->>Mobile: Error Message
    else Authorized
        RLS->>DB: Execute Transaction
        DB->>DB: Lock Event Row (SELECT FOR UPDATE)
        DB->>DB: Check Capacity
        alt Full
            DB->>DB: Create Waitlist Record
            DB-->>RLS: Waitlisted Status
        else Available
            DB->>DB: Create Registration Record
            DB-->>RLS: Registered Status
        end
        RLS-->>RPC: Success Result
        RPC-->>API: 200 OK / Response Payload
        API-->>Mobile: Update UI Status
        Mobile-->>Student: Show Success/Waitlist
    end
```
