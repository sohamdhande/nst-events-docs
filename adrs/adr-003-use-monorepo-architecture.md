# ADR-003: Use Monorepo Architecture

## Status
Accepted

## Context
The platform consists of a mobile app, web dashboard, and Express backend. Code sharing is highly desirable.

## Decision
Use Monorepo Architecture.

## Consequences
Positive: Shared types, shared UI, shared validation, easier AI-assisted development.
Negative: Initial tooling complexity (Turborepo/Yarn Workspaces).

## Alternatives Considered
* **Multiple repositories**: Rejected due to the overhead of keeping API schemas and types in sync.

## Future Impact
Improves developer velocity across the 6-person team.
