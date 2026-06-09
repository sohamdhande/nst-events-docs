# ADR-010: Restrict login to institutional domains

## Status
Accepted

## Context
The platform is exclusively for NST students and faculty.

## Decision
Restrict login to `@adypu.edu.in` (students) and `@newtonschool.co` (faculty and management). This is enforced in the Express backend during the Google OAuth callback.

## Consequences
Positive: Campus-only access is strictly enforced at the authentication layer via the Express OAuth callback. No user with a non-institutional email can receive a JWT.
Negative: Users without these institutional emails cannot use the platform.

## Alternatives Considered
* **Manual user whitelist**: Rejected due to high administrative burden.

## Future Impact
Automates user verification.
