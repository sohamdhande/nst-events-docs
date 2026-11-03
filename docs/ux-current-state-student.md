# NST Events — Student App UX Flowchart (Current State)

This document maps the **existing, in-code UX flows** for the student-facing web application (`apps/dashboard/app/(student)`). It serves as a faithful snapshot of the currently implemented UI states, conditional branches, and missing/stubbed functionality prior to the redesign.

---

## 1. Auth & Entry

```mermaid
flowchart TD
    A[Login Page] --> B(Google OAuth Button)
    B --> C{Domain Validation}
    C -->|Invalid Domain| D[Error: RESTRICTED DOMAIN]
    C -->|Not in Directory| E[Error: STUDENT NOT ENROLLED]
    C -->|Auth Cancelled| F[Error: SIGN-IN CANCELLED]
    C -->|General Error| G[Error: AUTH FAILED]
    C -->|Success| H{Has return_to param?}
    H -->|Yes| I[Redirect to return_to]
    H -->|No| J[Redirect to /student/home]
```

- **UI States:**
  - **Idle:** Brand header, domain requirements text, Google login button.
  - **Error:** Neo-brutalist error enclosure with specific badge, message, and action buttons.
- **Conditional Branches:**
  - `INSTITUTIONAL_DOMAIN_NOT_ALLOWED`: Shows "Switch Account" action.
  - `STUDENT_ACCESS_NOT_AUTHORIZED`: Shows "Contact Coordinator" email trigger.
  - `ACCESS_DENIED` & `AUTHENTICATION_FAILED`: Shows "Switch Account".

---

## 2. Home Dashboard

```mermaid
flowchart TD
    A[Home Page] --> B{Data Loading?}
    B -->|Yes| C[Loading Skeletons]
    B -->|Error| D[Error: Could not load home page]
    B -->|Success| E[Home Content]
    E --> F[Greeting & Stats]
    E --> G{Active/Live Event?}
    G -->|Yes| H[Live Now Card]
    G -->|No| I{Upcoming Events?}
    I -->|Yes| J[Your Next Event Card]
    I -->|No| K[Empty Upcoming List]
    H --> L[Upcoming Registrations List]
    J --> L
```

- **UI States:**
  - **Loading:** Pulse skeletons for stats and cards.
  - **Error:** "Could not load your home page" + Retry button.
  - **Content:** Dynamic greeting based on time of day.
- **Conditional Branches:**
  - **Live Now Card:** Shows if a registered event's start time is within 15 minutes of the current time. Links to Event Detail.
  - **Your Next Event:** Shows the immediate next upcoming event (excluding the live one).
  - **Upcoming List:** Shows the rest of the future registered events.

---

## 3. Campus: Discover Feed

```mermaid
flowchart TD
    A[Discover Page] --> B[Search Input & Category Filters]
    B --> C{API Fetch}
    C -->|Loading| D[Pulse Skeletons]
    C -->|Error| E[Error: Couldn't load events]
    C -->|Empty| F[Empty: No events found]
    C -->|Success| G[Poster Wall Grid]
    G --> H[Event Card]
    H --> I[Navigate to Event Detail]
```

- **UI States:**
  - **Loading:** Grid of pulse skeletons.
  - **Error:** "Something went wrong" + Retry button.
  - **Empty:** "Try a different search" + Clear Filters button.
- **Conditional Branches:**
  - Search input (`q`) is debounced and updates the URL.
  - Category pill clicks update the `type` URL param.
  - API fetches `PUBLISHED` events matching the filters.

---

## 4. Campus: Clubs Directory

```mermaid
flowchart TD
    A[Clubs Page] --> B[Search Input]
    B --> C{List Fetch}
    C -->|Empty/Error| D[Empty/Error State]
    C -->|Success| E[Clubs Grid]
    E --> F[Navigate to Club Detail]
    
    F --> G{Detail Fetch}
    G -->|Error| H[Error: Club not found]
    G -->|Success| I[Club Detail Content]
    
    I --> J{Is Current User CLUB_ADMIN?}
    J -->|Yes| K[Show Club Operations stub]
    J -->|No| L[Hide Operations]
    
    I --> M[Roster List]
    I --> N[Upcoming Events List]
```

- **UI States (List & Detail):**
  - **Loading:** Skeletons for the grid or the club banner.
  - **Error/Not Found:** "Club not found" + "Back to Directory" button.
- **Conditional Branches:**
  - **Role check:** If user has `CLUB_ADMIN` role for this specific club, a "Club Operations" panel is rendered.
- **Stubbed/Unfinished:**
  - **[NOT YET BUILT]** "Club Operations" panel is currently a placeholder stating: *Administrative controls are currently being integrated. Event creation and roster management will appear here.*

---

## 5. Campus: Leaderboard

```mermaid
flowchart TD
    A[Leaderboard Page] --> B{Active Tab}
    B -->|Students| C[Fetch Global Students]
    B -->|Clubs| D[Fetch Global Clubs]
    
    C & D --> E{Status}
    E -->|Loading| F[Pulse Podiums & Rows]
    E -->|Empty| G[Empty: No rankings yet]
    E -->|Success| H[Top 3 Podiums]
    H --> I[Ledger Rows 4+]
```

- **UI States:**
  - **Loading:** Pulse animation for top 3 podiums and list rows.
  - **Empty:** "Check back later when activity begins".
