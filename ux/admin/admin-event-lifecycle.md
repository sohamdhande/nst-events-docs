# Event Lifecycle & Operations

This document maps the complete operational workflows, state machine, lock mechanism, and operational sub-page behaviors for events within the NST Events platform, as implemented in `apps/dashboard`.

---

## Event State Machine

An event's `state` strictly governs its visibility, mutability, and which lifecycle actions are available in the UI.

```mermaid
stateDiagram-v2
    [*] --> DRAFT : Create Event

    DRAFT --> PENDING_APPROVAL : Submit for Approval
    DRAFT --> PUBLISHED : Publish Event (Admin Bypass)
    DRAFT --> [*] : Delete Event

    PENDING_APPROVAL --> PUBLISHED : Approve
    PENDING_APPROVAL --> DRAFT : Reject (with reason)

    PUBLISHED --> ARCHIVED : Auto-Archive (end date passed)

    ARCHIVED --> [*]
```

### State Definitions

| State                | Visibility                      | Editability                            | Description                                                                           |
| -------------------- | ------------------------------- | -------------------------------------- | ------------------------------------------------------------------------------------- |
| `DRAFT`            | Hidden from students            | Fully editable                         | Event is being authored. Only visible to organizers.                                  |
| `PENDING_APPROVAL` | Hidden from students            | Locked                                 | Awaiting Faculty Mentor or Admin review.                                              |
| `PUBLISHED`        | Visible per visibility settings | Read-only (details), operations active | Event is live. Registration, teams, and attendance are operational (subject to lock). |
| `ARCHIVED`         | Visible (historical)            | Completely read-only                   | Past event. All operations frozen.                                                    |

---

## Event Lock Mechanism

