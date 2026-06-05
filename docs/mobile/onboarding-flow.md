# Onboarding Flow (New User Empty State)

## Context
If a user logs in and has no clubs, no registrations, and no activity, the Home tab defaults to an Onboarding State.

## NST Getting Started Checklist
The UI displays a step-by-step checklist to guide the user:
1. Complete Profile
2. Join a Club
3. Allow Notifications
4. Register for First Event
5. Attend First Event

## Progression Behavior
As the user completes actions across the app, the Home feed delta sync recognizes the state changes. The checklist dynamically crosses off items and eventually disappears once the user achieves a critical mass of activity, replaced entirely by the standard Campus Dashboard layout.
