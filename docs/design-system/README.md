# NST-Events Design System

## Purpose
The NST-Events Design System is the single source of truth for all visual and interactive elements across the platform. It ensures consistency, accelerates development, and guarantees accessibility across both the Next.js Dashboard and the Expo Mobile Application.

## Philosophy & Inspiration
Our visual philosophy is heavily inspired by modern, developer-centric, and highly functional platforms like Linear, Stripe, Vercel, and Notion.
We explicitly reject:
* Legacy ERP styling (dense, unreadable, outdated).
* Material Design shadow-heavy interfaces.
* Consumer social media aesthetics (overly rounded, distracting).

## Architecture Overview
The system is built on a strict **Token Driven Architecture**. Colors, spacing, typography, and motion are defined as atomic design tokens and consumed via Tailwind CSS (Web) and NativeWind (Mobile).

## Relationship with Navigation Systems
The Design System strictly informs the layout boundaries defined in the Dashboard Navigation Architecture and Mobile Navigation Architecture. It provides the exact surface contrasts, borders, and interactive feedback mechanisms required to make soft navigation and state isolation visually apparent.

## Document Map
1. [Design Principles](01-design-principles.md)
2. [Color System](02-color-system.md)
3. [Typography](03-typography.md)
4. [Layout & Spacing](04-layout-and-spacing.md)
5. [Elevation System](05-elevation-system.md)
6. [Motion System](06-motion-system.md)
7. [Component Library](07-component-library.md)
8. [Accessibility](08-accessibility.md)
9. [Dark Mode](09-dark-mode.md)
10. [Design Tokens](10-design-tokens.md)
11. [Cross-Platform Architecture](11-cross-platform-architecture.md)
12. [Future Theming](12-future-theming.md)
