# Infrastructure Architecture

This document defines the verified production topology and high-level architecture for the `nst-events` platform. It serves as the authoritative guide for how components interact within the deployed environment.

## 1. Verified Production Topology

The infrastructure relies on a standard Kubernetes architecture with external edge routing provided by Cloudflare.

```text
[ External Traffic ]
        │
[ Cloudflare DNS & Proxy ]  <-- TLS Termination (Edge), DDoS Protection, WAF
        │
[ K3s Traefik Ingress ]     <-- Internal TLS Termination (cert-manager), Path/Host Routing
        │
        ├── Ingress: dashboard.nst-events.nstsdc.org
        │    └── Service (Port 80)
        │         └── Deployment: Next.js Dashboard
        │
        ├── Ingress: api.nst-events.nstsdc.org
        │    └── Service (Port 80)
        │         └── Deployment: Express API
        │
[ Internal Network Only ]
        │
        ├── Deployment: Express Worker (Asynchronous Jobs)
        │
        └── StatefulSet: PostgreSQL + PostGIS (Persistent Database)
```

## 2. Component Responsibilities & Communication

### Edge & Ingress
* **Cloudflare**: Handles public DNS resolution, edge caching, and security (WAF/DDoS). It proxies traffic to the cluster's public IPs.
* **Traefik Ingress Controller**: The cluster's native ingress. It routes traffic based on HTTP host headers (`api.nst-events.nstsdc.org` vs `dashboard.nst-events.nstsdc.org`) to the appropriate internal Kubernetes `Service`.
* **cert-manager**: Automatically provisions and renews Let's Encrypt TLS certificates for the Traefik Ingress routes.

### Application Workloads
* **Dashboard (`apps/dashboard`)**: The Next.js web interface. It does not communicate directly with the database. It makes HTTP REST calls exclusively to the API.
* **API (`apps/api`)**: The core Express.js server. It handles all synchronous business logic, connects directly to PostgreSQL, and pushes asynchronous jobs to the database queue.
* **Worker (`apps/worker`)**: The background job processor. It does not accept inbound HTTP traffic. It polls the database queue (or pgmq) and executes long-running tasks like push notifications. It communicates directly with the database.

### Data Layer
* **PostgreSQL (`packages/database`)**: The single source of truth. It is strictly isolated from the public internet. Only the API and Worker components are permitted to connect to it.

## 3. Environment Isolation (Staging vs. Production)

The architecture supports complete isolation between staging and production environments using Kubernetes namespaces.

* **Staging (`nst-events-staging`)**: A complete replica of the production topology with reduced resource allocations. It uses separate database volumes and separate subdomains (e.g., `staging-api.nst-events.nstsdc.org`).
* **Production (`nst-events-prod`)**: The highly available production environment with strict access controls and full resource allocations.

Both environments run on the same physical K3s cluster but are isolated logically via namespaces, RBAC, and NetworkPolicies.
