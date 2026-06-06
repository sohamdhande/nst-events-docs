# ADR-999: Product Surface Freeze

## Status
Accepted

## Context
With the Component Inventory locked, we need to finalize the specific application screens and workflows that assemble those components. We must incorporate essential edge-cases (Attendance Disputes, Leadership Handovers) while explicitly rejecting scope creep (Optimistic Registration, Guest Passes).

## Decision
We freeze the Product Surface Architecture. We adopt the Attendance Dispute System, Leadership Handover workflow, Competition Results Management, and Multi-Club Event Association rules.
We explicitly reject optimistic UI for registrations.
Guest Event Passes are deferred to future expansions.

## Consequences
* **Positive**: Absolute clarity on the exact number of screens and APIs required for V1.
* **Future Impact**: Guest management will require a V2 architectural shift for external authentication.
