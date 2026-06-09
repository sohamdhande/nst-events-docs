# NST-Events

## Project Vision
NST-Events is the definitive campus operating platform for Newton School of Technology (NST), Pune. It is designed to serve 3000+ students with a robust, scalable, and highly available architecture. The platform centralizes event management, club operations, dynamic QR attendance tracking, and campus-wide communications.

## Repository Purpose
This repository serves as the **Single Source of Truth (SSOT)** for the NST-Events project. 
It contains all architectural decisions, project planning, meeting notes, research, and documentation. No implementation code lives here. It is designed to be easily consumed by new developers onboarding to the project and by AI assistants acting as project memory.

## Documentation Structure
The repository is organized as follows to ensure scalability and maintainability:

* **`/docs`**: Core system documentation. This is heavily populated with V1 Frozen Architecture:
  * `/docs/backend/`: Primary implementation blueprint for developers (Roadmap, Task Breakdown, Prisma schemas, API contracts).
  * `/docs/architecture/`: High-level stack and system definitions.
  * `/docs/database/`: Complete schema, indexes, queries, relationships.
  * `/docs/api/`: Routing, RPC catalogs, Express route handlers, queues.
  * `/docs/security/`: RLS architecture, JWT strategy, threat models.
  * `/docs/mobile/`: Mobile UX navigation, priorities, state management.
  * `/docs/dashboard/`: Web Dashboard soft navigation, Operations mode, routing rules.
  * `/docs/design-system/`: Design tokens, component specs, accessibility, and multi-campus theming.
  * `/docs/diagrams/`: System architecture, sequence diagrams, and flowcharts mapped via Mermaid.
  * `/docs/features/`: Core business logic (events, roles, attendance).
* **`/adrs`**: Architecture Decision Records (ADRs). Immutable records of technical choices.
* **`/research`**: Technical spikes, feasibility studies, and technology evaluations.
* **`/planning`**: Sprint plans, roadmaps, and delivery tracking.
* **`/meetings`**: Minutes of meetings, action items, and alignment records.
* **`/design`**: UI/UX guidelines, design system rules, and wireframe references.

## How To Use This Repository
1. **Start Here**: Read the `MASTER_CONTEXT.md` for the current state of the project.
2. **Reviewing V1**: To understand how the app works, read the `docs/backend/`, `docs/database/`, `docs/api/`, and `docs/security/` sections. The `docs/backend/` folder contains the explicit, step-by-step roadmap and task breakdown required to build the application.
3. **Making a Decision**: Any significant technical decision must be recorded as an ADR in `/adrs` using the `ADR-000-template.md`.
4. **Planning Work**: Use `/planning` to document sprint goals and deliverables.

## Decision Making Process
1. **Research phase**: Document findings in `/research`.
2. **Proposal**: Draft an ADR.
3. **Review**: Core team reviews the ADR.
4. **Acceptance**: ADR is merged and its status is set to 'Accepted'.
5. **Implementation**: Code repositories are updated to reflect the ADR.

## Architecture Workflow
- Changes to the architecture must first be discussed in a technical meeting (recorded in `/meetings`).
- A subsequent ADR is required.
- Following ADR acceptance, the respective `/docs/` folders must be updated to maintain consistency.
