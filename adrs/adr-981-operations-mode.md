# ADR-981: Operations Mode

## Status
Accepted

## Context
During high-pressure live events, organizers need a distraction-free, highly responsive interface to handle dynamic QR code generation, live attendance monitoring, and manual overrides.

## Decision
We implement "Operations Mode"—a specialized, locked-down dashboard state that suppresses standard navigation, highlights the active QR TOTP seed, and exposes emergency actions (like Lockdown).

## Consequences
* Positive: Reduces organizer error during live events. Guarantees QR visibility.
* Negative: Requires specialized responsive UI development separate from the standard dashboard layout.
