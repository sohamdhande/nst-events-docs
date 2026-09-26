# Deployment Model

This document outlines the lifecycle of code moving from the repository to the live cluster.

## 1. Environment Isolation

We maintain two distinct environments on the shared Kubernetes cluster, separated by namespaces:

* **Staging (`nst-events-staging`)**: Receives automated deployments from the `develop` or `main` branch. Used for QA, integration testing, and stakeholder review.
* **Production (`nst-events-prod`)**: Receives controlled, tagged releases.

Both environments share identical architectural topologies but utilize separate databases, separate secrets, and separate routing domains.

## 2. CI/CD Pipeline (GitHub Actions)

The repository's current deployment workflow (`.github/workflows/deploy.yml`) is a stub. The final CI/CD pipeline must implement the following flow:

1. **Build & Test**: Run typechecking, linting, unit tests, and `pnpm build`.
2. **Containerization**: Build Docker images for the API, Worker, Dashboard, and the custom PostgreSQL image.
3. **Registry Push**: Push the tagged images to a container registry (e.g., GitHub Container Registry - GHCR).
4. **Kubernetes Deployment**: Update the image tags in the Kubernetes manifests and apply them to the cluster.

### Kubernetes RBAC for CI/CD
To securely deploy from GitHub Actions without exposing a highly privileged `cluster-admin` kubeconfig:
1. We will create a scoped `ServiceAccount` strictly within the `nst-events-prod` and `nst-events-staging` namespaces.
2. A `Role` and `RoleBinding` will restrict this account to managing only Deployments, Services, Ingresses, and StatefulSets within those specific namespaces.
3. The token for this ServiceAccount will be stored as a GitHub Secret.

## 3. Rollout & Rollback Strategy

* **Application Workloads (API, Dashboard)**: Kubernetes `Deployment` resources natively handle Rolling Updates. When a new image is deployed, new pods are spun up and traffic is shifted only after readiness probes pass, ensuring zero-downtime deployments.
* **Worker**: Must use a `Recreate` strategy (scale to 0, then scale to 1) to prevent two versions of the worker from conflicting on queue items during a rollout.
* **Rollback**: If a deployment fails health checks, Kubernetes halts the rollout. We can instantly rollback by reverting the image tag in the deployment manifest.

## 4. Database Migration Strategy

Database schema changes represent the highest risk during deployments. `nst-events` strictly follows the **Expand and Contract** pattern for database migrations.

### The Expand and Contract Rule
1. **Expand**: Add new columns/tables (nullable) in Migration A. Deploy the code that writes to both old and new structures.
2. **Migrate**: Run background scripts to backfill data into the new structure.
3. **Contract**: Deploy code that reads/writes only from the new structure. In Migration B, drop the old columns/tables.

**Never perform destructive schema changes (renaming a column, dropping a table) in the same deployment step as the code that relies on it.** During a rolling deployment, both old and new pods will be running simultaneously. Destructive schema changes will cause the old pods to crash immediately.
