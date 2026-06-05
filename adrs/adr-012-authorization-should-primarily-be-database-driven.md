# ADR-012: Authorization should primarily be database-driven

## Status
Accepted

## Context
We need a foolproof way to manage permissions across multiple client applications.

## Decision
Authorization should primarily be database-driven.

## Consequences
Positive: Security and consistency across all touchpoints.
Negative: Policies can be harder to test than application logic.

## Alternatives Considered
* **Frontend gating only**: Rejected as completely insecure.

## Future Impact
Ensures long-term data integrity.
