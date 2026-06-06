# Dashboard Architecture

## Single Dashboard Shell
The NST-Events web interface operates within a Single Dashboard Shell. All state transitions, role changes, and feature access happen inside this persistent shell without full page reloads.

## Soft Navigation & Routing Strategy
We utilize soft navigation (client-side routing) via Next.js to ensure immediate UX responsiveness.

## Layout Structure
* **Dynamic Sidebar**: Left-aligned, collapsible context menu.
* **Top Bar**: Search/Command Palette, Notifications, Profile Context.
* **Content Canvas**: Primary workspace with Horizontal Context Tabs.

## Context Injection
Data context is injected at the layout root based on the user's active role. E.g., if rendering `/events/123`, the shell verifies the user's permissions and injects `role: 'CLUB_ADMIN'` into the component tree.

## State Preservation
The UI state is deliberately volatile to prevent stale permission leakage. Navigating between distinct roles forces a state destruction and context reset.

## Future Scalability
The shell is built to support dynamic domain injection (e.g., dynamically loading a "Recruitment" module in year 2 without altering the shell architecture).
