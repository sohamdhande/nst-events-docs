# NST-Events

## Project Vision
NST-Events is the definitive campus operating platform for Newton School of Technology (NST), Pune. It is designed to serve 3000+ students with a robust, scalable, and highly available architecture. The platform centralizes event management, club operations, dynamic QR attendance tracking, and campus-wide communications.

## Repository Purpose
This repository serves as the **Single Source of Truth (SSOT)** for the NST-Events project. 
It contains all architectural decisions, project planning, meeting notes, research, and documentation. No implementation code lives here. It is designed to be easily consumed by new developers onboarding to the project and by AI assistants acting as project memory.

## Documentation Structure
The repository is organized as follows to ensure scalability and maintainability:

* **`/docs`**: Core system documentation (Architecture, Database, Security, Features).
* **`/adrs`**: Architecture Decision Records (ADRs). Immutable records of technical choices.
* **`/research`**: Technical spikes, feasibility studies, and technology evaluations.
* **`/planning`**: Sprint plans, roadmaps, and delivery tracking.
* **`/meetings`**: Minutes of meetings, action items, and alignment records.
* **`/design`**: UI/UX guidelines, design system rules, and wireframe references.

## How To Use This Repository
1. **Start Here**: Read the `MASTER_CONTEXT.md` for the current state of the project.
2. **Making a Decision**: Any significant technical decision must be recorded as an ADR in `/adrs` using the `ADR-000-template.md`.
3. **Planning Work**: Use `/planning` to document sprint goals and deliverables.
4. **Documenting Features**: Use the templates in `/docs/features` for comprehensive capability documentation.

## Decision Making Process
1. **Research phase**: Document findings in `/research`.
2. **Proposal**: Draft an ADR.
3. **Review**: Core team reviews the ADR.
4. **Acceptance**: ADR is merged and its status is set to 'Accepted'.
5. **Implementation**: Code repositories are updated to reflect the ADR.

## Architecture Workflow
- Changes to the architecture must first be discussed in a technical meeting (recorded in `/meetings`).
- A subsequent ADR is required.
- Following ADR acceptance, `/docs/architecture` and `/docs/database` must be updated to maintain consistency.
