# 05 — CI/CD Pipeline, Self-Hosted Runner & Rollout Gating

This document details the automated continuous integration and continuous deployment (CI/CD) pipeline for **NST Events**. It explains how code moving from git commits to live production containers is tested, built on a self-hosted runner, deployed to K3s, and automatically rolled back if health checks fail.

---

## 1. Automated Deployment Pipeline Overview

Every deployment to the live cluster follows a strictly gated, zero-downtime pipeline:

```text
  Developer Push to 'main'
            │
            ▼
┌───────────────────────────────────────┐
│ GitHub Actions CI (.github/workflows/ci.yml)
│  • Turbo typecheck & lint             │
│  • Prisma schema validation           │
│  • Unit & Integration Test Suites     │
└───────────────────┬───────────────────┘
                    │ Tests Pass (conclusion == 'success')
                    ▼
┌───────────────────────────────────────┐
│ Deploy Trigger (.github/workflows/deploy.yml)
│ Runs On: [self-hosted, linux, x64, n6]│
└───────────────────┬───────────────────┘
                    │
                    ▼
┌───────────────────────────────────────┐
│ Path Filtering & Change Detection     │
│  • Compares diff between last deploy  │
│    SHA and target commit SHA          │
│  • Rebuilds ONLY modified components  │
└───────────────────┬───────────────────┘
                    │
                    ▼
┌───────────────────────────────────────┐
│ Docker Build & Push to GHCR           │
│  • Image: ghcr.io/nst-sdc/nst-*:SHA   │
│  • Layer caching on local NVMe disk   │
└───────────────────┬───────────────────┘
                    │
                    ▼
┌───────────────────────────────────────┐
│ Cluster Rollout & Health Gating       │
│ (scripts/deploy-rollout.sh)           │
│  • kubectl set image                  │
│  • kubectl rollout status --timeout=300s
└───────────────────┬───────────────────┘
                    │
          ┌─────────┴─────────┐
          │ Health Verified?  │
          ▼                   ▼
      [ YES ]              [ NO ]
    Deployment          Automatic
     Succeeds           Rollback
  (Zero Downtime)  (kubectl rollout undo)
```

---

## 2. The Self-Hosted Cluster Runner (`nst-n6`)

Deployments do not execute on generic cloud-hosted GitHub Actions runners. Instead, they run on a dedicated **self-hosted runner installed directly on cluster node `nst-n6` (`192.168.136.150`)**:

### Why a Self-Hosted Runner?
1. **Direct Campus LAN Access**: The runner can connect directly to the private K3s API server (`https://192.168.136.145:6443`) across the campus network without exposing the API server to the public internet.
2. **Fast Local Docker Layer Caching**: Because `nst-n6` has a 913 GB NVMe drive, Docker layer caches remain warm between builds. Base images and `pnpm` virtual stores do not need to be re-downloaded over the campus internet uplink.
3. **Hardware Headroom**: With 64 GB of RAM, `nst-n6` effortlessly handles parallel container builds alongside live workloads.

### Scoped Deployer RBAC (`github-actions-deployer`)
The self-hosted runner does **not** operate with `cluster-admin` privileges. It authenticates using a dedicated Kubernetes `ServiceAccount`:

- **Kubeconfig Location on `n6`**: `/home/github-runner/.kube/config` (permission `0600`, owned by system user `github-runner`).
- **Cluster Token**: Injected via secret `github-actions-deployer-token` in namespace `nst-events`.
- **RBAC Boundary**: Bound strictly to `Role/github-actions-deployer-role`:
  - **Allowed Resources**: `deployments`, `deployments/rollback`, `deployments/scale`, `replicasets`, `pods`, `pods/log`, `services`, `events`, `configmaps`, `ingresses`, `poddisruptionbudgets`.
  - **Allowed Verbs**: `get`, `list`, `watch`, `create`, `update`, `patch`.
  - **Prohibitions**:
    - The runner **cannot** access other namespaces (`kube-system`, `default`, `longhorn-system`).
    - The runner **cannot** read sensitive application secrets (`nst-db-admin-secrets`, `nst-api-secrets`).

---

## 3. Intelligent Path Filtering & Change Detection

To prevent slow, monolithic deployments when only a single service has changed, `.github/workflows/deploy.yml` implements dynamic diff detection:

