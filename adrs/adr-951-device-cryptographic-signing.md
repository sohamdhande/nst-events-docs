# ADR-951: Device Cryptographic Signing

## Status
Rejected for V1

## Context
API requests can theoretically be spoofed to bypass location checks.

## Decision
We reject implementing Device Cryptographic Signing (verifying requests from genuine mobile secure enclaves) for V1.

## Consequences
Positive: Saves significant engineering time during the initial 2-month build phase. Complexity outweighs the benefit.
Negative: Leaves a highly advanced attack vector open for determined API spoofers.

## Alternatives Considered
* **Implementing Secure Enclave signing**: Complexity outweighs the benefit for a campus app V1.

## Future Impact
Can be revisited later if API spoofing becomes a measurable issue. For now, standard JWTs and geofencing will suffice.
