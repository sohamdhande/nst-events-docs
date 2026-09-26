# NST Events Documentation Portal

This repository (`nst-events-docs`) is the **single canonical source of truth** for all documentation, architecture decisions, and feature specifications for the NST Events platform.

The codebase resides separately in the [`nst-events`](https://github.com/nst-sdc/nst--events) repository.

The documentation is organized by domain ownership rather than technical implementation.

---

## 🏛 Architecture & Governance

### [Engineering](./docs/engineering/README.md)
* **Purpose**: The constitutional laws and standards governing how software is built in this repository.
* **Ownership**: Principal Architect / Engineering Team.
* **Audience**: All contributors, AI Agents, Reviewers.
* **Relationship**: Overrides all other implementation documentation. Contains ADRs.

### [Architecture](./docs/architecture/README.md)
* **Purpose**: High-level system diagrams, flowcharts, and structural documentation.
* **Ownership**: Architecture Team.
* **Audience**: Engineers, Product Managers.
* **Relationship**: Provides the visual mapping of the contracts defined in API and Database.

### [Cluster & Infrastructure](./docs/cluster/README.md)
* **Purpose**: Full bare-metal college cluster architecture, 7-node topology, Cloudflare tunnels, Traefik ingress, Longhorn 3-way replicated storage, self-hosted CI/CD, and disaster recovery runbooks.
* **Ownership**: DevOps / SRE / Platform Engineering.
* **Audience**: All Engineers, SREs, Auditors, Operators, and New Team Members.
* **Relationship**: Complete zero-code-reading operational guide for how the entire platform runs on physical hardware.

### [Security](./docs/security/README.md)
* **Purpose**: Threat models, JWT strategies, and cryptographic signing guidelines.
* **Ownership**: Security Team / Platform Admins.
* **Audience**: Backend Engineers, Security Auditors.
* **Relationship**: Secures the boundaries defined in Database and API.

---

## 💻 Systems & Implementation

### [API](./docs/api/README.md)
* **Purpose**: API routing matrices, edge function logic, and RPC contracts.
* **Ownership**: Backend Engineering.
* **Audience**: Frontend Engineers, Mobile Engineers.
* **Relationship**: Consumed by Mobile and Dashboard.

### [Backend](./docs/backend/README.md)
* **Purpose**: Node.js worker implementation, core job logic, and event scheduling.
* **Ownership**: Backend Engineering.
* **Audience**: Backend Engineers, DevOps.
* **Relationship**: Executes long-running tasks for the API.

### [Database](./docs/database/README.md)
* **Purpose**: Prisma schemas, Row-Level Security (RLS) policies, and indexing strategies.
* **Ownership**: Database Administrators / Backend Engineering.
* **Audience**: All Engineers.
* **Relationship**: The foundational layer for Security and Backend.

---

## 📱 Product & Interfaces

### [Product](./docs/product/README.md)
* **Purpose**: Feature specifications, leaderboard scoring rules, and role definitions.
* **Ownership**: Product Management.
* **Audience**: All Stakeholders, Engineers.
* **Relationship**: Defines *what* needs to be built across all surfaces.

### [Mobile](./docs/mobile/README.md)
* **Purpose**: React Native / Expo architecture, navigation state, and user flows.
* **Ownership**: Mobile Engineering.
* **Audience**: Mobile Engineers, Designers.
* **Relationship**: Consumes the API and Frontend Design System.

### [Dashboard](./docs/dashboard/README.md)
* **Purpose**: Next.js administration web interface, operations mode, and analytics.
* **Ownership**: Frontend Engineering.
* **Audience**: Frontend Engineers, Club Admins.
* **Relationship**: Consumes the API and Frontend Design System.

### [Frontend Design System](./docs/frontend/README.md)
* **Purpose**: Shared UI components, typography, design tokens, and accessibility standards.
* **Ownership**: Design / Frontend Engineering.
* **Audience**: Frontend and Mobile Engineers.
* **Relationship**: Defines the visual implementation for Mobile and Dashboard.

---

## ⚙️ Operations & Support

### [Operations](./docs/operations/README.md)
* **Purpose**: Historical meeting notes, sprint planning, and non-technical coordination.
* **Ownership**: Project Management.
* **Audience**: Product Managers, Leads.
* **Relationship**: Supports product delivery.

### [Templates](./docs/templates/README.md)
* **Purpose**: Standardized markdown templates for ADRs, meetings, and feature specs.
* **Ownership**: Engineering.
* **Audience**: Document Authors.
* **Relationship**: Used to bootstrap new documents.

### [Assets](./docs/assets/README.md)
* **Purpose**: Centralized storage for diagrams, SVGs, logos, and shared images.
* **Ownership**: Design / Architecture.
* **Audience**: Document Authors.
* **Relationship**: Embedded across all documentation.

### [Archive](./docs/archive/README.md)
* **Purpose**: Storage for deprecated, superseded, or historical documents.
* **Ownership**: Engineering.
* **Audience**: Historians, Auditors.
* **Relationship**: Replaces deleted knowledge to prevent context loss.

---

For the high-level project overview, see [PROJECT_CONTEXT.md](./PROJECT_CONTEXT.md).
