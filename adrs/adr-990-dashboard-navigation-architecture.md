# ADR-990: Dashboard Navigation Architecture

## Status
Accepted

## Context
NST-Events requires a robust, scalable dashboard for operational management by faculty and club leaders. We must establish strict rules for routing, state management, and permission boundaries to prevent UX bloat and security leakage.

## Decision
We adopt a Single Dashboard Shell architecture utilizing soft navigation, governed by the Two-Click Rule, Command Parity Rule, URL State Rule, and State Isolation Rule. Events are modeled dynamically with Multi-Club ownership, and contexts are driven by `active_campus_id`.

## Consequences
* Positive: Extremely fast UX. Predictable, deep-linkable URLs.
* Negative: High complexity in maintaining state isolation during role switching.
