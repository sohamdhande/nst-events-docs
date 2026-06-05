# ADR-904: Reject Campus IP Validation

## Status
Rejected

## Context
We considered validating attendance against the campus Wi-Fi IPs.

## Decision
Reject Campus IP Validation.

## Consequences
Positive: Students often use mobile data and hotspots. IP validation would block legitimate users.
Negative: Loses one potential location signal.

## Alternatives Considered
* **Geofencing**: Used instead.

## Future Impact
Maintains an accessible check-in experience.