### 1. Discovering the Previous Successful Commit
The pipeline queries the GitHub Actions API for the last successful deploy run on `main`. If no run exists (or if deploying for the first time), it falls back to inspecting the live container image running in the cluster:
```bash
RUNNING_IMG=$(kubectl get deployment nst-api -n nst-events -o jsonpath='{.spec.template.spec.containers[0].image}')
```

### 2. Computing the Diff Matrix
The workflow compares the previous commit against the target commit:
```bash
DIFF=$(git diff --name-only "${PREV_DEPLOY_SHA}" "${TARGET_SHA}")
```
- **Shared packages or root configs changed** (`packages/`, `pnpm-lock.yaml`, `package.json`):
  Sets `BUILD_API=true`, `BUILD_WORKER=true`, `BUILD_DASHBOARD=true`.
- **Only API changed** (`apps/api/`, `docker/Dockerfile.api`):
  Rebuilds and redeploys **only** `nst-api`.
- **Only Worker changed** (`apps/worker/`, `docker/Dockerfile.worker`):
  Rebuilds and redeploys **only** `nst-worker`.
- **Only Dashboard changed** (`apps/dashboard/`, `docker/Dockerfile.dashboard`):
  Rebuilds and redeploys **only** `nst-dashboard`.

---

## 4. Rollout Gating & Automatic Rollback Engine

Deployments are executed through a hardened shell wrapper: `scripts/deploy-rollout.sh`.

### Invocation Pattern
```bash
./scripts/deploy-rollout.sh <deployment-name> <namespace> <timeout>
# Example:
./scripts/deploy-rollout.sh nst-api nst-events 300s
```

### The Step-by-Step Rollout Lifecycle

#### 1. Snapshot Revisions Before Deploy
The script checks existing deployment history:
```bash
REVISIONS_BEFORE=$(kubectl rollout history deployment/"${DEPLOYMENT}" -n "${NAMESPACE}")
```

#### 2. Apply New Container Image
The deployment manifest is patched with the new immutable commit tag:
```bash
kubectl set image deployment/nst-api api=ghcr.io/nst-sdc/nst-api:${TARGET_SHA} -n nst-events
```

#### 3. Monitor Health Status (300-Second Gate)
The script blocks and watches the rollout status:
```bash
kubectl rollout status deployment/"${DEPLOYMENT}" -n "${NAMESPACE}" --timeout=300s
```
- For `nst-api` and `nst-dashboard`, Kubernetes performs a **RollingUpdate**:
  - New pods are spawned.
  - Traffic is only directed to new pods once `/ready` and `/login` HTTP probes return 200 OK.
  - Old pods are drained and terminated only after new pods are fully healthy.
- If all replicas pass probes, the script exits with code `0`, and the deployment succeeds.

#### 4. Automatic Rollback on Failure
If new pods crash (e.g., runtime exception, missing environment variable, failed DB migration) or fail health checks within 300 seconds:
1. **First-Deploy Check**:
   If the cluster has only 1 revision (first-ever deployment), there is no previous version to roll back to. The script halts, leaves the failing pods intact for inspection, and alerts the operator.
2. **Automatic `kubectl rollout undo`**:
   If previous healthy revisions exist, the script immediately dispatches:
   ```bash
   kubectl rollout undo deployment/"${DEPLOYMENT}" -n "${NAMESPACE}"
   ```
   It then blocks until the rollback completes and the previous known-good revision is restored to 100% healthy traffic serving.
3. **Step Summary Annotation**:
   The script writes a formatted markdown alert to `$GITHUB_STEP_SUMMARY` detailing the failure and confirming that previous traffic was safely preserved.

---

## 5. Summary of CI/CD Deployment Commands

Operators can also manually trigger rollouts or inspect deployment status directly from `nst-n6` or via `kubectl`:

```bash
# Check current rollout status
kubectl rollout status deployment/nst-api -n nst-events

# View rollout revision history
kubectl rollout history deployment/nst-api -n nst-events

# Manually trigger an immediate rollback to the previous revision
kubectl rollout undo deployment/nst-api -n nst-events

# Restart pods with zero downtime
kubectl rollout restart deployment/nst-api -n nst-events
```
