# ADR-013: Use Dynamic QR Attendance

## Status
Accepted

## Context
We need a fast, secure way to check students into events.

## Decision
Use Dynamic QR Attendance.

## Consequences
Positive: Static QR codes are vulnerable to screenshot sharing. Dynamic QR prevents proxy attendance.
Negative: Requires active internet or synchronized offline validation.

## Alternatives Considered
* **Static QR**: Rejected (see ADR-901).

## Future Impact
Forms the core mechanism for event check-ins.
