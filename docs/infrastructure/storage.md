# Storage Architecture & Persistence

This document defines the persistent storage architecture for the `nst-events` platform, based on verified K3s cluster capabilities and repository requirements.

---

## 1. Storage Status & Overview

| Category | Definition |
| :--- | :--- |
| **CURRENT STATE (REPO)** | `postgres-statefulset.yaml` requests a 10Gi volume named `postgres-data` without specifying `storageClassName`. `docker-compose.yml` mounts a local Docker named volume `pgdata`. |
| **VERIFIED CLUSTER CAPABILITY** | K3s cluster has Longhorn installed with 6 dedicated storage nodes (`nst-n2` through `nst-n7`), 3-way synchronous block replication, dynamic volume expansion enabled (`allowVolumeExpansion: true`), and ~282.8 GB total free storage. |
| **INTENDED ARCHITECTURE** | Dedicated Longhorn PVC for PostgreSQL per environment (`nst-events-prod` 10Gi, `nst-events-staging` 5Gi). Explicit `storageClassName: longhorn`, `ReadWriteOnce`. |
| **MISSING CONFIGURATION** | Explicit `storageClassName` in StatefulSet manifests; automated volume capacity alerting. |
| **REQUIRED ACTION** | Add `storageClassName: longhorn` to `postgres-statefulset.yaml`; define disk pressure alerting rules. |

---

## 2. Verified Cluster Storage Infrastructure

### Storage Classes
The cluster provides four storage classes (verified via `k3s kubectl get sc`):

1. **`longhorn` (Default)**:
   - **Provisioner**: `driver.longhorn.io`
   - **Reclaim Policy**: `Delete`
   - **Volume Binding Mode**: `Immediate`
   - **Allow Volume Expansion**: `true`
   - **Replica Count**: `3` (synchronously replicated across 3 independent physical nodes)
   - **Primary Use**: **All stateful production and staging databases for `nst-events`**.

2. **`local-path`**:
   - **Provisioner**: `rancher.io/local-path`
   - **Reclaim Policy**: `Delete`
   - **Binding Mode**: `WaitForFirstConsumer`
   - **Limitation**: Bound to a single host node's local disk without replication or failover capability. **Unsuitable for production PostgreSQL**.

3. **`longhorn-r1-local`**:
   - **Provisioner**: `driver.longhorn.io`
   - **Replica Count**: `1` (single replica, `Retain` policy). Used only for non-critical caches or disposable dev environments.

4. **`longhorn-static`**:
   - For statically provisioned pre-existing Longhorn volumes.

### Storage Node Topography & Capacity
A live audit of the Longhorn storage engine (`nodes.longhorn.io`) confirmed:

* **Control Plane Node (`nst-n1`)**: Contains no Longhorn storage disks (`allowScheduling: false`).
* **Storage Nodes (`nst-n2` through `nst-n7`)**: 6 active storage nodes, all `Ready` and `Schedulable`:
  - `nst-n2`: 24.4 GB available / 97.9 GB max disk
  - `nst-n3`: 29.1 GB available / 97.9 GB max disk
  - `nst-n4`: 33.9 GB available / 97.9 GB max disk
  - `nst-n5`: 30.6 GB available / 97.9 GB max disk
  - `nst-n6`: 108.1 GB available / 913.3 GB max disk
  - `nst-n7`: 56.7 GB available / 97.9 GB max disk
* **Total Cluster Free Storage**: ~282.8 GB.
* **Over-Provisioning Setting**: `100%` (`storage-over-provisioning-percentage: 100` — strict physical enforcement, no overcommit).
* **Minimal Available Threshold**: `10%` (`storage-minimal-available-percentage: 10`).

---

## 3. Longhorn Volume Behavior & Appropriateness for PostgreSQL

### Synchronous 3-Way Replication
When PostgreSQL writes to `/var/lib/postgresql/data`, the Longhorn CSI engine synchronously mirrors every block to 3 distinct storage nodes via iSCSI over the internal network. 

* **High Availability (HA)**: If the physical node hosting the PostgreSQL pod fails, Kubernetes reschedules the pod to another node. The newly scheduled pod re-attaches to the Longhorn volume via CSI immediately. Data is intact because 2 other replicas were continuously updated.
* **Replica Rebuild**: If a single node disk fails permanently, Longhorn waits `600s` (`replica-replenishment-wait-interval`) before automatically rebuilding the missing 3rd replica on another healthy node.

### Appropriateness Assessment
* **Workload Fit**: For the college-scale event platform (~1,000–10,000 students, bursty event check-in traffic), the network latency overhead of 3-way synchronous replication is negligible compared to the operational safety of automatic volume failover.
* **Bottlenecks to Watch**: Disks on nodes `n2`–`n5` are ~100GB NVMe drives with ~24–34GB available. Because a 10Gi volume with 3 replicas reserves **30Gi raw space** across 3 nodes, storage must be actively monitored before expanding volumes.

---

## 4. PostgreSQL Storage Allocations & Growth Management

### Volume Allocations
* **Production Database (`nst-events-prod`)**: Initial **10Gi** PVC.
  - Accommodates base PostgreSQL 16 system files, PostGIS spatial indices, table data for users, events, registrations, attendance records, audit logs, and push tokens.
  - Projected 1-year growth with 50,000 attendance records and audit logs: < 3Gi raw DB data. 10Gi provides ample headroom for WAL files and index bloat.
* **Staging Database (`nst-events-staging`)**: Initial **5Gi** PVC.
  - Fully isolated PVC, separate volume, separate namespace.

### WAL & Temporary Files
* **WAL (Write-Ahead Logging)**: Kept inside `/var/lib/postgresql/data/pg_wal`. Standard PostgreSQL defaults maintain 1GB to 2GB of WAL files.
* **Temporary Files**: Large analytical queries or sorts spill to `pgsql_tmp`. Queries must be tuned so temporary sort files do not exhaust disk capacity.
* **No Backups on Data Volume**: Backups (`pg_dump`) must **NEVER** be saved to `/var/lib/postgresql/data`. Writing dumps to the data volume risks cascading disk-full failures that crash the database.

### Dynamic Volume Expansion
Because the `longhorn` storage class has `allowVolumeExpansion: true`, expanding database storage does not require recreating the PVC or StatefulSet:
1. Update `spec.resources.requests.storage` in the PVC (e.g., from `10Gi` to `20Gi`).
2. Longhorn dynamically expands the underlying volume and resizes the ext4 filesystem online while the PostgreSQL container remains running.

---

## 5. Repository Storage Discrepancies & Required Fixes

1. **Missing StorageClass**:
   - *Current Code* (`postgres-statefulset.yaml:35`):
     ```yaml
     volumeClaimTemplates:
       - metadata:
           name: postgres-data
         spec:
           accessModes: ['ReadWriteOnce']
           resources:
             requests:
               storage: 10Gi
     ```
   - *Required Fix*: Explicitly set `storageClassName: longhorn`.
2. **Secret Reference Name Mismatch**:
   - *Current Code* (`postgres-statefulset.yaml:24`): `secretRef: name: nst-secrets`
   - *Repository Reality* (`secrets.yaml:26`): Secret is named `nst-db-admin-secrets`.
   - *Required Fix*: Align the secretRef name to `nst-db-admin-secrets`.
3. **Missing Health Probes**:
   - `postgres-statefulset.yaml` has no `livenessProbe` or `readinessProbe`.
   - *Required Fix*: Add `exec` probe running `pg_isready -U postgres -d nst_events`.
