# 04 — PostgreSQL, PostGIS & Longhorn 3-Way Replicated Storage

This document details the persistence layer of **NST Events**. It explains how the PostgreSQL database is configured, how PostGIS geospatial extensions enable campus geofencing, how Longhorn provides distributed 3-way block replication, and how network policies isolate data from unauthorized access.

---

## 1. Persistence Layer Architecture

The database infrastructure combines a containerized PostgreSQL instance with enterprise-grade, distributed block storage:

```text
 ┌──────────────────────────────────────────────────────────────────────────────────────────┐
 │                                   KUBERNETES COMPUTE LAYER                               │
 │                                                                                          │
 │  ┌───────────────────────────┐                     ┌──────────────────────────────────┐  │
 │  │ nst-api Pods              │                     │ nst-worker Pod                   │  │
 │  │ (app: nst-api)            │                     │ (app: nst-worker)                │  │
 │  └─────────────┬─────────────┘                     └─────────────────┬────────────────┘  │
 │                │                                                     │                   │
 │                │ TCP Port 5432 (Authorized by NetworkPolicy)         │                   │
 │                └──────────────────────────┬──────────────────────────┘                   │
 │                                           │                                              │
 │                                           ▼                                              │
 │                ┌───────────────────────────────────────────────────────┐                 │
 │                │ Service: nst-postgres-service (Port 5432)             │                 │
 │                └──────────────────────────┬────────────────────────────┘                 │
 │                                           │                                              │
 │                                           ▼                                              │
 │  ┌────────────────────────────────────────────────────────────────────────────────────┐  │
 │  │ StatefulSet Pod: nst-postgres-0                                                    │  │
 │  │ • Image: postgis/postgis:16-3.4 (PostgreSQL 16 + PostGIS)                          │  │
 │  │ • Mount Path: /var/lib/postgresql/data                                             │  │
 │  │ • PGDATA: /var/lib/postgresql/data/pgdata                                          │  │
 │  │ • Probes: pg_isready -U postgres -d nst_events                                     │  │
 │  │ • Resources: 100m–500m CPU / 256Mi–1Gi RAM                                        │  │
 │  └────────────────────────────────────────┬───────────────────────────────────────────┘  │
 └───────────────────────────────────────────┼──────────────────────────────────────────────┘
                                             │ iSCSI Block Storage Layer
                                             ▼
 ┌──────────────────────────────────────────────────────────────────────────────────────────┐
 │                        LONGHORN DISTRIBUTED STORAGE LAYER                                │
 │                         (StorageClass: longhorn, 10Gi PVC)                               │
 │                                                                                          │
 │              Synchronous 3-Way Block Mirroring Across Bare-Metal Nodes                   │
 │                                                                                          │
 │   ┌──────────────────────┐    ┌──────────────────────┐    ┌──────────────────────┐       │
 │   │  Physical Node nst-n2│    │  Physical Node nst-n4│    │  Physical Node nst-n6│       │
 │   │  ┌────────────────┐  │    │  ┌────────────────┐  │    │  ┌────────────────┐  │       │
 │   │  │ Longhorn Vol   │  │    │  │ Longhorn Vol   │  │    │  │ Longhorn Vol   │  │       │
 │   │  │ Replica 1 (NVMe)  │  │    │  │ Replica 2 (NVMe)  │  │    │  │ Replica 3 (NVMe)  │  │       │
 │   │  └────────────────┘  │    │  └────────────────┘  │    │  └────────────────┘  │       │
 │   └──────────────────────┘    └──────────────────────┘    └──────────────────────┘       │
 └──────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. PostgreSQL StatefulSet (`nst-postgres`)

The database runs as a single-replica Kubernetes `StatefulSet` (`infrastructure/kubernetes/postgres-statefulset.yaml`).

### Configuration Highlights
- **Base Image**: `postgis/postgis:16-3.4`
  - Runs PostgreSQL 16 on Debian.
  - Bundles PostGIS spatial extensions, GEOS, and Proj libraries.
- **Resource Allocations**:
  - CPU Request: `100m` | CPU Limit: `500m`
  - RAM Request: `256Mi` | RAM Limit: `1Gi`
- **Volume Mounts**:
  - Bound to PVC `postgres-data` at `/var/lib/postgresql/data`.
  - `PGDATA` subpath: `/var/lib/postgresql/data/pgdata` (prevents data collision with `lost+found` filesystem roots).
- **Probes**:
  - Readiness & Liveness both execute:
    ```bash
    pg_isready -U postgres -d nst_events
    ```
    This validates that the PostgreSQL server process is accepting TCP connections and ready to execute SQL transactions.

---

## 3. PostGIS Geospatial Capabilities in NST Events

NST Events uses **PostGIS** for physical campus attendance verification and anti-spoofing:

### 1. Campus Venue Geofencing
Every campus venue (e.g., *Auditorium 1*, *Innovation Lab*, *Main Football Ground*) is stored with geographic coordinates (latitude, longitude) and a permitted check-in radius (e.g., 50 meters).

### 2. Tamper-Proof Attendance Check-In
When a student scans a rotating event QR code with their mobile device:
1. The mobile app captures the student's GPS device coordinates.
2. The API executes a PostGIS spatial query:
   ```sql
   SELECT ST_DWithin(
     v.coordinates::geography,
     ST_SetSRID(ST_MakePoint(:userLongitude, :userLatitude), 4326)::geography,
     v.checkin_radius_meters
   ) AS is_within_venue
   FROM campus_venues v
   WHERE v.id = :venueId;
   ```
3. If `is_within_venue` returns `false`, the attendance record is rejected with `OUT_OF_BOUNDS`, preventing students from sharing QR code screenshots with friends outside the venue.

---

## 4. Longhorn Distributed Block Storage

A traditional Kubernetes volume is tied to a single physical disk. If that node dies, the database goes down permanently. To prevent this, the cluster uses **Longhorn** (`driver.longhorn.io`).

### Verified Cluster Storage Pool
A live audit of the Longhorn storage engine (`nodes.longhorn.io`) confirms:
- **Storage Nodes**: 6 active physical nodes (`nst-n2` through `nst-n7`).
- **Total Free Storage Pool**: **~282.8 GB** across NVMe drives.
- **Over-Provisioning Setting**: `100%` (`storage-over-provisioning-percentage: 100` — strict physical enforcement, no dangerous overcommit).
- **Minimal Available Threshold**: `10%`.

### Synchronous 3-Way Replication Mechanics
When PostgreSQL writes a block to `/var/lib/postgresql/data`:
1. The local Longhorn CSI driver intercepts the POSIX write.
2. The write is synchronously mirrored over the campus 1Gbps/10Gbps LAN to **three separate physical nodes** simultaneously via iSCSI.
3. The write operation returns success to PostgreSQL only after all three replicas acknowledge the write.

### What Happens During Hardware Crashes?
- **Scenario A: PostgreSQL Pod Crash**:
  K3s restarts the container on the same machine. It re-mounts the volume in < 5 seconds.
- **Scenario B: Physical Node Power Loss**:
  If the physical machine hosting the database loses power:
  1. K3s detects the node is `NotReady` after 40 seconds.
  2. The K3s controller reschedules `nst-postgres-0` to a healthy physical node (e.g., `nst-n6`).
  3. The Longhorn CSI driver attaches the existing volume to the new node.
  4. The database boots up with **zero data loss** because two other replicas were continuously updated.
- **Scenario C: Single Disk Hardware Failure**:
  If an NVMe drive physically fails:
  1. The volume continues operating smoothly in `Degraded` status on the 2 healthy replicas (zero downtime).
  2. Longhorn waits 600 seconds (`replica-replenishment-wait-interval`).
  3. Longhorn automatically provisions a new 3rd replica on another available physical node and synchronizes data in the background.

### Dynamic Online Volume Expansion
Because the `longhorn` storage class specifies `allowVolumeExpansion: true`:
- The volume can be expanded from `10Gi` to `20Gi` or `50Gi` **without taking the database offline**.
- The operator updates `resources.requests.storage` on the PVC. Longhorn dynamically expands the block device and resizes the ext4 filesystem live while PostgreSQL continues serving user traffic.

---

## 5. NetworkPolicy Database Isolation

To adhere to the principle of least privilege, the database is surrounded by a strict Kubernetes `NetworkPolicy` (`infrastructure/kubernetes/network-policies.yaml`):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-network-policy
  namespace: nst-events
spec:
  podSelector:
    matchLabels:
      app: nst-postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: nst-api
        - podSelector:
            matchLabels:
              app: nst-worker
        - podSelector:
            matchLabels:
              app: nst-migration
      ports:
        - protocol: TCP
          port: 5432
```

### Security Guarantees:
1. **Default-Deny Ingress**: All inbound connections to the database pod are dropped by default.
2. **Strict Whitelist**: Only pods explicitly labeled `app: nst-api`, `app: nst-worker`, or `app: nst-migration` within the `nst-events` namespace can establish a TCP handshake on port 5432.
3. **No Direct Frontend Access**: The dashboard (`app: nst-dashboard`) cannot connect to PostgreSQL.
4. **No Cross-Namespace Access**: Pods in other cluster namespaces (or student sandbox environments) cannot reach the database.
5. **No Public Routing**: The database has no Kubernetes `Ingress` rule and no public IP exposure.
