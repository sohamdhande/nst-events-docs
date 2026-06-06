# ADR-995: Design System Architecture

## Status
Accepted

## Context
NST-Events requires a cohesive, highly accessible, and scalable design language that works seamlessly across a Next.js Dashboard and an Expo Mobile app, while supporting future multi-campus theming.

## Decision
We adopt a Token-Driven Architecture utilizing Geist fonts, a borders-over-shadows elevation philosophy, and semantic color mapping via Tailwind CSS and NativeWind. We explicitly reject arbitrary absolute scores for UX quality. Attendance relies on a Swipe To Check-In gesture with a Tap To Confirm accessibility fallback.

## Consequences
* **Positive**: Absolute UI consistency. Instant dark mode and multi-campus support. Highly professional, developer-centric aesthetic.
* **Negative**: High initial engineering setup required for precise token mapping across two distinct platforms.

## Rejected Alternatives
* **Material Design**: Rejected due to heavy reliance on drop-shadows and complex elevation layers.
* **Inter/Orbitron**: Rejected in favor of Geist Sans/Mono for a sharper, distinct identity.
