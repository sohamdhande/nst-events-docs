# Student UX Principles

These principles guide the design of the NST Events student web experience. They are derived from product requirements and backend constraints.

## 1. Action-Oriented Home
Home is not a static feed. It is a contextual dashboard prioritizing immediate student needs (e.g., active attendance sessions, pending team invites, newly promoted waitlists).

## 2. No Optimistic UI for Commitments
Actions that change a student's commitment (Registering, Canceling, Leaving a team, Disputing attendance) must wait for explicit backend confirmation before updating the UI state. The application must show clear loading states during these mutations.

## 3. Immediate State Clarity
The student should never wonder about their status. Badges and states (`REGISTERED`, `WAITLISTED`, `PENDING INVITATION`, `PRESENT`) must be unambiguous and highly visible. 

## 4. Contextual Actions Only
Buttons and actions appear only when valid. A student should not see a "Check In" button unless an attendance session is actively open. They should not see "Cancel Registration" if the event is locked.

## 5. Waitlists are Asynchronous
Waitlist promotion is automatic via the backend. The UX does not require an "Accept" step for waitlists. Instead, it relies on real-time notifications and state changes to inform the student of their promotion.

## 6. Neutral Fraud Messaging
When an attendance scan fails due to location restrictions, device fingerprinting, or session expiration, the messaging must remain neutral (e.g., "Verification Failed" or "Location out of bounds"). It must not expose the internal fraud detection mechanics or accuse the student.

## 7. Hide Administrative Concepts
The student does not care about `is_locked`, `DRAFT`, or `PENDING_APPROVAL`. They only see events that are `PUBLISHED` or `ARCHIVED`. Administrative states are filtered out of the student UX entirely.
