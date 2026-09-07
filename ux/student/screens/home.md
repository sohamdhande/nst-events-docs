# Student UX — Home Screen

## 1. Screen Overview

The Home screen is the student's personal operating surface within NST Events.

Its purpose is not to expose every available piece of campus information. Its purpose is to immediately answer:

> "What matters to me right now?"

Home prioritizes the student's current actions and commitments before broader campus discovery.

The screen follows this strict hierarchy:

```text
GREETING
↓
PRIORITY (NOW)
↓
YOUR NEXT
↓
YOUR PROGRESS
↓
EXPLORE CAMPUS
```

The Home experience is intentionally contextual. Sections appear, disappear, or change prominence depending on the student's current state.

---

## 2. Screen Purpose

Home should allow a student to:

* Understand whether an event requires immediate action (attendance).
* See their single next upcoming commitment.
* See their overall progress (points and events attended).
* Reach deeper workflows without making Home itself a complex management interface.

Home is not intended to replace:

* Event discovery
* My Events
* Notifications
* Attendance history
* Leaderboard
* Club browsing
* Profile/settings

Those experiences have dedicated destinations.

---

## 3. Global Placement

Home is the default authenticated landing screen.

The application shell contains:

```text
┌──────────────────────────────────────────────────────────┐
│ Global Header                                            │
│                                                          │
├──────────────────┬───────────────────────────────────────┤
│ Primary Nav      │ Home                                  │
│                  │                                       │
│ Home             │                                       │
│ Campus           │                                       │
│ My Events        │                                       │
│ Profile          │                                       │
│                  │                                       │
└──────────────────┴───────────────────────────────────────┘
```

### Primary navigation

```text
Home
Campus
My Events
Profile
```

### Global header

The header provides:

```text
NST Events
Notifications
Student profile / account
```

Notifications remain globally accessible rather than occupying permanent Home space.

---

# 4. Home Information Hierarchy

The screen is structured into the following levels:

```text
1. GREETING
   Lightweight personalized greeting.

2. PRIORITY (NOW)
   Time-sensitive or immediately actionable information (Live attendance).

3. YOUR NEXT
   The single next upcoming event the student has committed to.

4. YOUR PROGRESS
   Quick metrics (Total Points, Events Attended).

5. EXPLORE CAMPUS
   A clear, lightweight CTA entry point to the Discover route.
```

The order is intentional. Home should prioritize the student's personal context over campus-wide information.

---

# 5. Header & Greeting

The main content begins with a lightweight personalized greeting.

Example:

```text
Good afternoon, Soham.
```

The greeting should remain visually subordinate to the actual actionable content. Do not turn the greeting into a large hero area or administrative dashboard.

---

# 6. PRIORITY (NOW) Section

## Purpose

`PRIORITY` is the highest-priority contextual area of Home. It represents an event that is currently live or starting very soon (within 15 minutes).

If there is nothing urgent or actionable, `PRIORITY` disappears.

## Active Attendance State

When an event the student is registered for is active or starting soon, it receives the highest priority.

Example:

```text
┌────────────────────────────────────────────────────────────┐
│ ● HAPPENING NOW                                            │
│                                                            │
│ AI/ML Workshop                                             │
│ Main Auditorium · Attendance closes in 18 min             │
│                                                            │
│                                      [ VIEW TICKET ]       │
└────────────────────────────────────────────────────────────┘
```

The card communicates:
- Live state indicator
- Event Title
- Location (if useful)
- Time remaining
- Primary action

The primary action takes the student directly into the read-only ticket screen (`/student/events/:id/ticket`). The web application does NOT implement QR scanning, camera, or browser geolocation.

---

# 7. YOUR NEXT Section

## Purpose

`YOUR NEXT` is the persistent commitment layer.

It answers:
> "What am I participating in next?"

It contains a single compact preview of the student's next upcoming registered/waitlisted event.

Example:

