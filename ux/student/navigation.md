# Student Navigation

This document defines the physical and contextual navigation structures for the student web experience.

## Primary Navigation (Global Header/Sidebar)
Depending on responsive breakpoints, this is either a top navbar (desktop) or a bottom tab bar (mobile).
- **Home** (default route `/`)
- **Campus** (`/campus`)
- **My Events** (`/my-events`)
- **Profile** (`/profile`)

*Contextual overlay*: The **Notification Bell** exists globally in the header, revealing a dropdown or drawer.

## Secondary Navigation (Sub-tabs)
Used within major areas to partition information without losing context.

- **Campus Sub-navigation**:
  - Discover (`/campus/discover`)
  - Leaderboard (`/campus/leaderboard`)
  - Clubs (`/campus/clubs`)

- **My Events Sub-navigation**:
  - Upcoming (`/my-events/upcoming`)
  - Waitlist (`/my-events/waitlist`)
  - Past (`/my-events/past`)

## Contextual Navigation (State-Driven)
The UI dynamically surfaces links based on the backend state.
- **If attendance session is active**: A persistent "Live Scan" floating action button (FAB) appears on Home and the Event Detail page.
- **If user has pending team invites**: A banner appears on Home linking directly to the Team Invitation resolution modal.

## Deep Links & URLs
The web app must support direct sharing.
- Event Details: `/events/:id` (accessible from anywhere, does not force a specific parent tab active, though structurally belongs under Campus).
- Club Details: `/clubs/:id`
- Dispute details: `/attendance/disputes/:id`
