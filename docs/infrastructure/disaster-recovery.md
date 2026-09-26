# Disaster Recovery Matrix & Operational Runbook

This document defines the disaster recovery matrix, expected recovery procedures, and RPO/RTO targets for all failure modes of the `nst-events` platform.

---

## 1. Disaster Recovery Status & Overview

| Category | Definition |
| :--- | :--- |
| **CURRENT STATE (REPO)** | No disaster recovery runbook, automated failover manifests, or off-cluster replication scripts exist. |
| **VERIFIED CLUSTER CAPABILITY** | High availability at the pod and node level via K3s controller and Longhorn 3-way storage replication. Zero disaster recovery capability if the physical cluster or college datacenter is compromised. |
| **INTENDED ARCHITECTURE** | Multi-tier recovery plan leveraging Longhorn for local hardware failover (< 5 min RTO) and off-cluster S3 backups for catastrophic disaster recovery (< 2 hour RTO). |
| **MISSING CONFIGURATION** | Off-cluster backup destination; automated disaster recovery restore runbooks; regular disaster simulation tests. |
| **REQUIRED ACTION** | Establish off-cluster storage bucket; validate disaster recovery restore procedure in staging. |

---

## 2. Disaster Recovery Matrix

The following matrix documents the survival, recovery method, automation level, and target RPO/RTO for each critical failure mode.

> [!NOTE]
> **Target Values**
> All RPO (Recovery Point Objective) and RTO (Recovery Time Objective) values below are **recommended targets** and remain targets until formally validated through staging simulation drills.

| Failure Mode | What Survives | What Does NOT Survive | Recovery Method | Target RPO | Target RTO | Recovery Type |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **1. PostgreSQL Pod Failure** | Persistent volume, all data, all 3 Longhorn replicas. | Ephemeral container memory, in-flight uncommitted transactions. | K3s automatically restarts the pod on the same node; mounts the existing PVC. | **0 seconds** | **< 30 seconds** | **Automated** |
| **2. Kubernetes Node Failure** | Longhorn replicas on the other 5 storage nodes. | The crashed physical node and the pod running on it. | K3s reschedules the StatefulSet pod to a healthy node; Longhorn CSI re-attaches the volume. | **0 seconds** | **< 3 minutes** | **Automated** |
| **3. Longhorn Replica Failure** | 2 remaining healthy replicas; active PostgreSQL pod. | 1 replica on the degraded disk. | Volume remains ReadWrite on healthy replicas. After 600s wait, Longhorn auto-rebuilds the 3rd replica on another healthy node. | **0 seconds** | **0 downtime** (rebuild takes ~10m background) | **Automated** |
| **4. PostgreSQL Data Corruption** | Raw storage volume, unaffected tables/files. | Corrupted tables, invalid indexes, or broken WAL stream. | Stop PostgreSQL pod. Spin up a new instance, restore the latest uncorrupted logical `pg_dump` from backup storage. | **Target: < 24 hours** (last daily dump) | **Target: < 45 minutes** | **Manual** |
| **5. Accidental Deletion (`DROP TABLE` / bad migration)** | Cluster infrastructure, storage volumes (holding the dropped state). | Dropped data, dropped tables. | **Longhorn replication CANNOT save this.** Spin up a temporary database, restore yesterday's `pg_dump`, extract the lost table, and re-import. | **Target: < 24 hours** | **Target: < 1 hour** | **Manual** |
| **6. Namespace Deletion (`kubectl delete ns nst-events-prod`)** | Off-cluster backups, other cluster namespaces. | Deployments, Services, Secrets, Ingress, and PVCs (due to `Delete` reclaim policy). | Re-apply Kubernetes manifests via GitOps/CI-CD; restore data from latest off-cluster `pg_dump`. | **Target: < 24 hours** | **Target: < 1.5 hours** | **Manual** |
| **7. Complete K3s Cluster Failure** | Off-cluster S3 backups, container images in registry. | All nodes, cluster state, K3s etcd, local Longhorn volumes. | Re-provision K3s; re-install Longhorn and cert-manager; redeploy manifests; restore database from off-cluster S3 backup. | **Target: < 24 hours** | **Target: < 4 hours** | **Manual** |
| **8. Longhorn Storage Engine Failure** | Off-cluster S3 backups, application code. | All local volume replicas and Longhorn engine metadata. | Re-install Longhorn or switch temporarily to `local-path` storage; restore PostgreSQL from off-cluster S3 backup. | **Target: < 24 hours** | **Target: < 2 hours** | **Manual** |
| **9. Total College Infrastructure Outage (Datacenter Down)** | Off-cluster S3 backups, GitHub repository, Cloudflare DNS. | All on-premise servers, power, network connectivity. | Redirect Cloudflare DNS to temporary cloud instance (e.g., AWS/DigitalOcean); deploy container images; restore S3 backup. | **Target: < 24 hours** | **Target: < 4 hours** | **Manual** |

