# ADR-001: Use Custom Node.js + Express + TypeScript + Prisma as the Primary Backend Platform

## Status
Accepted

## Context
The project requires a robust backend for a 3,000-student campus platform with a 6-person team and a 2-month build timeline. The team evaluated managed Backend-as-a-Service (BaaS) platforms and fully custom backend approaches.

Key drivers for this decision:
- The platform will be deployed on the NST Cluster, a 5-node bare-metal K3s Kubernetes cluster at Newton School of Technology, Pune — providing a self-hosted, cost-controlled runtime without external service dependencies.
- The team is primarily TypeScript-fluent, and the frontend stacks (Expo + Next.js) are already TypeScript. Keeping the backend in the same language minimizes context switching.
- The business logic is complex: pessimistic registration locks, multi-step event approval workflows, geofence-validated attendance, and audit-immutable leaderboard scoring all require precise, testable server-side control that is difficult to express purely in database functions or BaaS configuration.

## Decision
Use a custom **Node.js + Express + TypeScript** backend with **Prisma ORM** connecting to a **PostgreSQL** database.

All client applications (Expo mobile app and Next.js dashboard) communicate exclusively with the Express API. Clients never interact with the database directly.

## Consequences

**Positive:**
- Full control over the API layer, request lifecycle, error handling, and response shaping — no vendor abstractions in the critical path.
- No vendor lock-in. The entire backend is standard Node.js and can be hosted anywhere.
- Custom business logic (registration locking, approval workflows, QR token generation) lives in standard Express route handlers and service modules — fully unit-testable.
- Deployable on the NST Cluster without depending on any external paid service.
- Team builds transferable backend engineering skills in a standard, widely-documented stack.
- Prisma provides type-safe database access with TypeScript types generated directly from the database schema.

**Negative:**
- More backend code to write and maintain compared to a managed BaaS.
- The team is responsible for authentication, API design, deployment, and infrastructure — not abstracted away.
- Longer initial development time for boilerplate (middleware, auth flows, error handling) compared to a managed platform.

## Alternatives Considered

- **Supabase**: Rejected because of potential cost at scale, vendor lock-in to Supabase-specific features (PostgREST auto-API, proprietary Edge Functions runtime), and the availability of the NST Cluster as a capable self-hosted deployment target that eliminates the core value proposition of a managed BaaS.
- **NestJS**: Considered but rejected for V1 due to steeper learning curve and heavier framework overhead for the team's current experience level. Can be adopted in a future major version.
- **FastAPI / Python**: Rejected because the team is TypeScript-fluent across all layers. Introducing a Python backend would create a language boundary without sufficient benefit.
- **Firebase**: Rejected because the platform requires relational data modeling, row-level security, and complex transactional logic that document databases do not support well.

## Future Impact
Provides the team with a stable, fully-owned backend that scales with the platform. Because the Express API is a standard HTTP service, it can be containerized, load-balanced, and extended without rearchitecting the data layer. The Prisma schema becomes the single source of truth for TypeScript types across the backend.