Independent of the event `state`, a secondary `lockState` is dynamically resolved via [`resolveEventLockState()`](file:///Users/sohamdhande/Docs_Local/nst-events/apps/dashboard/lib/event-utils.ts) to control operational mutations during the `PUBLISHED` state.

```mermaid
stateDiagram-v2
    [*] --> UNLOCKED : Event Published
    UNLOCKED --> MANUALLY_LOCKED : Lock Event action
    MANUALLY_LOCKED --> UNLOCKED : Unlock Event action
    UNLOCKED --> PERMANENTLY_LOCKED : End date passes
    MANUALLY_LOCKED --> PERMANENTLY_LOCKED : End date passes
    PERMANENTLY_LOCKED --> [*]
```

### Lock State Effects

| Lock State             | Effect on Operations                                                                                                                              | Reversible?         |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------- |
| `UNLOCKED`           | All operations active. Registration open (capacity permitting). Teams can form/modify.                                                            | N/A                 |
| `MANUALLY_LOCKED`    | Registration frozen. Team operations frozen. Attendance QR generation disabled. All action buttons disabled. "LOCKED — READ-ONLY" tag displayed. | Yes (Unlock button) |
| `PERMANENTLY_LOCKED` | Identical to manual lock but cannot be reversed. "PERMANENTLY LOCKED — READ-ONLY" displayed.                                                     | No                  |

### Lock-Aware UI Behavior

All operational sub-pages (`/registrations`, `/teams`, `/attendance`) check `resolveEventLockState()`:

- Mutation buttons (Cancel Team, Promote Waitlist, Generate QR, End Session) are hidden or disabled.
- Red "LOCKED — READ-ONLY" tag is displayed in page headers.
- QR auto-refresh loop is terminated and fullscreen mode is exited.

---

## Event Creation Workflow

```mermaid
sequenceDiagram
    participant U as User
    participant UI as Create Event Page
    participant API as Backend API

    U->>UI: Fill out form sections
    U->>UI: Click "Save Draft"
    UI->>API: POST /events (create)
    API-->>UI: { id: eventId }
    UI->>UI: Navigate to /events/{id}

    Note over U,UI: Alternative: "Submit for Approval"
    U->>UI: Click "Submit for Approval"
    UI->>API: POST /events (create)
    API-->>UI: { id: eventId }
    UI->>API: POST /events/{id}/submit
    API-->>UI: 200 OK
    UI->>UI: Navigate to /events/{id}

    Note over UI: If create succeeds but submit fails
    UI->>UI: Show warning alert with "View Draft" and "Retry Submission" buttons
```

---

## Event Approval Workflow

Two paths exist for transitioning a DRAFT event to PUBLISHED:

### Path A: Standard Approval (Club Admin → Faculty Mentor)

```mermaid
sequenceDiagram
    participant CA as Club Admin / Core Member
    participant EP as Event Detail Page
    participant FM as Faculty Mentor / Admin
    participant AP as Approvals Page
    participant API as Backend

    CA->>EP: Click "Submit for Approval"
    EP->>EP: Show Popconfirm
    CA->>EP: Confirm
    EP->>API: POST /events/{id}/submit
    API-->>EP: Event → PENDING_APPROVAL

    Note over FM: Sees event in Approvals page and Dashboard

    FM->>AP: Click "Review" → navigates to Event Detail
    FM->>EP: Reviews event metadata (Review Summary Card)
  
    alt Approve
        FM->>EP: Click "Approve Event"
        EP->>EP: Show Popconfirm
        FM->>EP: Confirm
        EP->>API: POST /events/{id}/approve
        API-->>EP: Event → PUBLISHED
    else Reject
        FM->>EP: Click "Reject Event"
        EP->>EP: Open Reject Modal
        FM->>EP: Enter reason, click "Reject Event"
        EP->>API: POST /events/{id}/reject
        API-->>EP: Event → DRAFT
    end
```

### Path B: Admin Bypass (Global Admin Direct Publish)

```mermaid
sequenceDiagram
    participant GA as Global Admin
    participant EP as Event Detail Page
    participant API as Backend

    Note over GA: Sees "Publish Event" button on DRAFT events

    GA->>EP: Click "Publish Event"
    EP->>EP: Open Publish Modal
    GA->>EP: Confirm
    EP->>API: POST /events/{id}/submit
    API-->>EP: Event → PENDING_APPROVAL
    EP->>API: POST /events/{id}/approve
    API-->>EP: Event → PUBLISHED

    Note over EP: If approve fails after submit succeeds
    EP->>EP: Show "Publish Incomplete" panel
    EP->>EP: "Retry Publish" button available
```

---

## Event Edit & Delete Workflow

```mermaid
sequenceDiagram
    participant U as User
    participant EP as Event Detail Page
    participant ED as Edit Event Page
    participant API as Backend

    U->>EP: Click "Edit Event" (DRAFT only)
    EP->>ED: Navigate to /events/{id}/edit
    ED->>API: GET /events/{id} (load current data)
    API-->>ED: Event data (pre-populate form)
    U->>ED: Modify fields (isDirty = true)
  
    alt Save
        U->>ED: Click "Save Changes"
        ED->>API: PATCH /events/{id}
        API-->>ED: Success
        ED->>ED: Navigate to /events/{id}
    else Cancel (dirty)
        U->>ED: Click "Cancel"
        ED->>ED: Show confirm "Discard unsaved changes?"
        U->>ED: Confirm → navigate back
    else Delete
        U->>ED: Click "Delete Event" (Danger Zone)
        ED->>ED: Show confirm "Delete Event?"
        U->>ED: Confirm
        ED->>API: DELETE /events/{id}
        API-->>ED: Success
        ED->>ED: Navigate to /events
    end
```

---

## Event Lock/Unlock Workflow

```mermaid
sequenceDiagram
    participant U as Authorized User
    participant EP as Event Detail Page
    participant API as Backend
    participant OP as Operations Pages

    Note over EP: Event is PUBLISHED + UNLOCKED

    U->>EP: Click "Lock Event" (danger button)
    EP->>API: POST /events/{id}/lock
    API-->>EP: Event → MANUALLY_LOCKED
    EP->>EP: Re-render lifecycle panel with "Unlock Event" button
    Note over OP: All operations pages now show LOCKED tag, mutations disabled

    U->>EP: Click "Unlock Event"
    EP->>API: POST /events/{id}/unlock
    API-->>EP: Event → UNLOCKED
    EP->>EP: Re-render with "Lock Event" button
    Note over OP: Operations resume
```

---

## Attendance Session Lifecycle

```mermaid
sequenceDiagram
    participant O as Organizer
    participant AP as Attendance Page
    participant API as Backend
    participant S as Student (Mobile)

    O->>AP: Click "+ New Session"
    AP->>AP: Open Create Session Modal (auto-capture geolocation)
    O->>AP: Fill title, times, geofence radius
    O->>AP: Submit
    AP->>API: POST /events/{id}/attendance/sessions
    API-->>AP: { id: sessionId }

    Note over AP: Session selector auto-selects new session

    O->>AP: Click "Generate QR"
    AP->>API: POST /sessions/{id}/qr
    API-->>AP: { qr_payload, expires_at }
    AP->>AP: Display QR + countdown timer

    loop Auto-Refresh (every ~30s)
        AP->>API: POST /sessions/{id}/qr
        API-->>AP: New { qr_payload, expires_at }
        AP->>AP: Replace QR silently
    end

    S->>S: Scan QR with mobile app
    S->>API: POST /attendance/mark (qr_payload)
    API-->>S: Attendance recorded

    O->>AP: Click "End Session"
    AP->>AP: Confirm modal
    O->>AP: Confirm
    AP->>API: PATCH /sessions/{id} (close_at = now)
    API-->>AP: Session → ENDED
    AP->>AP: QR cleared, "SESSION ENDED" card displayed
```

### QR Refresh Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Idle : Page loads
    Idle --> Generating : Click "Generate QR"
    Generating --> Active : QR received
    Active --> Refreshing : ~2s before expiry
    Refreshing --> Active : New QR received
    Refreshing --> RetryBackoff : Error (non-429)
    Refreshing --> RateLimit : 429 Too Many Requests
    RetryBackoff --> Refreshing : Retry after 5s (max 3)
    RateLimit --> Refreshing : Retry after 10s (max 5)
    RetryBackoff --> Expired : Max retries exceeded
    RateLimit --> Expired : Max retries exceeded
    Expired --> Generating : Click "Regenerate"
    Active --> Terminated : Session ended / Event locked
    Terminated --> [*]
```

---

## Attendance Dispute Resolution Workflow

```mermaid
sequenceDiagram
    participant S as Student
    participant API as Backend
    participant O as Organizer (Disputes Tab)

    S->>API: Submit dispute (reason text)
    API-->>S: Dispute → PENDING

    Note over O: Sees dispute in Attendance page "Disputes" tab

    O->>O: Click "Review" on PENDING dispute
    O->>O: Read student's reason
    O->>O: Select decision (APPROVED / REJECTED)
    O->>O: Optionally add review notes
    O->>API: PATCH /disputes/{id} (resolution, review_notes)
    API-->>O: Dispute → APPROVED or REJECTED
```

---

## Team Management Workflow

```mermaid
sequenceDiagram
    participant A as Admin / Club Admin
    participant TP as Teams Page
    participant API as Backend

    Note over TP: Table shows all teams with expandable member rosters

    alt Cancel Team
        A->>TP: Click "⋯" → "Cancel Team"
        TP->>TP: Confirm modal
        A->>TP: Confirm
        TP->>API: POST /teams/{id}/cancel
        API-->>TP: Team → CANCELLED (capacity freed)
    end

    alt Promote Waitlisted Team
        A->>TP: Click "⋯" → "Promote Waitlist" (WAITLISTED teams only)
        TP->>TP: Confirm modal
        A->>TP: Confirm
        TP->>API: POST /teams/{id}/promote
        API-->>TP: Team → REGISTERED
    end

    alt Remove Member
        A->>TP: Expand team row → click "⋯" on member → "Remove Member"
        TP->>TP: Confirm modal with member name
        A->>TP: Confirm
        TP->>API: POST /teams/{id}/members/{userId}/remove
        API-->>TP: Member removed
    end

    alt Transfer Leadership
        A->>TP: Expand team row → click "⋯" on non-leader → "Transfer Leadership"
        TP->>TP: Confirm modal with member name
        A->>TP: Confirm
        TP->>API: POST /teams/{id}/transfer-leadership
        API-->>TP: Leadership transferred
    end
```

---

## Health Alerts

The dashboard and event detail page surface contextual alerts based on real-time event health:

| Alert                | Condition                                           | Location                    | Visual                                              |
| -------------------- | --------------------------------------------------- | --------------------------- | --------------------------------------------------- |
| `NEAR CAPACITY`    | `registrationCount / maxCapacity >= 0.9`          | Event Detail sidebar        | Warning tag                                         |
| `NEEDS ATTENTION`  | `below_minimum_team_count > 0` and event UNLOCKED | Event Detail main + sidebar | Warning tag + inline alert with "Manage Teams" link |
| `HEALTHY`          | Published, unlocked, no attention flags             | Event Detail sidebar        | Success tag                                         |
| `PENDING APPROVAL` | `state === 'PENDING_APPROVAL'`                    | Event Detail sidebar        | Warning tag                                         |

---

## Live Updates

The event detail page subscribes to real-time updates via the [`useEventLiveUpdates`](file:///Users/sohamdhande/Docs_Local/nst-events/apps/dashboard/hooks/useEventLiveUpdates.ts) hook, which keeps the displayed data in sync with server-side changes (e.g., another admin approving/locking the event, or registrations changing the count).