---

## 3. Environment Isolation (Staging vs. Production)

To prevent operational accidents in staging from ever impacting production data:

### Namespace & Volume Boundary
* **Production**: Namespace `nst-events-prod`. Volume `postgres-data-prod` (10Gi).
* **Staging**: Namespace `nst-events-staging`. Volume `postgres-data-staging` (5Gi).
* **No Cross-Mounting**: Kubernetes PVCs are strictly namespace-scoped. A pod in `nst-events-staging` cannot bind or mount a volume residing in `nst-events-prod`.

### Network Isolation
* `NetworkPolicies` in `nst-events-staging` and `nst-events-prod` reject all cross-namespace traffic.
* The staging API cannot connect to the production PostgreSQL instance under any circumstance.

### Backup Storage Isolation
* **IAM / Secret Boundaries**:
  - Production backup secret has Read/Write permissions to `s3://nst-backups/production/`.
  - Staging backup secret has Read/Write permissions strictly to `s3://nst-backups/staging/`.
  - Staging has **Read-Only** access to production backups, restricted solely to the sanitization restoration pipeline.

---

## 4. Catastrophic Disaster Recovery Runbook

In the event of total cluster loss, follow this step-by-step restoration procedure:

### Phase 1: Infrastructure Initialization
1. Verify network connectivity and DNS propagation for the new node(s).
2. Install K3s:
   ```bash
   curl -sfL https://get.k3s.io | sh -
   ```
3. Verify Traefik, cert-manager, and Longhorn are healthy:
   ```bash
   kubectl get pods -A
   ```

### Phase 2: Workload Deployment
1. Create namespaces:
   ```bash
   kubectl create namespace nst-events-prod
   ```
2. Apply Secrets, NetworkPolicies, and Storage configurations:
   ```bash
   kubectl apply -f infrastructure/kubernetes/secrets.yaml -n nst-events-prod
   kubectl apply -f infrastructure/kubernetes/network-policies.yaml -n nst-events-prod
   kubectl apply -f infrastructure/kubernetes/postgres-statefulset.yaml -n nst-events-prod
   ```

### Phase 3: Database Data Restoration
1. Wait for the PostgreSQL pod to reach `Running` status:
   ```bash
   kubectl wait --for=condition=ready pod/nst-postgres-0 -n nst-events-prod --timeout=120s
   ```
2. Download the latest verified backup from off-cluster S3:
   ```bash
   aws s3 cp s3://nst-backups/production/latest.dump.enc /tmp/backup.dump.enc
   # Decrypt
   gpg --decrypt /tmp/backup.dump.enc > /tmp/backup.dump
   ```
3. Restore using `pg_restore`:
   ```bash
   kubectl cp /tmp/backup.dump nst-events-prod/nst-postgres-0:/tmp/backup.dump
   kubectl exec -it nst-postgres-0 -n nst-events-prod -- pg_restore -U postgres -d nst_events --clean --if-exists /tmp/backup.dump
   ```

### Phase 4: Application Deployment & Traffic Routing
1. Apply API and Dashboard deployments:
   ```bash
   kubectl apply -f infrastructure/kubernetes/api-deployment.yaml -n nst-events-prod
   kubectl apply -f infrastructure/kubernetes/ingress.yaml -n nst-events-prod
   ```
2. Verify API readiness probes pass:
   ```bash
   kubectl get pods -n nst-events-prod
   ```
3. Update or verify Cloudflare DNS points to the new ingress IPs.
