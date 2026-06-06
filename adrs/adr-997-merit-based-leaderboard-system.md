# ADR-997: Merit-Based Leaderboard System

## Status
Accepted

## Context
The previous leaderboard design was primarily attendance-focused. This model is insufficient for rewarding true campus engagement. Registration alone or waitlist placement must not generate points. We need a system that rewards meaningful contributions: organizing, speaking, volunteering, mentoring, and performing well in competitions.

## Decision
We adopt a Merit-Based Participation Model. 
Points are derived strictly from explicit backend actions. Manual point insertion is prohibited except for disciplinary adjustments by Platform Admins via audited RPCs.
We introduce `participation_role` into `event_registrations` and a new `event_results` table.

## Consequences
* **Positive**: Creates a highly motivating, fair gamification system that recognizes diverse student efforts.
* **Negative**: Requires additional operational overhead for Club Admins to assign roles and submit results after an event.
