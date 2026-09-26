# NST Events — Full Cluster Architecture & Infrastructure Master Guide

This directory is the comprehensive, authoritative guide to how the **NST Events** platform operates on the college bare-metal Kubernetes (K3s) cluster.

Anyone reading this guide will gain a complete, end-to-end understanding of the platform's hardware topology, network paths, container workloads, data persistence, automated CI/CD deployments, security boundaries, and disaster recovery procedures—**without needing to inspect a single line of source code**.

---

## 1. Executive Summary

**NST Events** is an enterprise-grade campus event and student engagement platform built for Newton School of Technology (NST) / Ajeenkya DY Patil University (ADYPU). It powers student event discovery, team formation, live QR-code check-ins with spatial geofencing, real-time push notifications, club management, and campus-wide gamified leaderboards.

The entire production ecosystem runs on-premise on a **7-node bare-metal server cluster** located in the campus server room, managed via **K3s (lightweight Kubernetes)**. Public internet traffic is securely routed through a zero-trust **Cloudflare Tunnel** and reverse-proxied by **Traefik**, with Let's Encrypt TLS certificates automated by **cert-manager**. State is maintained in a **PostgreSQL 16 + PostGIS** database backed by **Longhorn 3-way synchronous block storage replication**, and asynchronous background tasks (such as push notification delivery) are processed by a dedicated worker engine.

---

## 2. End-to-End System Architecture Map

The following diagram illustrates the complete request and data lifecycle from external end-users to the physical hardware and back:

```text
========================================================================================================================
                                                  EXTERNAL TRAFFIC LAYER
========================================================================================================================
 [ Student Mobile App (Expo) ]                      [ Student / Coordinator / Admin Browser ]
               │                                                       │
               │ HTTPS (Port 443)                                      │ HTTPS (Port 443)
               ▼                                                       ▼
 ┌────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
 │                                           CLOUDFLARE EDGE NETWORK                                                  │
 │  • DNS Resolution: events.nstsdc.org / api.nst-events.nstsdc.org                                                   │
 │  • Edge TLS Termination (Universal SSL) & HTTP -> HTTPS Redirection                                                │
 │  • DDoS Mitigation & Cloudflare Web Application Firewall (WAF)                                                     │
 │  • 100-Second Idle Connection Management (kept alive by 30s application heartbeats)                                │
 └────────────────────────────────────────┬───────────────────────────────────────────────────────────────────────────┘
                                          │ Encrypted Tunnel (Outbound Egress, No Open Inbound Router Ports)
                                          ▼
========================================================================================================================
                                              CAMPUS BARE-METAL CLUSTER LAYER
========================================================================================================================
                                          │
                                          ▼
 ┌────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
 │  CONTROL PLANE NODE: nst-n1 (192.168.136.145)                                                                      │
 │  ├─ cloudflared Daemon (Tunnel: krushn-node1)                                                                      │
 │  │    └─ Forwards HTTP traffic to localhost:80                                                                     │
 │  │                                                                                                                 │
 │  ├─ Traefik Ingress Controller (IngressClass: traefik)                                                             │
 │  │    ├─ Host: events.nstsdc.org                                                                                   │
 │  │    │    ├─ /health, /ready, /auth/*, /v1/*, /webhooks/* ──> nst-api-service (ClusterIP: Port 80)               │
 │  │    │    └─ / (All Web UI Routes) ─────────────────────────> nst-dashboard-service (ClusterIP: Port 80)         │
 │  │    └─ Host: api.nst-events.nstsdc.org                                                                           │
 │  │         └─ /* ─────────────────────────────────────────────> nst-api-service (ClusterIP: Port 80)               │
 │  │                                                                                                                 │
 │  └─ Platform Management: cert-manager (letsencrypt-prod), K3s API Server, etcd, Rancher, Fleet                    │
 └────────────────────────────────────────┬───────────────────────────────────────────────────────────────────────────┘
                                          │ Internal Pod Overlay Network (Flannel CNI)
                                          │ [NodeAffinity: Excludes nst-n1, Prefers nst-n6]
                                          ▼
 ┌────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
 │  COMPUTE WORKLOADS (Nodes nst-n2 through nst-n7)                                                                   │
 │                                                                                                                    │
 │  ┌──────────────────────────────────────────────┐        ┌───────────────────────────────────────────────────────┐  │
 │  │  nst-dashboard Pods (Replicas: 2, HA)        │        │  nst-api Pods (Replicas: 2, HA)                       │  │
 │  │  • Next.js 15 App Router (Standalone Node)   │        │  • Express.js REST & SSE Server (Node 20)             │  │
 │  │  • Listens on Port 3000                      │        │  • Listens on Port 3001                               │  │
 │  │  • Server-Side Rendering (SSR) & Static HTML │        │  • Stateless JWT Auth (HTTP-Only Secure Cookies)      │  │
 │  │  • Reverse Proxy Rewrites:                   │        │  • Server-Sent Events (SSE) live streams              │  │
 │  │    Client API requests (/clubs, /users)      │───────>│  • PostGIS Spatial Geofencing Attendance Validation   │  │
 │  │    forwarded to internal nst-api-service     │ (HTTP) │  • Database Connection Pooling (Prisma Client)        │  │
 │  └──────────────────────────────────────────────┘        │  • Emits jobs to notification_jobs queue table        │  │
 │                                                          │  • PostgreSQL LISTEN/NOTIFY for cross-pod SSE sync    │  │
 │                                                          └──────────────────────────┬────────────────────────────┘  │
 │                                                                                     │                               │
 │                                                                                     │ SQL Queries & Queue Inserts   │
 │                                                                                     ▼ (TCP Port 5432)               │
 │  ┌──────────────────────────────────────────────┐        ┌───────────────────────────────────────────────────────┐  │
 │  │  nst-worker Pod (Replicas: 1, Recreate)      │        │  nst-postgres Pod (StatefulSet, Replicas: 1)          │  │
 │  │  • Node 20 Background Job Processor          │        │  • PostgreSQL 16 + PostGIS 3.4                        │  │
 │  │  • Internal Health/Metrics on Port 3002      │        │  • Listens on internal Port 5432                      │  │
 │  │  • Atomic Queue Polling:                     │<──────>│  • Isolated by NetworkPolicy (API + Worker Only)      │  │
 │  │    SELECT ... FOR UPDATE SKIP LOCKED         │  SQL   │  • Persistent Volume: 10Gi on /var/lib/postgresql/data│  │
 │  │  • Expo Push Notification Delivery Pipeline  │        └──────────────────────────┬────────────────────────────┘  │
 │  │  • Push receipt verification & error backoff │                                   │                               │
 │  └──────────────────────┬───────────────────────┘                                   │ iSCSI Block Writes            │
 └─────────────────────────┼───────────────────────────────────────────────────────────┼───────────────────────────────┘
                           │ Outbound HTTPS                                            ▼
                           ▼                                ┌─────────────────────────────────────────────────────────┐
               ┌────────────────────────┐                   │  LONGHORN DISTRIBUTED STORAGE LAYER                     │
               │ Expo Push Notification │                   │  • StorageClass: longhorn (3-Way Synchronous Mirroring) │
               │ Service (APNs / FCM)   │                   │  • Replicated across 6 physical node NVMe disks         │
               └────────────────────────┘                   │  • Automatic volume re-attachment on node failover      │
                                                            │  • Online dynamic volume expansion supported            │
                                                            └─────────────────────────────────────────────────────────┘
```

---

## 3. Core Component Matrix

The table below summarizes every core component of the NST Events infrastructure, its runtime characteristics, high-availability posture, and network exposure:

| Component                         | Workload Type                  | Replicas         | Internal Port | Ingress Host / Path                                                                  | Storage Mechanism                                           | Health / Readiness Probe                                          | Primary Function                                                                            |
| :-------------------------------- | :----------------------------- | :--------------- | :------------ | :----------------------------------------------------------------------------------- | :---------------------------------------------------------- | :---------------------------------------------------------------- | :------------------------------------------------------------------------------------------ |
| **`nst-dashboard`**       | Kubernetes`Deployment`       | 2 (HA)           | 3000          | `events.nstsdc.org/`                                                               | Stateless ephemeral                                         | HTTP GET`/login` (Port 3000)                                    | Web frontend for students, club leads, and platform administrators.                         |
| **`nst-api`**             | Kubernetes`Deployment`       | 2 (HA)           | 3001          | `events.nstsdc.org/{health, ready, auth, v1, webhooks}api.nst-events.nstsdc.org/*` | Stateless ephemeral                                         | HTTP GET`/ready` (DB query)HTTP GET `/health`                 | Core REST API, Google OAuth flow, geofenced attendance, and live SSE event streams.         |
| **`nst-worker`**          | Kubernetes`Deployment`       | 1 (`Recreate`) | 3002          | Internal cluster only (No public ingress)                                            | Stateless ephemeral                                         | HTTP GET`/ready` (DB query)HTTP GET `/health`GET `/metrics` | Asynchronous job consumer (push notifications via Expo, receipt polling, scheduled jobs).   |
| **`nst-postgres`**        | Kubernetes`StatefulSet`      | 1                | 5432          | Internal cluster only (Port 5432, NetworkPolicy enforced)                            | 10Gi PVC (`storageClassName: longhorn`, 3-way replicated) | Exec probe:`pg_isready -U postgres -d nst_events`               | Relational database (users, clubs, events, registrations, attendance, audit logs).          |
| **`nst-postgres-backup`** | Kubernetes`CronJob`          | Scheduled        | N/A           | Off-cluster outbound HTTPS to Cloudflare R2                                          | Ephemeral scratch (`emptyDir`)                            | Exit code verification & SHA-256 checksum match                   | Daily automated`pg_dump -Fc` compressed logical database backup to Cloudflare R2.         |
| **`Traefik`**             | K3s Ingress Controller         | DaemonSet        | 80, 443       | Cluster node interface                                                               | Cluster host networking                                     | K3s internal probe                                                | Ingress traffic router evaluating HTTP`Host` and `PathPrefix` rules.                    |
| **`cloudflared`**         | Systemd Service (on`nst-n1`) | 1 daemon         | 80, 443       | Outbound connection to Cloudflare edge                                               | Host filesystem`/etc/cloudflared/`                        | Systemd service watchdog                                          | Encrypted Cloudflare Tunnel connecting the college LAN to the public internet.              |
| **`cert-manager`**        | Platform Service               | Deployment       | N/A           | Interacts with Let's Encrypt ACME                                                    | Internal cluster secrets                                    | Standard K8s pod probes                                           | Automatically negotiates, issues, and renews TLS certificates (`letsencrypt-prod`).       |
| **`Longhorn`**            | Distributed Storage Engine     | CSI Driver       | N/A           | Internal cluster communication                                                       | 6 physical node disks (~282.8 GB free capacity)             | Longhorn engine manager health checks                             | Synchronously mirrors block data across 3 separate physical machines.                       |
| **`GitHub Runner`**       | Self-Hosted CI/CD Runner       | Host Service     | N/A           | Node`nst-n6` (`192.168.136.150`)                                                 | Host disk (`/home/github-runner/`)                        | GitHub Actions runner heartbeat                                   | Builds images locally on LAN, runs automated test suites, and gates zero-downtime rollouts. |

---

## 4. Documentation Chapters & Reading Paths

To explore specific subsystems in granular detail, follow the dedicated chapters below:

```text
├── 01-hardware-and-nodes.md            Physical Bare-Metal Nodes, OS, Specs & Scheduling Topology
├── 02-networking-and-edge-routing.md   Edge Ingress, Cloudflare Tunnel, TLS & Traffic Flow
├── 03-workload-architecture.md         Compute Workloads, Pods, Scaling & Multi-Replica State
├── 04-storage-and-database.md          PostgreSQL, PostGIS & Longhorn 3-Way Replicated Storage
├── 05-cicd-and-deployment-lifecycle.md CI/CD Pipeline, Self-Hosted Runner & Rollout Gating
├── 06-security-and-isolation.md        Defense-in-Depth, RBAC, Secrets & Token Isolation
└── 07-operations-backup-and-dr.md      Operational Runbooks, Disaster Recovery & Troubleshooting
```

### Recommended Reading Paths

- **For Full-Stack Developers & Contributors**:
  Start with [02-networking-and-edge-routing.md](./02-networking-and-edge-routing.md) to understand how routes reach your services, then read [03-workload-architecture.md](./03-workload-architecture.md) and [04-storage-and-database.md](./04-storage-and-database.md).
- **For DevOps, SREs & Cluster Administrators**:
  Read [01-hardware-and-nodes.md](./01-hardware-and-nodes.md), [05-cicd-and-deployment-lifecycle.md](./05-cicd-and-deployment-lifecycle.md), and [07-operations-backup-and-dr.md](./07-operations-backup-and-dr.md).
- **For Security Auditors & Institutional IT**:
  Read [06-security-and-isolation.md](./06-security-and-isolation.md), [02-networking-and-edge-routing.md](./02-networking-and-edge-routing.md), and [07-operations-backup-and-dr.md](./07-operations-backup-and-dr.md).