- **Conditional Branches:**
  - **Tabs:** State toggles between `students` and `clubs` queries.

---

## 6. Event Detail & Registration

```mermaid
flowchart TD
    A[Event Detail Page] --> B{Fetch Event & Reg}
    B -->|Loading| C[Spinner]
    B -->|Error| D[Error: Cannot load]
    B -->|Success| E[Detail Content]
    
    E --> F{Event Lock State}
    F -->|Locked/Past| G[Read Only Mode]
    F -->|Unlocked| H{Registration Type}
    
    H -->|TEAM| I{Is Registered?}
    I -->|Yes| J[Manage Team Button -> Team Hub]
    I -->|No| K[Join / Create Team -> Gateway Modal]
    
    H -->|SOLO| L{Is Capacity Full?}
    L -->|No| M[Register Button -> Confirm Modal]
    L -->|Yes| N[Join Waitlist Button -> Confirm Modal]
    
    E --> O{Is Dispute Window Open?}
    O -->|Yes| P[Report Issue Button -> Dispute Modal]
```

- **UI States:**
  - **Loading:** Spinner with "Loading Event...".
  - **Error:** "This event could not be loaded."
  - **Read Only:** "Event Locked — Read Only" banner.
- **Conditional Branches:**
  - **Lock State:** True if `lock_state !== 'UNLOCKED'`, or 24 hours past `endTime`, or `endTime` has passed. Disables registration actions.
  - **Dispute Window:** Open for 24 hours after the event ends. Allows filing attendance disputes.
  - **Capacity:** If registrations >= maxCapacity, switches SOLO flow to Waitlist.
  - **Admin View:** If user is a `CLUB_ADMIN` for the organizing club, shows a hidden link to the organizer manage page.

---

## 7. Team Hub

```mermaid
flowchart TD
    A[Team Hub Page] --> B{Fetch Team Data}
    B -->|Loading| C[Spinner]
    B -->|Success| D{Is user in a team?}
    D -->|No| E[Empty: Not in team -> Back to event]
    D -->|Yes| F[Team Content]
    
    F --> G{Is Leader?}
    G -->|Yes| H[Transfer Leadership Modal]
    G -->|Yes| I{Only Member?}
    I -->|Yes| J[Cancel Team Reg Modal]
    
    G -->|No| K[Leave Team Modal]
    
    F --> L{Is Below Minimum Size?}
    L -->|Yes| M[Warning Banner]
```

- **UI States:**
  - **Loading:** Spinner.
  - **Warning:** `isBelowMinimum` triggers a warning state indicating the team is invalid.
- **Conditional Branches:**
  - **Leadership restrictions:** Leaders cannot leave the team unless they are the *only* member (which cancels the team entirely) or they transfer leadership first.

---

## 8. Organizer/Manage Flows

```mermaid
flowchart TD
    A[Club Admin Portal] --> B[Roster Management]
    A --> C[Event Management List]
    
    C --> D[Create Event Form]
    D --> E[Draft State]
    E --> F[Submit For Approval]
    F --> G[Published]
    
    C --> H[Manage Specific Event]
    H --> I[Registrations Tab]
    H --> J[Teams Tab]
    H --> K[Attendance Sessions Tab]
    H --> L[Edit Event Details Tab]
```

- **UI States & Conditional Branches:**
  - Depends on `role === 'CLUB_ADMIN'` for the active club context.
  - **Event Lifecycle:** Draft -> Submitted -> Published/Rejected.
  - **Teams Management:** Admins can manually promote waitlisted teams, cancel teams, or forcibly transfer leadership.

---

## 9. Notifications Flow

```mermaid
flowchart TD
    A[Notifications Page] --> B{Fetch List}
    B -->|Loading| C[Pulse List]
    B -->|Error| D[Error State]
    B -->|Empty| E[Empty State]
    B -->|Success| F[Chronological Feed]
    
    F --> G[Notification Item]
    G -->|Click| H{Is Unread?}
    H -->|Yes| I[Mark Read Mutation]
    H -->|No| J[Navigate to targetHref]
```

- **UI States:**
  - **Loading:** Pulse animation list.
  - **Empty:** "No notifications."
- **Conditional Branches:**
  - `readAt` timestamp presence determines visual unread indicator.
  - Clicking an item triggers navigation and background read mutation simultaneously.

---

## 10. Profile

```mermaid
flowchart TD
    A[Profile Page] --> B{Fetch User & Prefs}
    B -->|Loading| C[Skeleton]
    B -->|Error| D[Error: Failed to load]
    B -->|Success| E[Identity Card]
    
    E --> F[My Clubs List]
    E --> G[Notification Toggles]
    E --> H[Sign Out Button]
    
    G --> I[Mutation: Update Preferences]
    H --> J[Sign Out Confirm Modal]
```

- **UI States:**
  - **Loading:** Skeleton structure.
  - **Error:** "Failed to load profile" + Retry.
- **Conditional Branches:**
  - **Academic Profile:** Fallbacks for displaying Batch, Program, or generic Student.
  - **Clubs Empty State:** If `club_memberships` is empty, shows CTA to Discover Clubs.
  - **Toggles:** Immediate optimistic UI mutation for push, reminders, announcements, and alerts.
