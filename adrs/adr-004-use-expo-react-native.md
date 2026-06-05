# ADR-004: Use Expo React Native

## Status
Accepted

## Context
We need to deploy a mobile application quickly to both iOS and Android platforms.

## Decision
Use Expo React Native.

## Consequences
Positive: Fast development, OTA updates, camera support, notifications, and reduced native complexity.
Negative: Dependent on Expo ecosystem.

## Alternatives Considered
* **Swift / Kotlin Native**: Rejected due to timeline and team size.
* **React Native (Bare)**: Rejected due to unnecessary native build complexities.

## Future Impact
Ensures the mobile app can be built entirely by JavaScript/TypeScript developers.
