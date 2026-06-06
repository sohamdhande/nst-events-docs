# Design Principles

## Predictable Density
Interfaces must provide high information density for operational efficiency without feeling cluttered. We achieve this through strict typography scales and consistent grid spacing, avoiding unnecessary whitespace while preventing visual fatigue.

## Contextual Friction
Friction is introduced deliberately. Destructive actions (like rejecting an event) or high-stakes actions (Swipe To Check-In) require explicit, deliberate gestures. Standard navigation and passive consumption remain frictionless.

## Single Source Token Architecture
Design tokens are the absolute source of truth. No hardcoded hex values or arbitrary pixel spacing are allowed in component code. Everything maps back to the semantic token tables.

## Graceful Scalability
The UI is engineered to scale across screen sizes, device capabilities, and organizational complexity. As new features or campuses are added, the UI absorbs them through dynamic token injection rather than hardcoded edge cases.