```text
Your next                            View all →

┌──────────────────────────┐
│ TOMORROW · 10:00 AM     │
│                          │
│ Hackathon                │
│ Coding Club              │
│                          │
│ ● REGISTERED             │
└──────────────────────────┘
```

The section header contains "View all →" which navigates to `My Events`.
Clicking the event card navigates to the canonical Event Detail page.

Do NOT create a massive grid of events. This section is strictly for the *next* event.

---

# 8. NO UPCOMING EVENTS State

When the student has no upcoming registered/waitlisted events, do not display a blank card. 
The page gracefully falls back to:

```text
Nothing scheduled yet.
```

---

# 9. YOUR PROGRESS Section

Progress is secondary to immediate action. It displays high-level verified metrics.

Example:

```text
YOUR PROGRESS
┌────────────┐ ┌───────────────┐
│ POINTS     │ │ EVENTS ATTENDED│
│ 150        │ │ 12            │
└────────────┘ └───────────────┘
```

These metrics are read directly from `GET /v1/dashboard/summary`. 
Do not invent unsupported statistics or add complex charts here.

---

# 10. EXPLORE CAMPUS Section

`EXPLORE CAMPUS` provides a lightweight entry point into other campus opportunities.

It is NOT a replacement for Campus → Discover. Do not render a secondary event feed here.

Example:

```text
EXPLORE CAMPUS
Discover events, clubs and campus activity.
[ Explore Campus → ]
```

Clicking the action navigates the student directly to `Campus / Discover`.

---

# 11. Overall Home Flow & State Model

The complete Home decision model:

## STATE A — PRIORITY + UPCOMING
```text
(Left Column)             (Right Column)
Greeting                  Progress
↓                         ↓
Priority (Live Event)     Explore Campus
↓
Your Next
```

## STATE B — NO PRIORITY + UPCOMING
```text
(Left Column)             (Right Column)
Greeting                  Progress
↓                         ↓
Your Next                 Explore Campus
```

## STATE C — NO PRIORITY + NO UPCOMING
```text
(Left Column)             (Right Column)
Greeting                  Progress
↓                         ↓
Nothing scheduled         Explore Campus
```

---

# 12. Visual Hierarchy & Composition

- **Desktop Layout:** Uses a 2-column grid (`max-w-6xl`) to avoid a narrow, dashboard-like column. The left column contains primary content (Greeting, Priority, Your Next) while the right column contains secondary content (Progress, Explore Campus).
- **Typography:** Reduces uppercase treatment to subtle section headers. Does not make everything a card.
- **Card Usage:** Cards are strictly used where they improve grouping (like Priority). The "Your Next" event uses a lightweight outlined container with strong hover states, rather than a heavy material card.

---

# 12. Loading & Error Handling

## Loading (Section-Aware Skeletons)
Do not replace the entire page with a spinner. Use section-aware skeletons for independent data sources (Profile, Registrations, Progress). 

## Error Handling
Errors must be scoped to the affected Home section using a dedicated `ErrorState` component.

Example:
```text
Your Next
[!] Couldn't load events. We couldn't retrieve your upcoming schedule. [ Retry ]
```
A failure in one query (e.g., dashboard summary) must not crash the entire Home page.

---

# 13. Accessibility & Performance

- **Performance**: Independent queries run in parallel using React Query. No duplicate fetching.
- **Responsive**: Content collapses to a single column on mobile web. Strong editorial hierarchy is maintained.
- **Visual Design**: Uses existing M3 tokens and components exclusively. No arbitrary colors or gradients.

---

# 14. Implementation References

The Home UX uses the following verified APIs:
- **Greeting**: `GET /users/me` via `useCurrentUser()`
- **Registrations (Priority & Your Next)**: `GET /v1/users/me/registrations` via `useMyRegistrations()`
- **Progress**: `GET /v1/dashboard/summary` via `useDashboardSummary()`

### Excluded (Non-Goals):
- Speculative API endpoints (`GET /users/me/invitations` or `GET /v1/home/feed`)
- Notification settings prompts (Belongs in Profile)
- Public event discovery feeds (Belongs in Campus)
