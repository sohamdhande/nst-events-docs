# ADR-998: Component Inventory Architecture

## Status
Accepted

## Context
With the Design System and Navigation structures frozen, we require a master inventory of UI components to serve as the blueprint for frontend engineering, Storybook creation, and task tracking. We need to ensure complex missing pieces (like `AuditLogDiffViewer` and `CompetitionResultsTable`) are officially integrated.

## Decision
We freeze the Component Inventory Architecture into 15 specific domains. We formalize the existence of Dashboard-specific, Mobile-specific, and Operations-Mode-specific components, alongside strict Accessibility and Variant rules.

## Consequences
* **Positive**: Provides absolute clarity for React Native and Next.js developers. Prevents scope creep.
* **Future Impact**: Sets the exact scope for Storybook documentation and QA testing.
