# Kubernetes Configuration

This document defines the intended Kubernetes architecture for `nst-events` and addresses current discrepancies found in the repository's configuration.

## 1. Intended Kubernetes Architecture

### Namespaces
We will use dedicated namespaces to isolate environments:
* `nst-events-prod`: Production workloads and data.
* `nst-events-staging`: Staging workloads and data.

### Workloads
* **Dashboard**: `Deployment` managing Next.js pods.
* **API**: `Deployment` managing Express.js pods.
* **Worker**: `Deployment` managing the background processor.
* **Database**: `StatefulSet` managing a single PostgreSQL instance per environment.

### Replica Strategy & HA
* **API & Dashboard**: Should run with **2+ replicas** for High Availability.
* **Worker**: Must run with strictly **1 replica** to avoid race conditions in scheduled jobs and queue processing, unless the codebase explicitly implements distributed locks.
* **PostgreSQL**: Runs as **1 replica** backed by a highly available Longhorn volume.

### Networking & Exposure
* **Services**: Standard `ClusterIP` services for internal routing.
* **Ingress**: Standard Kubernetes `Ingress` resources routing traffic to the Dashboard and API services.
* **NetworkPolicies**: Default deny-all ingress for the database, with explicit exceptions allowing only the API and Worker pods to communicate on port 5432.

### Configuration & Secrets
* Environment variables and sensitive credentials (e.g., `DATABASE_URL`, `JWT_SECRET`) will be mounted securely via Kubernetes `Secret` resources.

### Probes & Resources
* **Liveness/Readiness Probes**: Configured to ping `/health` endpoints on the API and Worker to ensure traffic is only sent to healthy pods.
* **Resource Requests/Limits**: Strict CPU (`m`) and Memory (`Mi`) limits applied to all containers to prevent noisy-neighbor issues on the shared cluster.

---

## 2. Discrepancies & Required Changes

> [!WARNING]
> **Repository Discrepancies vs Cluster Reality**
> The current YAML files in the `nst-events` repository (`infrastructure/kubernetes/`) contain inaccurate assumptions that **must be fixed** before deployment.

### Ingress Configuration
* **Current Repository State**: `infrastructure/kubernetes/ingress.yaml` specifies `nginx.ingress.kubernetes.io` annotations and targets `api.nst-events.local`.
* **Verified Cluster Reality**: The cluster uses **Traefik**, not Nginx.
* **Required Change**: The Ingress manifests must be rewritten to remove Nginx annotations, utilize the `cert-manager.io/cluster-issuer: letsencrypt-prod` annotation, and target the actual production domains (e.g., `api.nst-events.nstsdc.org`).

### StatefulSet Configuration
* **Current Repository State**: `postgres-statefulset.yaml` requests a 10Gi volume but omits the `storageClassName`.
* **Verified Cluster Reality**: The cluster utilizes `longhorn` for persistent block storage.
* **Required Change**: The `volumeClaimTemplates` in the StatefulSet must explicitly specify `storageClassName: longhorn`.

### Dashboard Manifests
* **Current Repository State**: No Kubernetes manifests exist for the Dashboard.
* **Required Change**: Create `dashboard-deployment.yaml` and `dashboard-service.yaml`.
