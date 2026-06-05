# ADR-952: GPS Spoofing Strategy

## Status
Accepted (Basic Detection Only)

## Context
Students frequently use GPS mocking apps to bypass geofences.

## Decision
We will implement **Basic Detection Only**. The mobile app will use native APIs to check if a mock location provider is active and block check-in using basic heuristics. We will not use advanced anti-spoofing techniques.

## Consequences
Positive: Catches 90% of casual spoofing attempts with minimal development effort.
Negative: Advanced users compiling their own ROMs or using rooted hiding tools will bypass detection.

## Alternatives Considered
* **Advanced third-party anti-spoofing SDKs**: Rejected due to cost and integration overhead.

## Future Impact
Provides a solid baseline defense. Will be monitored via analytics to see if escalation is needed.
