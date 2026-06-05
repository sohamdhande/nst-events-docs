# ADR-019: Support public and private events

## Status
Accepted

## Context
Not all events should be visible to the entire campus.

## Decision
Support public and private events.

## Consequences
Positive: Some events should be visible only to club members.
Negative: Requires feed filtering logic and strict RLS.

## Alternatives Considered
* **All public**: Rejected as it violates privacy for internal meetings.

## Future Impact
Allows clubs to manage their internal operations securely.
