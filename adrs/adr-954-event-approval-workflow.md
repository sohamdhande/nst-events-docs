# ADR-954: Event Approval Workflow

## Status
Accepted

## Context
Events created by clubs require faculty oversight before becoming public to the campus.

## Decision
We will use a strict state-machine approval workflow with the following lifecycle:
`DRAFT` → `PENDING_APPROVAL` → `PUBLISHED` → `ARCHIVED`.

## Consequences
Positive: Ensures faculty can review all content before it goes live.
Negative: Adds friction and delay to event creation for clubs.

## Alternatives Considered
* **Post-creation review (Relaxed)**: Events are public immediately and faculty can take them down. Rejected to prevent inappropriate content from reaching the campus feed.

## Future Impact
Determines the speed at which campus life operates. Requires a dedicated dashboard for Faculty to review pending events efficiently.
