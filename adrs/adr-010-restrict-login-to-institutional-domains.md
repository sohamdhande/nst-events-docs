# ADR-010: Restrict login to institutional domains

## Status
Accepted

## Context
The platform is exclusively for NST students and faculty.

## Decision
Restrict login to `@adypu.edu.in` and `@newtonschool.co`.

## Consequences
Positive: Campus-only access is strictly enforced at the authentication layer.
Negative: Users without these emails cannot use the app.

## Alternatives Considered
* **Manual user whitelist**: Rejected due to high administrative burden.

## Future Impact
Automates user verification.
